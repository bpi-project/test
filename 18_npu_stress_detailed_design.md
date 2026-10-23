# NPU 与多摄像头稳定性测试详细设计

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
7. 平台错误保留原始信息，同时映射为跨平台统一错误码。

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
  ├─ LatestFrameStore × N
  ├─ InferenceWorker（一个）
  ├─ PlatformBackend
  │   ├─ RenesasDrpAiBackend
  │   ├─ RockchipRknnBackend
  │   └─ AllwinnerVipLiteBackend
  ├─ SharedYoloV5PostProcessor
  └─ PreviewSharedBufferPublisher
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
  → COMPLETED / FAILED / CANCELLED
```

- `PROBING`：检查设备节点、runtime、backend 能力、模型和标签；
- `LOADING_MODEL`：创建一个 backend context 并校验输入输出契约；
- `OPENING_CAMERAS`：按配置启动所有 pipeline，等待每路首帧；
- `OPTIONAL_SELF_TEST`：使用标准图片执行一次确定性推理；
- `RUNNING`：Camera 持续采集，NPU 连续逐路推理；
- `STOPPING`：停止新任务并按受控顺序清理。

正常 cycle 边界只清零本轮计数并生成 cycle record，不关闭 Camera、不卸载模型。这样避免周期性
重建 runtime 引入不属于硬件老化目标的会话建立噪声。

### 3.2 单轮状态

建议 `schedule.cycle_mode: timed`。每个 cycle 是固定统计窗口：

```text
cycle start
  → reset per-cycle counters
  → continuous capture/inference
  → evaluate thresholds at cycle end
  → write cycle record
  → optional interval
  → next cycle
```

具体默认周期时长、timeout 和最低 FPS 需要各平台实测后确定，文档当前不设未经验证的硬门限。

## 4. 双摄数据流和调度

### 4.1 Camera worker

每路 Camera 有独立 capture worker 和固定容量缓冲：

```text
Capture frame
  → validate timestamp/size
  → replace latest-frame slot
  → increment captured_frames
  → publish preview at capped rate
```

替换最新帧时允许统计 `overwritten_frames`。推理线程取得 frame handle 后必须拥有稳定生命周期，
capture worker 不得覆盖正在处理的存储；可使用双缓冲、引用计数帧或小容量 lock-free/latest queue。

### 4.2 串行逐路推理

第一版只有一个 `InferenceWorker` 和一个 `PlatformBackend` context：

```text
while running:
  for camera in configured_order:
    frame = camera.latestFrameAfter(lastSubmittedSequence[camera])
    if frame unavailable:
      evaluate frame timeout
      continue
    preprocess(frame)
    backend.run(input)
    canonicalizeOutputs()
    postprocess()
    publish result tagged with camera ID and frame sequence
```

双摄顺序由产品配置引用的 Camera 列表决定，不能依赖 `/dev/video*` 字典序。每个完整调度轮按顺序
覆盖所有启用 Camera；不能因 Front Camera 一直有新帧而饿死 Rear Camera。

该方式持续向 NPU 提交任务，仍可形成高 NPU 负载。两个线程同时调用同一 runtime 并不保证硬件
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

建议内部 C++ 接口：

```cpp
class INpuBackend {
public:
    virtual ProbeResult probe(const BackendConfig &) = 0;
    virtual Result loadModel(const ModelArtifact &, ModelMetadata *) = 0;
    virtual Result validateContract(const ModelContract &) = 0;
    virtual Result run(const InputTensor &, OutputTensorSet *,
                       std::chrono::milliseconds timeout) = 0;
    virtual BackendMetrics metrics() const = 0;
    virtual Result close() = 0;
    virtual ~INpuBackend() = default;
};
```

接口禁止使用 `void run()`。所有平台调用必须返回统一 `Result`，并携带：

- 统一错误码；
- 失败阶段；
- vendor 原始状态码；
- 受限长度的错误文本；
- 是否可重试；
- context 是否仍可继续使用。

如果 vendor API 本身不提供超时，应在专用 worker 中执行并由 watchdog 监督。超时后不能粗暴在
未知线程中销毁 context；应按平台安全方式请求退出，超过 grace period 后将本轮和资源状态标记
为失败，必要时停止该测试实例。

## 6. 各平台适配

### 6.1 Renesas DRP-AI/RUHMI

- RZ/V2H、RZ/V2N 使用对应平台编译模型；
- 复用平台 PreRuntime、DMA buffer 和 DRP-AI runtime 能力；
- 模型只加载一次；
- 双摄参考 `multi_camera_vehicle_detection`：多路同时采集，一个推理线程在 Camera 循环中依次
  调用 `runtime.Run()`；
- 不直接复制示例全局变量结构，应封装 Camera ID、buffer 和统计状态；
- 输出在进入共享 YOLOv5 后处理前转换为 canonical tensor。

### 6.2 Rockchip RKNN

- RK3576 使用经 RKNN Toolkit 转换的 `.rknn`；
- probe 需记录 RKNN runtime/driver 版本和目标 NPU core 能力；
- 量化输出必须使用 RKNN 提供的 scale、zero-point 和 tensor layout；
- 第一版一个 RKNN context；是否指定 NPU core mask 由板级 profile 定义，不写死在插件；
- 官方优化模型会把不利于量化的 YOLO decode 移到 CPU，正好适配统一三头后处理。

### 6.3 Allwinner VIPLite

- T527：NPU v2、VIPLite v1.13、v2 `.nb`；
- A733：NPU v3、VIPLite v2.0、v3 `.nb`；
- 两个平台可以共用 `AllwinnerVipLiteBackend` 源码，但不能假定 runtime 库和 `.nb` 通用；
- backend 必须检查 `vip_init`、create/prepare network、buffer、run 和 cache flush 的返回值；
- SDK 示例 `awnn_run()` 返回 `void`，正式实现不能原样作为错误边界，应包装底层 VIP API 或维护
  一个能返回状态的私有适配层；
- 当前 SDK 示例把输入复制到 VIP buffer、输出转换成 float；第一版可沿用有界复制路径，后续再
  在验证 DMA-BUF API 和 cache 一致性后优化零拷贝；
- VIPLite 与 Unified driver/runtime 的装载策略由板级镜像明确指定，测试进程不能在运行中自动
  混用或切换；
- SDK 自带 A733 v3 模型，但转换脚本没有显式列出 A733 target。重新转换 A733 模型前必须从
  Allwinner 确认正确 compiler optimize target，不能自行套用 T527 参数。

## 7. 共享预处理和后处理

### 7.1 预处理

共享预处理逻辑定义：

1. 校验输入帧格式和尺寸；
2. 转换为 RGB；
3. 按原始宽高比缩放；
4. 对剩余区域 letterbox；
5. 转换成平台需要的 NCHW/NHWC 和量化格式；
6. 保存 scale、padding、Camera ID、frame sequence 和 timestamp。

平台硬件预处理器可以替代 CPU 实现，但输出语义必须一致。报告应记录实际采用
`cpu|hardware` 预处理和耗时。

### 7.2 Canonical 输出

平台后端负责把 native tensor 描述为：

```text
head stride=8  shape=3×80×80×85
head stride=16 shape=3×40×40×85
head stride=32 shape=3×20×20×85
```

canonicalization 必须处理 FP32、FP16、INT8/UINT8 以及平台 layout 差异。发生 NaN、Inf、大小
不足、重复 head 或无法匹配 stride 时返回 `INVALID_OUTPUT` 或 `MODEL_CONTRACT_MISMATCH`。

### 7.3 后处理

共享后处理完成 sigmoid、anchor decode、confidence threshold、NMS 和坐标反变换。后处理参数来自
受 Schema 限制的配置，不从平台示例源码隐式继承。结果必须带原始 frame sequence，GUI 只把
结果叠加到同一路 Camera，并检查结果是否过期。

## 8. 配置设计

### 8.1 板级配置

板级配置描述硬件与平台差异：

```yaml
hardware:
  cameras:
    devices:
      - id: front
        pipeline_profiles:
          x11:
            pipeline: >-
              v4l2src device=/dev/video22 ! ... ! appsink name=camera_sink
          wayland:
            pipeline: >-
              v4l2src device=/dev/video22 ! ... ! appsink name=camera_sink
      - id: rear
        pipeline_profiles:
          x11:
            pipeline: >-
              v4l2src device=/dev/video31 ! ... ! appsink name=camera_sink

  npu:
    devices:
      - id: default
        backend: allwinner_viplite
        runtime_generation: v1.13
        device_nodes: [/dev/vipcore]
        model_profiles:
          - id: yolov5s_coco_640
            contract: yolov5s-coco-640-raw3head-v1
            model: /usr/share/stability-test/models/t527/yolov5s.nb
            labels: /usr/share/stability-test/models/coco80.txt
            sha256: "<packaged-model-sha256>"
```

实际 Camera pipeline、设备节点、runtime 世代和模型路径属于板级/BSP 差异，不能写死在插件。
pipeline 字符串只允许来自受信任板级配置，不允许普通产品配置或 GUI 输入任意 shell 命令。

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
    target_total_inference_fps: 10
    per_inference_timeout_ms: 2000
    frame_timeout_ms: 3000
    preview_max_fps: 30
    result_max_age_ms: 1000
    startup_self_test:
      enabled: false
      image: null
      expected_classes: []
      minimum_detections: 0
    postprocess:
      confidence_threshold: 0.4
      nms_threshold: 0.45
    detection_requirement:
      enabled: false
      required_classes: []
      minimum_detection_count_per_cycle: 0
  criteria:
    minimum_inferences_per_camera_per_cycle: null
    minimum_inference_fps_per_camera: null
    minimum_success_rate_percent: 100
    maximum_consecutive_inference_failures: 1
    maximum_frame_stalls_per_cycle: 0
    max_consecutive_failed_cycles: 1
```

示例中的时长和 FPS 仅说明字段结构，除已经确认的 640×640 模型契约外，不作为 AI2H/T527/
A733/RK3576 的最终默认门限。正式产品 YAML 应由板端实测结果确定。

### 8.3 语义校验

配置加载时至少校验：

- `cameras` 非空、无重复且均存在于 board profile；
- 第一版 Camera 数量只能为 1 或 2；
- backend、runtime generation 与模型 profile 兼容；
- 模型文件、标签和自检图片存在且 SHA-256 匹配；
- 逻辑模型 contract 是程序支持的版本；
- timeout、目标 FPS 和成功率范围有效；
- `startup_self_test.enabled=true` 时图片、期望类别和最低数量有效；
- `detection_requirement.enabled=true` 时类别和检测数量有效；
- 第一版只接受 `sequential_per_camera`，不能悄悄接受未实现的 `parallel`。

## 9. IPC、预览缓冲和 GUI 数据模型

NPU worker 向 Service 发布低频状态和测量值；GUI 从 Service 获取状态快照，并按 Service 授权的
只读共享内存描述映射每路最新预览。状态示例：

```json
{
  "test_id": "npu.camera_detection",
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

视频帧不得作为大块 CBOR 数据每帧经过 Service 转发。第一版每路使用固定双槽共享内存环：

- worker 只写非活动槽，写完像素、宽高、stride、format、sequence 和 timestamp 后原子切换；
- GUI 以只读方式映射，只显示 sequence 完整提交的最新槽；
- 预览使用经过缩放的固定上限尺寸，原始采集帧和 640×640 NPU 输入不通过 GUI IPC；
- 检测框通过普通 IPC 发送，并携带 Camera ID、frame sequence 和坐标映射参数；
- Service 创建随机共享内存名称/令牌并控制权限，Run 结束后回收；
- GUI 断开不阻塞 worker，重连后只消费最新帧，不补播历史帧。

后续如平台能够可靠跨进程传递 DMA-BUF FD，可增加零拷贝 preview transport；第一版不以此为
前置条件。IPC 只传递状态、指标、检测 metadata、共享内存描述和生命周期命令。

## 10. 并发、背压和公平性

- capture worker 永不等待 NPU 完成；
- inference worker 每次只处理一帧；
- 每路最多保留有限帧，旧帧被覆盖并计数；
- 双摄使用固定轮转顺序，若一路暂时没有新帧，另一条路可以继续，但故障路仍受 frame timeout；
- 后处理可以和下一路 capture 并行，但第一版不要求把同一个 backend context 的 `run()` 并行化；
- GUI 只显示最新有效结果，不补播积压帧；
- metrics 仍遵循全局采样/flush 策略，不能每帧 fsync；
- 详细 detection 列表应限长或聚合，防止长时间运行报告无限增长。

## 11. 停止与恢复

受控停止顺序：

1. Service 将实例状态改为 `STOPPING`；
2. 禁止 capture worker 向推理队列提交新帧；
3. 等待正在进行的单次推理在 grace period 内完成；
4. 停止 Camera pipeline 并等待 capture worker；
5. 释放预处理、output 和 DMA buffer；
6. 销毁 backend context；
7. 关闭 runtime；
8. 写入最终 cycle 和 cleanup 结果。

Retest 必须重新创建 Camera pipeline 和 backend context。不能把发生 timeout 或 runtime failure 的
context 直接带入下一次 Run。进程崩溃或掉电仍由 Stability Test 通用 run-state 恢复逻辑把旧 Run
结算为 `ABORTED`；新 Run 不承担恢复旧 NPU context 的责任。

## 12. 实施顺序

1. 定义模型 contract、配置 Schema、统一错误码和 mock backend；
2. 完成单摄 latest-frame、共享预处理/后处理、指标和无视频报告测试；
3. 完成双摄同时采集、左右预览、单 context 串行逐路推理；
4. 先实现第一参考平台 Renesas RZ/V2H/V2N backend；
5. 增加 Allwinner T527 backend，并验证 v2 `.nb`；
6. 增加 Allwinner A733 v3 profile，确认 compiler/runtime 目标；
7. 增加 RK3576 RKNN backend；
8. 完成单项长稳、双摄、并发满负载、Stop/Retest、错误注入和报告验收；
9. 平台实测后再确定默认 FPS、timeout、成功率和连续失败门限。

## 13. 待确认事项

以下内容尚未收口，不应作为当前硬编码默认值：

1. 第一实现平台是在 RZ/V2H/V2N 先落地，还是先使用已有配件最完整的板卡；
2. 各板 Camera 设备、格式、分辨率、帧率和 GStreamer pipeline；
3. 每个平台最终模型文件、量化类型、模型来源与许可证合规；
4. 静态图片自检是否默认启用，以及标准图片和期望检测结果；
5. 单摄/双摄目标总推理 FPS、单次 timeout 和每轮最低成功次数；
6. 是否需要固定目标物作为生产治具，以及哪些类别参与判定；
7. Allwinner A733 模型转换器对应的准确 optimize target；
8. Renesas、RKNN、VIPLite 是否能提供可靠 NPU utilization；不可读取时不伪造利用率；
9. 是否保存限量故障截图；默认不保存视频和截图；
10. `parallel` 多 context 模式的实际平台需求和内存预算。

## 14. 参考实现与验证依据

- Renesas `multi_camera_vehicle_detection`：支持一至四路采集，但使用一个推理线程和一个
  Camera 循环逐路执行 runtime；
- Allwinner `examples/yolov5`：640×640 RGB/NCHW 输入、三层 raw head、CPU YOLOv5 decode/NMS；
- Allwinner T527 `NPU_VERSION=v2/NPU_SW_VERSION=v1.13`，A733
  `NPU_VERSION=v3/NPU_SW_VERSION=v2.0`；
- Rockchip 官方 RKNN Model Zoo：RK3576 支持 YOLOv5，优化模型将量化不友好的 decode 移到
  CPU 后处理；
- Renesas 官方 RZ/V2H/RZ/V2N 模型列表：已验证 YOLOv5s 640×640。

