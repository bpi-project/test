# NPU 与多摄像头稳定性测试需求

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

## 2. 支持范围

### 2.1 第一阶段目标平台

| 平台 | 推理后端 | 平台模型产物 |
| --- | --- | --- |
| Renesas RZ/V2H、RZ/V2N | DRP-AI/RUHMI runtime | 平台编译模型目录 |
| Rockchip RK3576 | RKNN Runtime | `.rknn` |
| Allwinner T527 | VIPLite v1.13、NPU v2 | v2 `.nb` |
| Allwinner A733 | VIPLite v2.0、NPU v3 | v3 `.nb` |

目标镜像负责提供匹配 BSP 的 NPU 驱动和用户态 runtime。部署包提供 Stability Test 插件、
共享后处理、平台适配层、配置文件及经校验的平台模型，不允许运行时在线下载模型。

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

双摄测试采用“同时采集、同时预览、同轮取帧、单实例串行逐路推理”模式：

```text
Front Camera ─→ 最新帧槽 ─┐
                           ├→ Front 推理 → Rear 推理 → 下一轮
Rear Camera  ─→ 最新帧槽 ─┘
```

这不是一轮只测 Front、下一轮只测 Rear。每个完整检测轮都必须覆盖所有已启用 Camera：

```text
Cycle 1: front → rear
Cycle 2: front → rear
Cycle 3: front → rear
```

两路 Camera 分别维护采集帧数、预览帧数、提交推理帧数、成功/失败次数、连续失败次数、推理
时延、最后帧时间和最后检测结果。一路失败不能被另一路成功掩盖。

该设计与 Renesas `multi_camera_vehicle_detection` 示例的实际行为一致：示例只有一个推理线程，
在一个 Camera 循环内对每路输入依次调用同一个 runtime，而不是创建多个并行 NPU 实例。

## 5. 模型逻辑契约

第一版定义逻辑模型契约 `yolov5s-coco-640-raw3head-v1`：

```yaml
input:
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
```

三个逻辑输出分别对应：

- stride 8：`3 × 80 × 80 × 85`；
- stride 16：`3 × 40 × 40 × 85`；
- stride 32：`3 × 20 × 20 × 85`。

平台后端可以返回不同数据类型、维度顺序或量化 tensor，但必须提供足够的 shape、layout、scale
和 zero-point 信息，并在进入共享后处理前转换为统一逻辑输出。不得仅按数组下标假定三个输出
顺序，必须校验名称或形状。

平台允许使用针对量化友好性裁剪过的 YOLOv5 ONNX，只要逻辑输出仍满足该契约。不同平台模型
无需拥有相同文件格式或完全相同的图结构。

## 6. 通过与失败规则

### 6.1 默认通过依据

默认长稳测试不要求画面中持续存在可识别目标。每路 Camera 的每轮成功至少满足：

1. Camera 已成功打开且采集帧持续推进；
2. 获得有效、非空且时间戳更新的输入帧；
3. 预处理成功；
4. NPU 推理在单次超时内成功完成；
5. 输出数量、形状和数值有效；
6. 后处理成功完成；
7. 本轮有效推理次数、推理成功率和推理 FPS 达到配置门限；
8. 没有命中连续失败、帧停滞、runtime 崩溃或资源无法释放条件。

未检测到任何目标不是默认失败条件，因为实际现场可能暂时为空场景。

### 6.2 可选目标物校验

工厂提供固定目标物时，可以显式启用检测正确性要求：

```yaml
require_detection: true
required_classes: [person]
minimum_detection_count_per_cycle: 1
```

只有启用 `require_detection` 后，未检测到指定类别才参与 PASS/FAIL。检测类别、数量和最大置信度
无论是否参与判定都应写入报告。

### 6.3 失败类型

至少区分：

- `CAPABILITY_UNSUPPORTED`：runtime、模型格式或必需能力不支持；
- `MODEL_NOT_FOUND`：模型或标签资源缺失；
- `MODEL_INCOMPATIBLE`：模型与 NPU/runtime 世代不匹配；
- `MODEL_CONTRACT_MISMATCH`：输入输出数量、shape、layout 或类别数不符；
- `CAMERA_OPEN_FAILED`：摄像头无法打开；
- `CAMERA_FRAME_TIMEOUT`：指定时间内没有新帧；
- `PREPROCESS_FAILED`：颜色转换、缩放或 letterbox 失败；
- `INFERENCE_FAILED`：NPU runtime 返回错误；
- `INFERENCE_TIMEOUT`：单次推理超时；
- `INVALID_OUTPUT`：输出为空、包含非法数值或无法反量化；
- `POSTPROCESS_FAILED`：YOLO decode 或 NMS 失败；
- `PERFORMANCE_BELOW_THRESHOLD`：有效 FPS 或成功率低于门限；
- `DETECTION_CRITERIA_FAILED`：显式目标物条件未满足；
- `CLEANUP_FAILED`：Camera、runtime、DMA buffer 或 worker 无法有序释放。

单轮失败必须写入错误阶段、平台原始错误码、Camera ID 和本轮指标。是否在连续失败达到上限后
停止该测试项，沿用 Stability Test 通用策略；必测项任一失败锁定全局最终 `FAIL`，其余测试项
默认继续，除非命中全局安全停止规则。

## 7. 性能与资源约束

- NPU 推理不能阻塞 GUI 线程或 Camera capture 线程；
- 每路 Camera 使用有界 latest-frame 缓冲，默认只保留最新一帧或双缓冲；
- 推理落后时丢弃旧帧，不积累延迟和内存；
- GUI 刷新频率与 Camera/NPU 频率解耦，避免检测框和图像更新拖慢测试；
- 第一版不绘制 NPU 实时曲线，使用实时值、轮次统计和详细报告，避免重现高频 QML 更新干扰
  VPU 的问题；
- Camera 采集分辨率、预览尺寸和模型输入尺寸分别配置，不因 UI 卡片较小而降低传感器测试输入；
- 长时间运行不得每轮重复加载模型或重建 Camera pipeline，除非发生受控恢复；
- 停止、超温、Service 退出和全局超时必须停止提交新推理，然后按顺序释放 Camera、worker、
  backend 和 DMA 资源。

## 8. GUI 需求

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
- 模型 profile、逻辑契约版本、模型文件 SHA-256、量化类型和标签版本；
- Camera profile、pipeline、采集格式、分辨率和帧率；
- 调度策略和 Camera 顺序；
- 每路采集帧数、丢弃帧数、提交推理帧数、成功/失败次数；
- 每路平均、最小、最大及高分位推理时延；
- 每路实际推理 FPS、帧停滞次数和连续失败峰值；
- 检测类别、数量和置信度摘要；
- 每次失败的阶段、统一错误码、平台原始错误码和受限长度 stderr/runtime 信息；
- 停止时资源释放结果。

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

至少覆盖：

1. 单摄连续运行，画面无目标但推理链路正常，应通过；
2. 双摄同时预览，每个完整检测轮均按顺序覆盖两路；
3. 任一路 Camera 停止出帧，准确报告对应 Camera 的 frame timeout；
4. 模型与 runtime 世代不匹配，启动阶段报告 model incompatible；
5. 平台 runtime 返回错误或单次推理卡住，能够超时并受控清理；
6. 开启固定目标物条件时，能够分别验证检测成功和检测不足；
7. 与 CPU、DDR、GPU、VPU、Storage、Ethernet 并发长稳运行，GUI 仍可响应；
8. Stop test、全局到时和 critical thermal 停止后不残留 Camera/NPU worker；
9. Retest 能重新创建干净的 pipeline 和 backend，不复用失效 context；
10. 报告能区分 Camera、预处理、NPU、后处理和清理阶段错误。

