# NPU 与多摄像头稳定性测试需求

修订日期：2026-10-23。已合入多平台评审结论；实现细节见 [详细设计](18_npu_stress_detailed_design.md)，
公开源码与模型依据见 [平台核查](20_npu_platform_compatibility.md)。本文规定目标行为，不表示已经通过真机验收。

## 1. 目标

新增可配置的 `npu_inference` 测试插件，在长时间 Stability Test 中同时验证：

1. NPU 驱动、用户态 runtime 和平台模型能否持续稳定运行；
2. Camera 采集、图像预处理、NPU 推理、CPU 后处理和 GUI 显示整条链路；
3. 单摄或双摄持续出流时的帧推进、推理成功率、时延和资源释放；
4. 与 CPU、DDR、GPU、VPU、Storage 和 Network 等压力项目并发时，是否出现超时、崩溃、
   内存泄漏、帧停滞或 runtime 错误；
5. 同一套上层测试框架通过平台后端适配 Renesas、Rockchip 和 Allwinner NPU。

第一版检测算法统一采用 YOLOv5s、640×640 输入和 COCO 80 类语义，但不要求不同 SoC 使用
同一个模型二进制。统一的是测试流程、模型逻辑契约、后处理、UI、指标和报告；模型文件与
runtime 必须按平台生成和部署。

最高优先级是与主稳定性程序保持同一产品版本、公共源码、子项 ID、控制协议和判定逻辑，
不能按平台分成两套业务实现。RZ/V2H、RZ/V2N 是当前优先验证环境，五个平台均属于交付范围。
某个平台尚未适配属于未完成，不能通过隐藏必测项、静默 CPU 推理或另发专用版本满足兼容要求。

硬件与部署差异统一由板级 YAML 表达，包括 pipeline、设备节点、backend 模块、厂商库及依赖
路径、模型资源和硬件执行参数；程序按已定义的 backend API 调用，不把厂商调用流程写成 YAML 脚本。
如 BSP 工具链不同，允许从同一源码及产品版本构建对应产物，沿用主程序发布规则。

当前阶段先完成持续采集、检测、停止、指标与判定。最终嵌入 Qt 主界面卡片；本轮不以 UI 布局、
绘制或预览传输优化为程序逻辑的前置条件，先通过统一 ResultSink 输出帧引用、检测结果和状态。

## 2. 支持范围

### 2.1 第一阶段目标平台

| 平台 | 推理后端 | 平台模型产物 |
| --- | --- | --- |
| Renesas RZ/V2H、RZ/V2N | DRP-AI/RUHMI runtime | 平台编译模型目录 |
| Rockchip RK3576 | RKNN Runtime | `.rknn` |
| Allwinner T527 | VIP API；所查组合为 VIPLite v1.13、NPU v2 | T527 target `.nb` |
| Allwinner A733 | VIP API；所查组合为 v2.0、NPU v3 | A733 target `.nb` |

目标镜像负责提供匹配 BSP 的 NPU 驱动和用户态 runtime。部署包提供 Stability Test 插件、
共享后处理、平台适配层、配置文件及经校验的平台模型，不允许运行时在线下载模型。
三个 backend 家族为 Renesas、RKNN、VIP；T527/A733 优先共用 VIP 适配源码并配置不同库及模型。
兼容范围按实际 BSP/runtime/compiler/model 组合记录，不仅按 SoC 名称或库名判断。

### 2.2 摄像头范围

- 第一版必须支持单摄和双摄；
- 双摄必须同时保持采集和预览；
- 第一版使用单个模型实例，在同一轮中按配置顺序对各路最新帧逐路推理；
- 架构允许以后扩展至四路 Camera，但四路不是第一版验收条件；
- 第一版不要求多个模型实例或多个 NPU context 真正并行执行；
- 后续可在平台明确支持多 context、多 NPU core 或异步队列时增加 `parallel` 策略。

## 3. 测试输入模式

### 3.1 摄像头连续检测

正式长稳测试的主要模式为摄像头实时预览和连续目标检测：

```text
Camera 连续采集
  → 取得最新完整帧
  → RGB 转换、保持比例缩放和 letterbox 到 640×640
  → 平台 NPU 推理
  → 共享 YOLOv5 解码和 NMS
  → 将检测框、类别和置信度映射回预览画面
  → GUI 显示并记录指标
```

预览帧率和推理帧率必须解耦。例如摄像头可按 30 fps 采集和预览，而 NPU 每路只选择部分帧
进行推理。未被选择的预览帧不得进入无限队列。

### 3.2 静态图片启动自检

允许通过配置启用内置标准图片启动自检。它用于确认：

- runtime 和模型可以加载；
- 输入、输出数量和 tensor 形状符合模型契约；
- 预处理、推理和后处理结果可以得到预期类别；
- 模型包没有损坏或错配。

静态图片自检不是正式老化负载，也不能替代摄像头连续检测。是否默认启用、标准图片及期望类别
尚需结合各平台最终模型确认，因此第一版配置接口必须保留，但不得把未提供的图片写成硬依赖。

## 4. 双摄行为

双摄测试采用“同时采集、同时提供预览帧、单实例公平轮转逐路推理”模式。
每路在轮到自己时取最新未处理帧，不要求两路同时曝光或同时快照：

```text
Front Camera ─→ 最新帧槽 ─┐
                           ├→ Front 推理 → Rear 推理 → 下一轮
Rear Camera  ─→ 最新帧槽 ─┘
```

这不是一轮只测 Front、下一轮只测 Rear。每个完整检测轮都必须覆盖所有已启用 Camera：

```text
Round 1: front → rear
Round 2: front → rear
Round 3: front → rear
```

术语统一：`run` 是一次启动到停止；`round` 是按顺序访问各路一次；`cycle/window` 是框架的
统计与判定时间窗口。一路暂时无新帧可以跳过，仍独立检查其失流；不得重复推理旧帧制造工作量。

两路 Camera 分别维护采集帧数、预览发布帧数、提交推理帧数、成功/失败次数、连续失败次数、推理
时延、最后帧时间和最后检测结果。一路失败不能被另一路成功掩盖。

Renesas `multi_camera_vehicle_detection` 提供同一推理线程逐路调用 runtime 的参考，但使用
YOLOX-l；`R01_object_detection` 使用 YOLOv3。它们的模型及解码不能直接代替本项目 YOLOv5。

## 5. 模型逻辑契约

第一版定义逻辑模型契约 `yolov5s-coco-640-raw3head-v1`：

```yaml
logical_input:
  shape: [1, 3, 640, 640]
  color: rgb
  layout: nchw
  letterbox: true

output:
  type: yolov5_raw_heads
  strides: [8, 16, 32]
  anchors_per_head: 3
  classes: 80
  attributes_per_anchor: 85

postprocess:
  confidence_threshold: 0.4
  nms_threshold: 0.45
  nms_mode: class_aware
  multi_label: false
```

三个逻辑输出分别对应：

- stride 8：`3 × 80 × 80 × 85`；
- stride 16：`3 × 40 × 40 × 85`；
- stride 32：`3 × 20 × 20 × 85`。

上述输入是算法逻辑描述，实际提交格式由模型 profile 的 `input_binding` 定义，可以是
FP32/FP16 NCHW 或 UINT8 打包 RGB 等。必须明确颜色、数值范围、归一化由应用还是模型/runtime
执行、letterbox 补边和取整规则，避免重复归一化。

`raw3head` 表示尚未完成坐标解码/NMS 的三个 head，不强制它们都处于 sigmoid 之前。
每个模型必须明确 xywh、objectness、classes 的激活状态，公共后处理只执行一次 sigmoid。
平台后端可返回不同 dtype、维度顺序和量化 tensor，须提供实际 shape、字节 stride、量化类型
及所需参数。量化至少区分 none、affine、dynamic fixed point；未实现的类型明确拒绝。

输出通过显式 head 映射接入统一逻辑视图，不要求每帧产生完整 FP32 重排副本。映射按名称或
经模型 hash 固定的 index，并校验数量、维度及 stride 对应关系；不得依赖未验证的输出顺序。
runtime 无法查询的信息可以由经校验的 manifest 提供，但必须记录来源，不能把元素数校验
宣称为完整 shape/layout 校验。

平台允许使用针对量化友好性裁剪过的 YOLOv5 ONNX，只要逻辑输出仍满足该契约。不同平台模型
无需拥有相同文件格式或完全相同的图结构。

模型 manifest 使用 YAML，可内联在 profile 或作为其引用资源。至少记录源模型/产物 hash、
转换工具及 target、输入 binding、输出映射/激活、anchors、类别顺序和参考向量/容差。
多文件模型校验整个资源清单，不能只校验其中一个 `.so`。研发及发布验收必须验证参考输入的
数值与检测结果；现场启动自检仍按 §3.2 配置，不强制每次运行执行参考图片。

## 6. 通过与失败规则

### 6.1 默认通过依据

默认长稳测试不要求画面中持续存在可识别目标。每路 Camera 的每个完整统计窗口通过至少满足：

1. Camera 已成功打开且采集帧持续推进；
2. 获得有效、非空的新输入帧；保留可取得的源序号/PTS，并使用本地单调时钟监测到帧进度；
3. 预处理成功；
4. NPU 推理在单次超时内成功完成；
5. 输出数量、形状和数值有效；
6. 后处理成功完成；
7. 本窗口有效完成次数、整链路成功率和有效 FPS 达到配置门限；
8. 没有命中连续失败、帧停滞、runtime 崩溃或资源无法释放条件。

未检测到任何目标不是默认失败条件，因为实际现场可能暂时为空场景。

每路完整窗口必须至少完成一次有效检测链路；未配置最低次数/FPS 时也不允许零工作量通过。
`0/0` 成功率显示为无数据，不是 100%。成功率按已结束的整链路尝试计算，预处理失败也计入；
跨窗口未完成项单列，按完成或超时结算时间归属窗口。启动/自检不计入正式工作量。
提前停止的不完整窗口保留统计并标记 partial，不用作满足完整时长的证据。
正常停止导致的取消单列，不伪造推理失败；已有错误或超时不能被随后停止请求改成取消。

`target_total_inference_fps` 是所有 Camera 合计的限速值，不是最低通过门限；最低要求放在
criteria 中。持续端到端检测不承诺 NPU 始终满载，不把 runtime 耗时占比命名为 NPU 利用率。

### 6.2 可选目标物校验

工厂提供固定目标物时，可以显式启用检测正确性要求：

```yaml
detection_requirement:
  enabled: true
  required_classes: [person]
  minimum_detection_count_per_cycle: 1
```

只有启用 `detection_requirement.enabled` 后，未检测到指定类别才参与 PASS/FAIL。检测类别、数量和最大置信度
无论是否参与判定都应写入报告。
启用时，每路 Camera 的每个 required class 均须达到最低累计检测数；统计各帧阈值/NMS 后的框，
不表示跨帧去重后的物体数量，也不能用另一 Camera 的目标数抵消本路不足。

### 6.3 失败类型

至少区分：

- `CAPABILITY_UNSUPPORTED`：runtime、模型格式或必需能力不支持；
- `RUNTIME_LOAD_FAILED` / `BACKEND_ABI_MISMATCH`：库、依赖、必需符号或模块 ABI 不匹配；
- `MODEL_NOT_FOUND`：模型或标签资源缺失；
- `MODEL_INCOMPATIBLE`：模型与 NPU/runtime 世代不匹配；
- `MODEL_CONTRACT_MISMATCH`：输入输出数量、shape、layout 或类别数不符；
- `CAMERA_OPEN_FAILED`：摄像头无法打开；
- `CAMERA_FRAME_TIMEOUT`：指定时间内没有新帧；
- `PREPROCESS_FAILED`：颜色转换、缩放或 letterbox 失败；
- `INFERENCE_FAILED`：NPU runtime 返回错误；
- `INFERENCE_TIMEOUT`：单次推理超时；
- `STAGE_TIMEOUT` / `WORKER_EXITED`：其他阶段超时或 worker 异常退出；
- `INVALID_OUTPUT`：输出为空、包含非法数值或无法反量化；
- `POSTPROCESS_FAILED`：YOLO decode 或 NMS 失败；
- `PERFORMANCE_BELOW_THRESHOLD`：有效 FPS 或成功率低于门限；
- `DETECTION_CRITERIA_FAILED`：显式目标物条件未满足；
- `CLEANUP_FAILED`：Camera、runtime、DMA buffer 或 worker 无法有序释放；
- `DEVICE_UNHEALTHY` / `RESET_REQUIRED`：故障后设备不可继续使用或需要板级恢复。

失败必须写入错误阶段、可取得的平台原始错误码、相关 Camera ID 和窗口指标；无厂商状态码时
显式标记不可取得。按配置决定何时停止可恢复错误的测试实例；已判定失败的必测项锁定全局最终 `FAIL`，其余测试项
默认继续，除非命中全局安全停止规则。

Service 的阶段 deadline 与工作进度监督独立于 worker 心跳。支持时使用厂商 timeout/cancel，
仍须监督加载、取输出和清理等阶段。终止 worker 不等于硬件已恢复；未退出的旧 worker 或未恢复
的设备不得启动重叠 Retest，不承诺用户态可以解除所有内核不可中断等待。

## 7. 性能与资源约束

- NPU 推理不能阻塞 GUI 线程或 Camera capture 线程；
- 每路 Camera 使用有界 latest-frame 缓冲及帧池；仍被推理/结果消费者持有的存储不能覆盖；
- pipeline/appsink 内部队列同样有界，不能仅限制应用层队列；
- 推理落后时丢弃旧帧，不积累延迟和内存；
- GUI 刷新频率与 Camera/NPU 频率解耦，避免检测框和图像更新拖慢测试；
- 第一版不绘制 NPU 实时曲线，使用实时值、轮次统计和详细报告，避免重现高频 QML 更新干扰
  VPU 的问题；
- Camera 采集分辨率、预览尺寸和模型输入尺寸分别配置，不因 UI 卡片较小而降低传感器测试输入；
- 同一 Run 中不得在统计窗口边界重载模型或重建 Camera pipeline；
- backend 初始化、运行、输出释放和关闭归同一推理线程；
- 停止时先禁止新尝试，等待或监督 in-flight 操作结束，再按资源依赖释放；不得从其他线程
  释放仍被 runtime 使用的输入、输出、Camera buffer 或 context；
- 慢 ResultSink 不得反压采集/推理，消费者持帧及待发布结果数量有上限；
- 观测暖机后 RSS/PSS、FD/线程及可取得的设备内存趋势，并记录清理后的结果；不可获取项标记 unsupported。

## 8. GUI 需求

本节规定后续 Qt 集成目标。当前程序逻辑阶段只保留稳定的结果/预览接口，UI 验收在集成阶段执行。

NPU 卡片必须显示：

- 当前状态和阶段：加载模型、等待 Camera、运行、自检、停止或失败；
- backend、模型 profile 和启用 Camera 数量；
- 每路 Camera 的采集状态、推理成功/失败次数、实时推理 FPS 和最近推理时延；
- 单摄时一个预览区域；双摄时左右两个预览区域；
- 检测框、类别和置信度；
- 最近错误的 Camera、阶段和简化原因。

双摄预览必须从测试开始就使用稳定的左右布局，不能在第二路出流或检测结果到达后重新挤压画面。
预览无新推理结果时可以沿用该路最近一次检测框，但必须能够根据结果时间戳清除过期框，不能让
操作员误认为旧结果属于当前画面。

卡片空间不足时优先保留实时画面、状态和错误；完整时延分布、类别统计和平台原始错误进入详情页
及报告，不在主卡片堆叠过多文字。

## 9. 报告需求

运行快照和最终报告必须保存：

- backend 名称、版本和平台 runtime 版本；
- 生效 YAML 快照、实际加载库/依赖路径及可取得的 driver/BSP 版本、硬件运行参数；
- 模型 profile、逻辑契约版本、模型文件 SHA-256、量化类型和标签版本；
- Camera profile、pipeline、采集格式、分辨率和帧率；
- 调度策略和 Camera 顺序；
- 每路采集、源端/pipeline 可观测丢帧、latest-frame 覆盖、推理跳帧和预览跳帧分别计数；
- 每路整链路尝试、runtime 调用、成功/失败及跨窗口 in-flight 数，预览发布数不冒充 GUI 显示数；
- 每路预处理、backend run（含取输出/同步）、输出适配、后处理及端到端时延；
- 每路实际推理 FPS、帧停滞次数和连续失败峰值；
- 检测类别、数量和置信度摘要；
- 每次失败的阶段、统一错误码、平台原始错误码和受限长度 stderr/runtime 信息；
- 停止时资源释放结果；
- 错误所属 run/worker generation/operation、设备恢复状态及有界相关内核日志。

默认不保存完整摄像头视频，避免报告空间失控和隐私风险。后续如需故障截图，必须单独配置数量、
分辨率、存储上限和隐私策略。

## 10. 非目标

第一版不包含：

- 对画质、清晰度、曝光、色彩、坏点或摄像头方向做自动判定；
- 要求两个模型实例真正并行推理；
- 在线训练、模型转换或在线下载；
- 将检测到目标作为默认通过条件；
- 根据模型 benchmark 分数评价算法精度；
- 使用 NPU 项替代 Factory Test 的人工 Camera 画面确认。

## 11. 验收场景

所有核心场景使用同一入口和判定契约，分别在 RZ/V2H、RZ/V2N、RK3576、T527、A733 的指定
BSP/runtime/model/Camera profile 组合上取证。下列 UI 条件后置，不阻塞当前核心逻辑验收。

至少覆盖：

1. 单摄连续运行，画面无目标但推理链路正常，应通过；
2. 双摄同时采集并提供预览帧，正常有帧时每个完整 round 均按顺序处理两路；
3. 任一路 Camera 停止出帧，准确报告对应 Camera 的 frame timeout；
4. 模型与 runtime 世代不匹配，启动阶段报告 model incompatible；
5. 平台 runtime 返回错误、任一阶段卡住或只发心跳不推进，能够有界判定；记录清理/设备恢复是否成功；
6. 开启固定目标物条件时，能够分别验证检测成功和检测不足；
7. 与 CPU、DDR、GPU、VPU、Storage、Ethernet 并发长稳运行，GUI 仍可响应；
8. 正常 Stop、全局到时和 critical thermal 停止后不残留资源；异常无法退出时准确报告并禁止重叠 Retest；
9. Retest 能重新创建干净的 pipeline 和 backend，不复用失效 context；
10. 报告能区分 Camera、预处理、NPU、后处理和清理阶段错误；
11. 缺少其他厂商的库时公共程序仍可启动；当前平台必需库缺失或不匹配时明确失败；
12. 模型参考向量能检出重复归一化、重复 sigmoid、head 映射/量化错误；
13. 零工作量、跨窗口完成及 partial 窗口按统一口径处理；
14. 慢消费者与有界帧池耗尽不导致存储被覆盖、内存无限增长或其他 Camera 饥饿。

平台矩阵中任一必需项未完成，就不能宣称统一版本已经通过全平台验收。

