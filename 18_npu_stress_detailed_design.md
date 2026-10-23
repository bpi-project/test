# NPU 与多摄像头稳定性测试详细设计

修订日期：2026-10-23。以 [需求](17_npu_stress_requirements.md) 为行为约束，已合入
[可行性评审](19_npu_camera_feasibility_review.md) 与 [源码/模型核查](20_npu_platform_compatibility.md)。
当前目录为设计文档，以下接口及配置是实现约定，不代表已有程序或真机验证结果。

## 1. 设计原则

`npu_inference` 采用稳定性框架统一的 Service 插件模型。Camera 采集和 vendor NPU runtime 放在
Service 管理的独立 `stability-test-npu-worker` 中，GUI 只读取受控预览缓冲和检测 metadata，
不直接打开 Camera 或 NPU 设备。测试状态和最终判定仍由 Service 掌握。实现遵循：

1. 上层流程统一，平台 runtime 隔离；
2. Camera 采集、推理和 GUI 展示互不阻塞；
3. 双摄同时采集，但第一版使用单个模型 context 串行逐路推理；
4. 使用最新帧而不是无界帧队列；
5. 模型契约在开始运行前校验，错误不能延迟到长稳测试中才暴露；
6. cycle 是统计与判定窗口，不代表每轮重载模型或重启 Camera；
7. 平台错误保留原始信息，同时映射为跨平台统一错误码；
8. 同一产品版本、公共 worker 和控制协议覆盖五个平台，硬件/库/模型差异由 YAML 选择；
9. 优先完成程序逻辑，Qt 卡片通过统一 ResultSink/预览接口后续接入；
10. 一个 context 所属线程负责 backend 完整生命周期，进程外监督负责无法返回的调用。

业务层不包含按 SoC 分叉的调度、后处理或判定。共同使用 GStreamer 的板卡共用采集实现；
不同 pipeline 本身不要求增加 Camera backend。BSP 需要不同构建产物时，仍从同一源码和产品版本发布。

## 2. 组件划分

```text
stability-test-service
  ├─ NpuInferencePlugin
  │   ├─ NpuRunController
  │   ├─ CycleEvaluator
  │   └─ NpuReportAdapter
  ├─ NpuWorkerSupervisor
  └─ IPC
          │
          ▼
stability-test-npu-worker
  ├─ CameraCaptureWorker(front/rear)
  │   └─ ICameraSource / GstCameraSource（板级 pipeline）
  ├─ LatestFrameStore × N
  ├─ InferenceWorker（一个）
  ├─ PlatformBackend
  │   ├─ RenesasDrpAiBackend
  │   ├─ RockchipRknnBackend
  │   └─ AllwinnerVipLiteBackend
  ├─ SharedYoloV5PostProcessor
  ├─ ResultSink（有界检测结果/帧引用/指标）
  └─ PreviewPublisher 接口（Qt transport 后续实现）
          │
          ▼
stability-test-gui
  └─ read-only preview + detection overlay + NPU card
```

NPU worker 不创建窗口，通常不需要 `DISPLAY` 或 `WAYLAND_DISPLAY`。Service 通过受控 IPC 发送
不可变配置并接收 heartbeat、指标和结果，通过进程监督捕获崩溃与退出码。目标平台若要求特定
用户、设备组、环境变量或 runtime 初始化顺序，由板级 backend execution profile 指定，不把
整个 GUI 变成特权进程。Service 不信任 GUI 作为结果权威；GUI 退出时 worker 和后台 Run 继续，
GUI 重连后重新映射当前预览缓冲并恢复状态。

公共 worker 不强链接所有厂商库。Service 按 YAML 准备子进程环境，worker 只装载选定的 backend
模块与 runtime；某个平台未部署其他厂商 SDK 不影响启动。运行报告保存实际生效配置及装载结果。

## 3. 生命周期和状态机

### 3.1 Run 级资源

模型和 Camera 资源以 Run 为生命周期：

```text
NOT_STARTED
  → PROBING
  → LOADING_MODEL
  → OPENING_CAMERAS
  → OPTIONAL_SELF_TEST
  → RUNNING
  → STOPPING
  → COMPLETED / FAILED / CANCELLED / ABORTED
```

- `PROBING`：检查设备节点、runtime、backend 能力、模型和标签；
- `LOADING_MODEL`：创建一个 backend context 并校验输入输出契约；
- `OPENING_CAMERAS`：按配置启动所有 pipeline，等待每路首帧；
- `OPTIONAL_SELF_TEST`：使用标准图片执行一次确定性推理；
- `RUNNING`：Camera 持续采集，NPU 连续逐路推理；
- `STOPPING`：停止新任务并按受控顺序清理。

任一阶段失败均进入停止/清理分支；已完成步骤对应的资源才释放。Probe 可能调用厂商初始化 API，
因此 probe、load、run、close 均在同一推理线程执行。已判定失败的 Run 不能因清理成功改为成功。
COMPLETED 表示计划执行完成，最终 PASS/FAIL 由窗口和故障记录决定；用户提前停止且此前无失败
记为 CANCELLED。worker 异常退出由仍在运行的 Service 判为失败；Service/系统异常结束后恢复
旧 Run 时按通用流程记为 ABORTED。

正常 cycle 边界只读取单调累计计数的快照差值并生成 record，不清零共享计数、不关闭 Camera、
不卸载模型。启动及可选自检指标单列，正式窗口从进入 RUNNING 后开始。

### 3.2 统计窗口和工作量

建议 `schedule.cycle_mode: timed`。每个 cycle 是固定统计窗口：

```text
cycle start
  → snapshot monotonic cumulative counters
  → continuous capture/inference
  → evaluate thresholds at cycle end
  → write cycle record
  → next cycle
```

具体默认周期时长、timeout 和最低 FPS 需要各平台实测后确定，文档当前不设未经验证的硬门限。

`round` 是按配置访问各 Camera 一次；`cycle/window` 是框架统计窗口。首版要求 `interval_s: 0`，
窗口连续施压，不在窗口边界暂停。非零 interval 或其他尚未实现的调度模式在加载配置时拒绝。

计数口径：

| 名称 | 递增时机/含义 |
| --- | --- |
| `attempted_frames` | 取到新的帧并开始整链路尝试；前处理失败也计入最终失败数 |
| `submitted_inferences` | 进入 backend.run 的次数；是 API 调用数，不冒充硬件已执行次数 |
| `successful_inferences` | 本次 run、输出校验、后处理及输出释放均成功；空检测列表允许成功 |
| `failed_inferences` | 已开始的整链路尝试以错误/超时结束；同一 operation 只结算一次 |
| `cancelled_attempts` | 正常停止请求导致的中止；不含已发生的错误/超时，不混入成功率分母 |
| `in_flight` | 已开始但尚未结算的尝试；分别保留窗口开始/结束快照 |
| `preview_published_frames` | 已提供给预览消费者的帧，不能当作 GUI 实际显示帧数 |

成功率为 `success / (success + failure)`，分母为零时是无数据。完成/失败事件按单调结算时间归属
窗口，跨窗口 operation 不重复计算；每路有效 FPS 为窗口成功数除以该窗口实际活动时长。
每个完整窗口即使未设置最低次数/FPS，也要求每路成功数至少为 1。Camera 失流、进程故障等独立
判定，不因没有形成 inference attempt 而被排除。

提前结束窗口标记 partial，记录实际时长并保留已有失败；不据此满足计划完整时长。停止时冻结
活动窗口，之后的 in-flight 结算归 stop_drain 单列，不把清理耗时或迟到成功填进活动窗口。
只有明确由正常停止导致的取消才计 cancelled_attempts；未请求停止却收到 vendor cancel 仍按错误处理。

criteria 中的 `minimum_*` 按 `<` 触发不达标，`maximum_*` / `max_*` 为允许上限，按 `>` 触发；
配置值 0 表示不允许发生。接入主框架时如字段采用“触发次数”语义，需显式映射，不能直接沿用数值。
连续失败上限只控制可恢复错误的容忍/停止，context 无效、进程故障或设备不健康立即停止。
每路连续尝试失败数遇本路成功才清零，跨窗口保留；跳过无新帧不清零。连续失败窗口数遇成功窗口清零。
已判定失败的必测项锁定最终 FAIL，后续成功窗口不会清除。

## 4. 双摄数据流和调度

### 4.1 Camera worker

每路 Camera 有独立 capture worker 和固定容量缓冲：

```text
Capture frame
  → validate timestamp/size
  → increment captured_frames and update last_valid_arrival
  → copy into available pool slot, or count pool_exhaustion and drop
  → replace latest-frame slot
  → publish preview at capped rate
```

替换最新帧时统计 `overwritten_frames`。首版使用有限容量帧池和引用计数 FrameHandle，所有持有者
释放前不得覆盖该存储；仅有双槽加活动下标不足以保证慢读者安全。无空闲槽时丢弃新到帧并单独
计数，不等待 NPU/GUI。帧池上限至少覆盖 latest、in-flight 及允许的结果持帧，消费者有配额。
源帧到达和应用接纳分开记录，帧池耗尽不能误报成 Camera 没有出帧；整链路无进度仍独立判定。

Frame 至少包含 camera_id、应用 sequence、本地单调到帧时间、宽高、像素格式、各 plane offset/
字节 stride/有效长度和不可变存储引用；源 sequence、PTS、色彩矩阵/范围及错误标志能取得则保留，
不能取得则标记 unknown。PTS 的时钟域单列，不能直接与系统墙钟相减。

`GstCameraSource` 用指定 appsink 出口，检查协商 caps 与实际 buffer；处理 ERROR/EOS、首帧和失流。
pipeline/appsink 的队列及丢弃策略也必须有界并适配 BSP 的 GStreamer 版本。高频回调仅交付帧与
更新计数，不执行推理。第一版复制到公共帧池后及时释放源 sample，避免长期占用驱动采集 buffer。

### 4.2 串行逐路推理

第一版只有一个 `InferenceWorker` 和一个 `PlatformBackend` context：

```text
while running:
  for camera in configured_order:
    frame = camera.latestFrameAfter(lastConsumedSequence[camera])
    if frame unavailable:
      continue
    wait_for_global_rate_budget_or_stop()
    mark frame consumed; allocate operation_id; begin attempt
    preprocess(frame, model.input_binding)
    backend.run(input, output_lease, inference_deadline)
    validate_and_view_outputs(output_lease, model.manifest)
    shared_decode_and_nms()
    check output_lease.release() on owner thread
    finalize attempt; publish result via bounded ResultSink
```

伪代码省略错误分支；任一步失败都结束该 attempt、释放已安全取得的资源并按 context 状态决定
继续或停止。取帧后若等待限速导致帧过旧，刷新为该路更新帧或丢弃，不对过期帧形成成功工作量。
每路是否失流由独立监控判断，不依赖串行推理线程是否已轮到该 Camera。所有路均无新帧时等待
条件变量/有界事件，避免忙轮询。

双摄顺序由产品配置引用的 Camera 列表决定，不能依赖 `/dev/video*` 字典序。每个完整调度轮按顺序
覆盖所有启用 Camera；不能因 Front Camera 一直有新帧而饿死 Rear Camera。

`target_total_inference_fps` 是全部 Camera 合计的提交限速，正数限速，null 表示不主动限速；
不积累长时间等待后的突发额度，不因慢路跳过而饿死任一路。实际负载取决于采集、预处理和后处理，
不承诺 NPU 始终满载。两个线程同时调用同一 runtime 并不保证硬件
并行，反而可能在 vendor driver 内串行、争抢 context 或触发未定义行为，因此不作为跨平台默认。

### 4.3 后续并行模式

预留但第一版不实现：

```yaml
inference_strategy: parallel
contexts_per_camera: 1
```

只有 backend capability 明确声明 `multiple_contexts: true`、完成平台压力验证并定义内存预算后才
允许使用。不能在不支持的平台上静默退化，必须返回 `CAPABILITY_UNSUPPORTED`。

## 5. 平台后端接口

### 5.1 公共引擎接口

以下为接口形状；具体类型随主工程语言版本落地，不将示意声明当成已实现的头文件：

```cpp
class INpuBackend {
public:
    virtual ProbeResult probe(const BackendConfig &) = 0;
    virtual Result loadModel(const ModelArtifact &, ModelMetadata *) = 0;
    virtual Result validateContract(const ModelContract &) = 0;
    virtual Result run(const InputTensor &, OutputLease *, Deadline) = 0;
    virtual BackendMetrics metrics() const = 0;
    virtual Result close(Deadline) = 0;
    virtual ~INpuBackend() = default;
};
```

接口禁止使用 `void run()`。所有平台调用必须返回统一 `Result`，并携带：

- 统一错误码；
- 失败阶段；
- vendor 原始状态码及其是否可取得；
- 受限长度的错误文本；
- 是否可重试；
- context 是否仍可继续使用。

| 类型/操作 | 共同语义 |
| --- | --- |
| `ModelMetadata` | 逻辑模型契约、实际 InputBinding、各输出描述、元数据来源和适用 runtime/model 版本 |
| `InputTensor` | 按已协商 binding 准备的数据及长度/stride/所有权；至少保持到本次同步 run 完成 |
| `TensorDesc` | shape、字节 strides、dtype、有效/存储长度、量化类型与参数；数量相同不等于布局相同 |
| `OutputLease` | move-only 输出持有对象；视图在 release 前有效；显式 release 返回 Result，析构仅兜底 |
| `Deadline` | 单调时钟绝对截止时间；后端支持时换算 native timeout，不代表每个 vendor API 都可取消 |
| `run` 成功 | 输入已执行，输出已取回、完成必要 cache 同步且可读；不仅是提交成功 |
| `close` | 所属线程中按 vendor 依赖顺序清理，可处理部分初始化且重复调用安全；错误可报告 |

同一 context 不允许同时持有未释放输出并发起下一次 run，首版每次完整解码后释放 lease 再切换 Camera。
失败不得发布半成品 OutputLease，backend 负责回收安全可回收的部分资源；未知运行状态交由停止策略。
输出释放单独检查并受 deadline 监督；释放失败记 CLEANUP_FAILED 且不得复用相关 context，不靠
析构函数吞掉错误。无论正常结束还是解码失败，均走同一有界释放路径。
`metrics()` 也由所属线程采样后发布不可变快照，GUI/控制线程不直接调用 vendor 对象。

runtime 查询负责实际存储信息，manifest 负责颜色、激活、anchors、类别及参考向量。查询不到的
shape/layout 等可以使用经过完整模型 hash 校验的 manifest，并将来源标为 manifest；实际可查询的
数量、dtype、长度仍需交叉校验。禁止对未校验裸指针进行解码。

### 5.2 动态模块与 API 库

公共引擎内部使用 C++；跨 backend `.so` 的接口使用版本化窄 C ABI，含 abi_version、结构大小、
opaque handle、函数表及明确的分配/释放回调，不跨模块传 STL、C++ 异常或 vendor 对象。
必需入口涵盖 create/probe/load/describe/run/release_outputs/close/destroy；destroy 不替代可报告错误的 close。
模块 ABI 与厂商 SDK ABI 分别检查，不能把“能 dlopen”当成语义兼容。

RKNN/VIP 的函数表按目标 SDK 头文件定义，由 YAML 指定实际 API 库；Renesas wrapper 及 C++
依赖封在对应 SDK 构建的 bridge 内。Service 在启动 worker 前设置依赖环境；worker 按需装载并
检查必需符号，报告可取得的实际路径和版本。传递依赖仍需解析/校验，不能只验证顶层库路径。
运行中不混用不同 vendor/ABI 的 runtime，不通过自动尝试另一库或 CPU 回退掩盖兼容错误。

### 5.3 超时边界

厂商 timeout/cancel 能力由 backend 声明并在板端验证；Service 独立监督所有可能阻塞的阶段。
超时后不能从未知线程销毁仍在运行的 context/buffer。操作事件及进程终止规则见 §9、§11。

## 6. 各平台适配

### 6.1 Renesas DRP-AI/RUHMI

- RZ/V2H、RZ/V2N 使用对应平台编译模型；
- 公共 CPU 预处理先作为参考路径；PreRuntime/DMA 可按 profile 启用，须验证同等数值语义；
- 模型只加载一次；
- 双摄参考 `multi_camera_vehicle_detection`：多路同时采集，一个推理线程在 Camera 循环中依次
  调用 `runtime.Run()`；
- 不直接复制示例全局变量结构，应封装 Camera ID、buffer 和统计状态；
- 输出通过有所有权的视图进入共享 YOLOv5 后处理；
- `MeraDrpRuntimeWrapper` 是 C++ 接口，由本项目 bridge 封装；`Run` 为 void，捕获可取得的异常，
  并由进程退出、阶段 deadline、输出校验补充错误判定，不伪造 vendor 成功码；
- 区分 `mera.plan` 与旧 `deploy.json/.so/.params` 资源布局；按整包清单校验。

部分旧 wrapper 分支不支持完整 IO 描述，不能仅凭 GetOutput 的元素数声称 shape 已验证。
扩展图/plan 查询或使用固定 hash 的 manifest，并记录信息来源。官方 R01 用 YOLOv3，多摄参考用
YOLOX-l；本项目使用另行匹配的 YOLOv5s 模型/解码。V2H/V2N 的总线、频率、预留内存及门限按板配置。

### 6.2 Rockchip RKNN

- RK3576 使用经 RKNN Toolkit 转换的 `.rknn`；
- probe 需记录 RKNN runtime/driver 版本和目标 NPU core 能力；
- 量化输出必须使用 RKNN 提供的 scale、zero-point 和 tensor layout；
- 第一版一个 RKNN context；是否指定 NPU core mask 由板级 profile 定义，不写死在插件；
- 官方优化模型适合三头逻辑视图，但示例输出已含 sigmoid，按 manifest 避免二次激活；
- 示例用 UINT8/NHWC 提交 RGB，转换配置设置 mean=0/std=255；逻辑 NCHW 不要求应用先转成
  FP32/NCHW，input binding 必须明确 runtime 转换模式，避免重复归一化；
- 逐输出 query dtype/layout/量化参数，不仅按第一个输出推断全部输出；
- `rknn_outputs_get` 和必要同步属于 run 完成范围，使用后 `rknn_outputs_release`；
- 支持时设置 `rknn_run_extend.timeout_ms`，同时监督可能阻塞的取输出及其他阶段。

### 6.3 Allwinner VIPLite

- T527：NPU v2、VIPLite v1.13、v2 `.nb`；
- A733：NPU v3、VIPLite v2.0、v3 `.nb`；
- 所查 Model Zoo 为两者提供公共 VIP 头文件和封装，优先共用 `AllwinnerVipLiteBackend`；
- T527 API 库为 `libVIPlite.so`，依赖 `libVIPuser.so`；A733 为 `libNBGlinker.so`，依赖
  `libVIPhal.so`。通过 YAML 选择所需 `vip_*` API 库；不假定任意 SDK 的 ABI 或 `.nb` 通用；
- backend 必须检查 `vip_init`、create/prepare network、buffer、run 和 cache flush 的返回值；
- 旧 `awnn_run()` 返回 void；新 Model Zoo `network_run()` 虽返回 int，仍合并部分状态且 cache
  flush 失败仅打印。正式实现保留底层状态，cache 同步失败也映射为失败；
- 当前 SDK 示例把输入复制到 VIP buffer、输出转换成 float；第一版可沿用有界复制路径，后续再
  在验证 DMA-BUF API 和 cache 一致性后优化零拷贝；
- VIPLite 与 Unified driver/runtime 的装载策略由板级镜像明确指定，测试进程不能在运行中自动
  混用或切换；
- 所查 Model Zoo 脚本显式支持 target：T527 为 `VIP9000NANOSI_PLUS_PID0X10000016`，A733 为
  `VIP9000NANODI_PLUS_PID0X1000003B`；仍需锁定与实际 BSP 匹配的 compiler/runtime 组合；
- 使用 `VIP_NETWORK_PROP_SET_TIME_OUT` 时按 SDK 要求在 prepare 前设置并检查返回值；
- `vip_init/vip_destroy` 在同一线程调用；所查 T527/A733 路径均为无参数 `vip_init(void)`；
- 量化描述支持模型实际采用的 affine 或 dynamic fixed point；不统一假设只有 scale/zero-point。

旧 ai-sdk 示例是 UINT8 平面 CHW，新 Model Zoo 使用 UINT8 打包 RGB 与模型内预处理，输出布局也
不同；由模型 profile 描述这些差异。具体材料版本、归档 hash 及证据见文档 20，不能混用网页日志
和不同包的模型契约。

## 7. 共享预处理和后处理

### 7.1 预处理

共享预处理逻辑定义：

1. 校验输入帧格式和尺寸；
2. 转换为 RGB；
3. 按原始宽高比缩放；
4. 对剩余区域 letterbox；
5. 按 input_binding 执行由应用负责的归一化/量化和内存打包；其他步骤留给声明的模型/runtime；
6. 保存实际缩放后宽高、左右/上下 padding、比例、Camera ID、sequence 和单调到帧时间。

平台硬件预处理器可以替代 CPU 实现，但输出语义必须一致。报告应记录实际采用
`cpu|hardware` 预处理和耗时。
letterbox 的插值、填充值、尺寸取整及是否放大均来自 manifest。坐标反变换复用本帧实际变换参数，
不根据原尺寸重新计算另一套近似值。YUV 输入还需明确色彩矩阵/范围。

### 7.2 Canonical 输出

模型语义适配器在有效的 OutputLease 上建立统一逻辑视图：

```text
head stride=8  shape=3×80×80×85
head stride=16 shape=3×40×40×85
head stride=32 shape=3×20×20×85
```

canonicalization 必须处理 FP32、FP16、INT8/UINT8 以及平台 layout 差异。发生 NaN、Inf、大小
不足、重复 head 或无法匹配 stride 时返回 `INVALID_OUTPUT` 或 `MODEL_CONTRACT_MISMATCH`。
每个元素访问都受实际存储长度/stride 约束，使用可复用缓冲或 tensor view；不要求每帧分配整个
FP32 重排副本。未经实现与验证的 dtype/量化方式明确拒绝。模型语义映射不写进按 SoC 分支的调度代码。

### 7.3 后处理

共享后处理根据各字段的 logits/activated 声明按需执行 sigmoid，再完成 anchor decode、confidence
threshold、NMS 和坐标反变换。`raw3head` 表示尚未解码，不等于全部为 logits；不得通过值域猜测
激活边界或对量化概率做反 sigmoid。后处理参数来自
受 Schema 限制的配置，不从平台示例源码隐式继承。结果必须带原始 frame sequence，GUI 只把
结果叠加到同一路 Camera，并检查结果是否过期。

首版每个 anchor/cell 取最大类别分数，confidence=objectness×class_score，使用 class-aware NMS；
不实现 multi-label。阈值由产品配置给出，anchors/类别顺序由模型 manifest 给出。候选与最终输出
容量必须有界；按分数保留 top-K 并报告截断数，不能让坏输出造成无限分配或隐性无限耗时。
同一组固定图片和 tensor 参考向量分别验证预处理与解码，量化比较使用明确容差。

## 8. 配置设计

### 8.1 板级配置

板级配置描述硬件与平台差异。以下示例路径、hash、pipeline 和数值仅展示结构，不能直接部署；
正式 profile 必须填入实际 BSP 验证结果。`pipeline_profiles.capture` 是无窗口采集链路：

```yaml
hardware:
  cameras:
    devices:
      - id: front
        pipeline_profiles:
          capture:
            pipeline: >-
              <validated-front-pipeline-ending-in-appsink>
            sink_name: camera_sink
            expected_format: RGB
            max_buffers: 2
            overflow: drop_oldest
        frame_pool_capacity: 4
      - id: rear
        pipeline_profiles:
          capture:
            pipeline: >-
              <validated-rear-pipeline-ending-in-appsink>
            sink_name: camera_sink
            expected_format: RGB
            max_buffers: 2
            overflow: drop_oldest
        frame_pool_capacity: 4

  npu:
    devices:
      - id: default
        backend: allwinner_viplite
        backend_module: /opt/stability-test/backends/libst_npu_vip.so
        backend_abi: 1
        runtime_generation: v1.13
        runtime:
          api_library: /opt/vendor/t527/lib/libVIPlite.so
          dependency_dirs: [/opt/vendor/t527/lib]
          expected_version: "<validated-runtime-version>"
        device_nodes: [/dev/vipcore]
        model_profiles:
          - id: yolov5s_coco_640
            contract: yolov5s-coco-640-raw3head-v1
            model: /usr/share/stability-test/models/t527/yolov5s_rt_uint8_t527.nb
            labels: /usr/share/stability-test/models/coco80.txt
            sha256: "<packaged-model-sha256>"
            manifest: /usr/share/stability-test/models/t527/yolov5s_manifest.yaml
            manifest_sha256: "<packaged-manifest-sha256>"
```

实际 Camera pipeline、设备节点、runtime 世代和模型路径属于板级/BSP 差异，不能写死在插件。
pipeline 字符串只允许来自受信任板级配置，不允许普通产品配置或 GUI 输入任意 shell 命令。
`max_buffers/overflow` 由公共采集层映射成当前 GStreamer 版本支持的属性并核验，pipeline 内其他
queue 也需检查上限。原有 x11/wayland profile 可按主工程兼容规则保留，但核心检测不依赖显示初始化。

同一已验证 VIP ABI 下，A733 profile 选择其 `libNBGlinker.so`、包含 `libVIPhal.so` 的依赖目录和
A733 模型。Renesas bridge 的库集合/搜索环境、RKNN 的 `librknnrt.so` 也使用同一配置机制，
所需字段按 backend Schema 验证。core mask、频率索引、地址/内存及设备选项放在板级 execution
参数中，只接受所选 backend 支持的键；不支持的非空配置不得静默忽略。

Service 解析、校验并冻结本次配置再启动 worker；产品配置只能引用已存在的硬件与模型 profile。
库路径、模型语义不由 GUI 临时覆盖。实际运行信息与声明冲突时启动失败，报告保留声明值和实测值。

### 8.2 产品配置

```yaml
- id: npu.camera_detection
  plugin: npu_inference
  name: NPU Camera Detection
  enabled: true
  required: true
  device: npu.default
  schedule:
    cycle_mode: timed
    cycle_duration_s: 300
    cycle_timeout_s: 330
    interval_s: 0
    max_cycles: null
  config:
    model_profile: yolov5s_coco_640
    input_mode: camera
    inference_strategy: sequential_per_camera
    cameras: [front, rear]
    camera_pipeline_profile: capture
    target_total_inference_fps: 10
    per_inference_timeout_ms: 2000
    frame_timeout_ms: 3000
    max_frame_age_ms: 1000
    stage_timeouts_ms:
      probe: 10000
      load_model: 30000
      open_cameras: 10000
      self_test: 10000
      preprocess: 1000
      adapt_outputs: 1000
      postprocess: 1000
      release_outputs: 1000
      cleanup: 5000
    worker_heartbeat_timeout_ms: 3000
    progress_timeout_ms: 20000
    stop_grace_ms: 3000
    preview_max_fps: 30
    result_max_age_ms: 1000
    max_retained_frames_per_consumer: 1
    startup_self_test:
      enabled: false
      image: null
      expected_classes: []
      minimum_detections: 0
    postprocess:
      confidence_threshold: 0.4
      nms_threshold: 0.45
      nms_mode: class_aware
      multi_label: false
      max_candidates: 1000
      max_detections: 100
    detection_requirement:
      enabled: false
      required_classes: []
      minimum_detection_count_per_cycle: 0
  criteria:
    minimum_inferences_per_camera_per_cycle: null
    minimum_inference_fps_per_camera: null
    minimum_success_rate_percent: 100
    maximum_consecutive_inference_failures: 0
    maximum_frame_stalls_per_cycle: 0
    max_consecutive_failed_cycles: 0
```

示例中的时长、FPS、池/候选容量仅说明结构，不是 RZ/V2H、RZ/V2N、T527、A733、RK3576 的
最终默认值。正式 YAML 应由板端实测确定；每个 enabled 阶段都必须有有限 deadline。
`frame_timeout_ms` 同时规定单路启动后首帧和运行中最大无新帧间隔，open_cameras deadline
约束整体打开阶段；达到阶段 deadline 与失流 deadline 任一者均进入失败处理。

`progress_timeout_ms` 监控 RUNNING 时每路没有新完成工作的最长间隔，与心跳分别计时；校验时
需容纳配置限速、轮转及允许的单次处理时间。Self-test 使用其总 deadline，同时受内部推理等阶段
deadline 约束。清理阶段按 cleanup deadline 监督，stop_grace 是请求停止后的 in-flight 宽限期。

### 8.3 语义校验

配置加载时至少校验：

- `cameras` 非空、无重复且均存在于 board profile；
- 第一版 Camera 数量只能为 1 或 2；
- backend、runtime generation 与模型 profile 兼容；
- 模块 ABI、所选 API 库及必需依赖/符号匹配；实际路径和版本可用时交叉验证；
- 模型/标签/manifest 存在且与整包资源清单的 SHA-256 匹配；多文件模型使用目录文件清单校验；
- 逻辑模型 contract 是程序支持的版本；
- timeout 均为有限正数；限速为 null 或正数；成功率、阈值、窗口及帧池/候选容量范围有效；
- `startup_self_test.enabled=true` 时图片及其 hash、期望类别和最低数量有效；关闭时不校验空图片路径；
- `detection_requirement.enabled=true` 时类别和检测数量有效；
- 第一版只接受 `sequential_per_camera`、`interval_s: 0`、class-aware/single-label 后处理；
- input_binding 与所选 backend 可接受模式匹配；各输出有唯一 head 映射和完整激活/量化语义；
- 两个 minimum 字段可为 null，完整窗口每路至少一次成功的规则仍强制生效；
- 持帧消费者配额与帧池容量匹配；进度 deadline 不能小于配置必然产生的合法等待时间；
- 配置中保留的占位值不能进入正式运行；未知/未实现的键或选项报配置错误。

### 8.4 模型 manifest

manifest 与板级部署资源分开描述，也允许内联在 model profile。下面仅示意 RKNN 优化 YOLOv5 的
一种 binding，**不属于上面 T527 profile**；真实 output attr、源模型及转换参数仍需由所选产物核实：

```yaml
schema_version: 1
contract: yolov5s-coco-640-raw3head-v1
source:
  weights_sha256: "<source-weights-sha256>"
  onnx_sha256: "<source-onnx-sha256>"
  conversion_recipe: "<versioned-recipe-id>"
  compiler_version: "<validated-compiler-version>"
  target: rk3576
logical_input:
  shape: [1, 3, 640, 640]
  color: rgb
  layout: nchw
  value_range: [0, 1]
input_binding:
  dtype: uint8
  layout: nhwc
  color: rgb
  value_range: [0, 255]
  mode: runtime_conversion
  normalization:
    owner: model_runtime
    formula: "(x - mean) / std"
    mean: [0, 0, 0]
    std: [255, 255, 255]
geometry:
  resize: letterbox
  interpolation: bilinear
  fill_rgb: [114, 114, 114]
  dimension_rounding: floor
  padding: centered_remainder_right_bottom
  allow_upscale: true
outputs:
  - {index: 0, expected_shape: [1, 255, 80, 80], storage_layout: nchw, stride: 8}
  - {index: 1, expected_shape: [1, 255, 40, 40], storage_layout: nchw, stride: 16}
  - {index: 2, expected_shape: [1, 255, 20, 20], storage_layout: nchw, stride: 32}
output_storage:
  dtype_source: runtime_query
  quantization_source: runtime_query
  channel_packing: anchor_then_attribute
activation:
  xywh: sigmoid_applied
  objectness: sigmoid_applied
  classes: sigmoid_applied
decode:
  version: yolov5_anchor_v1
  classes: 80
  attributes_per_anchor: 85
  anchors:
    8: [[10, 13], [16, 30], [33, 23]]
    16: [[30, 61], [62, 45], [59, 119]]
    32: [[116, 90], [156, 198], [373, 326]]
reference_suite: "<versioned-inputs-results-and-tolerances>"
```

`activation` 每组只能取已实现的 `logits` 或 `sigmoid_applied`，前者由公共解码执行 sigmoid。
`expected_shape` 是由选定模型确认的实际张量形状，不是把不同平台强制改成同一存储布局；公共
视图再按 packing 映射为 anchor/y/x/attribute。SDK 维度顺序先由 backend 规范化，不能直接当成 NCHW。
query 来源字段运行时必须取得并校验；若 SDK 不提供，改为 manifest 的明确值及来源，不能保留空值。
letterbox 的示例取整规则是拟议公共 CPU 规则，选定模型使用硬件转换时须以参考向量验证相同语义。
标签文件属于资源清单，顺序必须对应 classes；完整 manifest 由其 profile 中的 hash 校验。

## 9. IPC、预览缓冲和 GUI 数据模型

### 9.1 当前阶段：进度、结果与状态

NPU worker 向 Service 发布状态、进度和测量值，ResultSink 消费有界的帧引用与检测结果。
每条生命周期/操作消息带 run_id、worker_generation、operation_id、stage、单调时间及事件序号。
Supervisor 丢弃旧 generation、重复结算和已超时 operation 的迟到成功消息。

每个可能阻塞的阶段调用前发布 BEGIN，结束时发布 END/ERROR；Service 保存开始时间和 deadline。
启动命令一发出就有响应 deadline，不能等 worker 自报 BEGIN 才开始监督。heartbeat 只证明通信线程
活着；RUNNING 还分别监督每路 Camera 到帧进度、整链路完成进度和当前阶段 deadline。
同机进程使用同一单调时钟域，重启 generation 后不可复用旧操作状态。

帧来源 PTS 单列，不能拿它当阶段 deadline 的时间基准。IPC 写入有界，持续无法发布关键进度也按
失联/无进度失败处理；不能在 vendor 调用前无限等发送队列。

检测结果至少包含 camera_id、source_frame_sequence、本地采集/完成时间、原始尺寸、实际几何变换、
有限长度 Detection 列表及阶段耗时。结果是独立数据，不暴露已释放的 vendor 输出指针。
ResultSink 的慢消费者可丢弃过期结果/帧，不阻塞执行；发布失败/跳帧计数与推理失败区分。

状态示例（数值仅示意）：

```json
{
  "test_id": "npu.camera_detection",
  "run_id": "example-run",
  "worker_generation": 1,
  "operation_id": 302,
  "phase": "running",
  "backend": "renesas_drpai",
  "model_profile": "yolov5s_coco_640",
  "cameras": [
    {
      "id": "front",
      "capture_state": "streaming",
      "inference_fps": 5.1,
      "last_inference_ms": 19.4,
      "successful_inferences": 301,
      "failed_inferences": 0,
      "detections": 2,
      "frame_sequence": 1873,
      "result_sequence": 1869
    }
  ]
}
```

### 9.2 后续 Qt 接入：预览 transport

GUI 从 Service 获取状态快照，预览 transport 后续独立实现；当前可用有界日志/结果消费者验证
程序逻辑。视频帧不得作为大块 CBOR 数据每帧经 Service 转发。

若采用共享内存，必须实现跨进程槽位借用/归还或等价的安全快照协议，覆盖慢读者和崩溃读者回收。
“写非活动槽再切换下标”不能阻止生产者绕回覆盖仍在读取的槽，不能单独作为正确性保证。
槽位数、消费者数、持帧配额及重连 generation 有限且可验证；无可用槽就跳过本次预览发布。

预览可缩放到固定上限，原始采集/模型输入不受卡片尺寸影响。检测结果必须匹配 Camera 和帧序号；
如果画面是更新帧，只能明确展示“最近结果”或不叠框，不能把未过期等同于属于当前帧。
Service 控制共享资源权限和回收，GUI 断开不阻塞 Run；重连只读当前 generation 的最新数据。
DMA-BUF 优化后置，不改变公共 ResultSink 与控制协议。

## 10. 并发、背压和公平性

- capture worker 永不等待 NPU 完成；
- inference worker 每次只处理一帧；
- 每路最多保留有限帧，旧帧被覆盖并计数；
- 双摄使用固定轮转顺序，若一路暂时没有新帧，另一条路可以继续，但故障路仍受 frame timeout；
- 后处理可以和下一路 capture 并行，但第一版不要求把同一个 backend context 的 `run()` 并行化；
- GUI 只显示最新有效结果，不补播积压帧；
- metrics 仍遵循全局采样/flush 策略，不能每帧 fsync；
- 详细 detection 列表应限长或聚合，防止长时间运行报告无限增长。

Camera、latest-frame、结果消费者和预览各自的跳帧独立计数。源端错误/丢帧仅在驱动或 pipeline
实际提供证据时记录，不能拿“采集帧数减推理次数”冒充 Camera 丢帧。静态画面像素不变不是失流。
阶段时延分别计量；backend run 含提交、等待、取输出和同步，能取得的纯硬件时延另列。
利用率不可得时标记 unsupported，不从时延比例推算。资源/内核日志均按有界采样策略进入报告。

## 11. 停止与恢复

正常停止按资源依赖执行，不能统一规定“先释放所有 DMA，再销毁所有 context”：

1. Service 将实例状态改为 `STOPPING`；
2. 设置 stop 标志，唤醒取帧/限速等待，禁止新整链路尝试和新的消费者持帧；
3. 监督 in-flight 操作在原 deadline 或 stop grace 的较早期限内结束；只有 SDK 明确允许且已验证
   时才请求 native cancel，不从其他线程擅自销毁 context；
4. 操作结束后在所属线程释放 OutputLease 和相关输入引用；按依赖停止 Camera、等待采集线程并
   归还 FrameHandle，处理结果消费者尚持有的引用；
5. 由 backend.close 按所选 SDK 顺序清理 network/context/buffer/runtime，包括部分初始化失败；
6. 记录 cleanup 与 partial/最终窗口结果，结束 worker，Service 确认退出后释放进程级资源。

所有清理阶段受 Service deadline 约束。未在宽限期返回时 Service 请求终止 worker，再在有限
等待后强制终止；操作按失败结算并锁定原因为超时/崩溃，迟到成功不能覆盖。不能在失控线程仍运行
时让另一线程继续释放 vendor 内存，也不能在线程级强杀后复用 context。

进程终止后单独检查设备可用性。若 kernel 不可中断等待导致旧 worker 未退出，或设备需 reset，
记录 `DEVICE_UNHEALTHY/RESET_REQUIRED`、保留诊断并拒绝重叠 Retest。不假定 kill 会重置 NPU，
不对不支持安全复位的平台自动无限重试。板级恢复由已验证的运维/BSP 机制执行。

Retest 必须重新创建 Camera pipeline 和 backend context。不能把发生 timeout 或 runtime failure 的
context 直接带入下一次 Run。进程崩溃或掉电仍由 Stability Test 通用 run-state 恢复逻辑把旧 Run
结算为 `ABORTED`；新 Run 不承担恢复旧 NPU context 的责任。

## 12. 实施顺序

1. 三类 backend 共同收口 Frame/InputBinding/TensorDesc/OutputLease/Result、模块 ABI、模型
   manifest 和 YAML Schema；固定五个平台的 BSP/runtime/工具链/模型/Camera 组合；
2. 实现公共装载入口与三类最小 backend，尽早验证各平台的编译和接口映射；当前先在可用的
   RZ/V2H 或 V2N 上运行相同参考输入，不能先形成 Renesas 专用业务循环再事后抽取；
3. 同一入口验证 load/run/output、真实 NPU 执行证据、参考结果、错误传播及关闭；mock 仅用于
   契约和故障注入，不作为某个平台已支持 NPU 的证据；
4. 完成共享单/双摄 latest-frame、轮转、预处理/解码、窗口判定和日志 ResultSink；无需先做 UI；
5. 验证慢消费者、帧生命周期、零工作量、窗口边界、启动/运行/清理卡住、崩溃、失流、Stop/Retest；
6. 五个平台分别完成实际 Camera、连续检测及与其他压力项并发验证，确定各板门限；
7. 全部必需能力通过统一矩阵后发布同一产品版本；Qt 卡片及预览 transport 后续集成验证。

当前验收矩阵如下，公开源码可行不等于本项目已实现：

| 项目 | RZ/V2H | RZ/V2N | RK3576 | T527 | A733 |
| --- | --- | --- | --- | --- | --- |
| 本项目 BSP/runtime/model/Camera 组合 | 待固定 | 待固定 | 待固定 | 待固定 | 待固定 |
| backend 构建及按 YAML 装载 | 待实现/验证 | 待实现/验证 | 待实现/验证 | 待实现/验证 | 待实现/验证 |
| 固定输入数值/检测参考及真实 NPU 执行 | 待验收 | 待验收 | 待验收 | 待验收 | 待验收 |
| 单/双摄连续循环、统计与结果 | 待验收 | 待验收 | 待验收 | 待验收 | 待验收 |
| 异常、超时、清理、Retest 与并发长稳 | 待验收 | 待验收 | 待验收 | 待验收 | 待验收 |

每列绑定产品提交、资源 hash 和测试证据，不能以同家芯片或同一 API 代替另一列的验证。

## 13. 待确认事项

已确定：优先 Renesas 验证、五个平台统一版本、硬件与库由 YAML 选择、先完成逻辑后集成 UI。
以下是工程/板端数据，尚不能作为硬编码默认值：

1. 主工程的 Service/worker/配置/报告接口及工具链集成位置；当前目录仅包含文档；
2. 各板 Camera 设备、格式、分辨率、帧率和 GStreamer pipeline；
3. 每个平台最终 BSP/runtime/compiler 版本、模块构建依赖、模型 hash、binding 和参考向量；
4. 静态图片自检是否默认启用，以及标准图片和期望检测结果；
5. 单摄/双摄目标总推理 FPS、单次 timeout 和每个统计窗口最低成功次数；
6. 是否需要固定目标物作为生产治具，以及哪些类别参与判定；
7. 所查 A733/T527 target 与项目实际 compiler/BSP 组合的验证、超时后设备恢复能力；
8. Renesas、RKNN、VIPLite 是否能提供可靠 NPU utilization；不可读取时不伪造利用率；
9. 是否保存限量故障截图；默认不保存视频和截图；
10. 如后续提出 `parallel`/DMA-BUF/独立诊断模式需求，再验证其能力和内存预算，不作为当前前置项。

## 14. 参考实现与验证依据

- Renesas R01 用 YOLOv3，多摄示例用 YOLOX-l；多摄线程结构可参考，路数/传感器支持按具体平台
  和 SDK 验证，不概括为所有目标板均支持四路；
- Renesas 官方 YOLOv5s 640 导出/模型表提供匹配逻辑契约的路径；C++ Wrapper 需 bridge，部分
  runtime 分支元数据接口有限；
- Rockchip Model Zoo 明确支持 RK3576；UINT8/NHWC 提交、模型转换归一化及已激活 head 必须匹配；
- Allwinner 旧 ai-sdk 与所查 Model Zoo 的预处理/输出布局不同；T527/A733 的 API 库导出共同
  VIP 接口，模型和依赖按 target 区分；
- Allwinner 公开下载镜像与页面标注版本存在差异，本设计依据已核实包，不把网页示例当成实际
  模型的 IO 查询结果。

固定提交链接、归档和 ONNX hash、API/ELF 检查结果统一保存在
[20_npu_platform_compatibility.md](20_npu_platform_compatibility.md)，作为本设计的平台事实依据。

