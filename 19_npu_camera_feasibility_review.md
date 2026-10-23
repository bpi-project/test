# NPU / Camera 稳定性测试可行性评审

评审日期：2026-10-23。输入为 `17_npu_stress_requirements.md` 与 `18_npu_stress_detailed_design.md`；首要评估一套 NPU/Camera 子项同时兼容 RZ/V2H、RZ/V2N、RK3576、T527、A733。Renesas 是当前优先验证环境，不改变全部目标平台的兼容要求。

后续源码、模型及运行库核查见 [20_npu_platform_compatibility.md](20_npu_platform_compatibility.md)。其中固定了参考版本，补充 T527/A733 共用 VIP API 的依据、模型输入 binding 与激活语义、native timeout，以及对原设计的具体修订建议。

**落实状态：** 本文核心结论已合入 17、18 两份文档；下文保留评审时的问题与建议用于追溯。
当前实现约定以修订后的需求和详细设计为准，不代表代码或五平台真机验收已经完成。

**结论：通过统一执行引擎和平台适配层实现同一套多平台 NPU/Camera 子项，技术路线可行；目前资料尚不足以证明所有目标板都已满足该契约。** 应先收口全部目标平台的兼容边界，再实现持续采集与检测逻辑，不能形成 Renesas 专用版和其他平台版。正式功能是 Camera → 预处理 → NPU → 后处理持续运行，并提供预览帧、检测结果和测试指标；其稳定性按约定工作量及错误条件判定，不要求首版证明 NPU 始终满载。

**用户确认的边界：** 最终行为类似 Renesas `rzv_ai_sdk` 的 `object_detection` 应用，作为 Qt 压力测试主程序的一个 NPU/Camera 子项，后续嵌入主界面卡片。当前先实现程序逻辑，UI 布局、绘制和预览传输优化后置；保留帧、检测结果、状态和指标接口即可。本文提出的 Camera-only / NPU-only 模式属于可选内部诊断能力，不是新增产品子项，也不是首版交付前置条件。

**最高优先级约束：** 与主稳定性测试程序保持统一产品版本、统一源码主线、统一子项 ID、统一行为和报告协议。目标平台中的某个 backend 尚未完成，属于兼容性工作未完成；不能通过另发一个产品版本、关闭该平台的 NPU 子项、静默使用纯 CPU 推理或跳过必测项来满足验收。

**已确认的配置原则：** 独立硬件和部署相关参数统一放在板级 YAML，包括 Camera pipeline、设备节点、backend 选择、厂商库路径及运行环境、模型和硬件运行参数。平台切换由配置选择适配实现及资源，公共执行逻辑保持相同。原设计已经包含板级 pipeline 配置，本评审将其视为既定机制；pipeline 字符串不同本身不是兼容性缺陷。

本次是文档与公开实现评审，没有连接目标板，也未获得当前 BSP、模型文件或 Stability Test 主工程。因此，以下平台支持结论与设计风险有文档/源码依据；具体 FPS、超时、内存上限、双摄链路及故障恢复能力仍需目标板验证。

**一、建议保留的设计**

- Service 负责判定，独立 worker 承载 vendor runtime，GUI 只消费预览及状态。
- 第一版单模型 context、双摄公平轮转推理；没有必要为了“双摄”先引入多 context。
- Camera、推理和预览频率解耦，使用有界最新帧缓冲。
- Run 内复用模型和 Camera，统计窗口不重建资源；启动/停止可靠性另设测试。
- 空场景不因“没检测到目标”失败；量化差异不要求跨平台二进制相同。
- 板级 profile 承载设备、模型和 runtime 差异；先有界复制，再按实测瓶颈优化。

**多平台兼容必须先收口的边界**

原文已有 `INpuBackend` 和板级 YAML，应继续沿用。当前需评估的是配置选择的各个 backend 能否满足共同的调用与数据语义，以及是否存在尚未进入配置的硬件常量。下表是建议的架构约束，不代表当前仓库已有这些实现。

| 边界 | 所有平台共享的部分 | 允许封装的平台差异 |
| --- | --- | --- |
| 子项控制 | 同一个 `npu_inference`、状态机、启动/停止/Retest、配置 Schema | board profile 引用不同适配模块和资源 |
| Camera | `ICameraSource` 的打开、取新帧、关闭；统一 Frame 描述、所有权、时间戳和统计 | V4L2/GStreamer pipeline、media graph、传感器配置、像素格式及驱动内存 |
| 预处理 | RGB、缩放/letterbox、归一化等完整数值契约及 CPU 参考实现 | 经一致性验证的 DRP/RGA/G2D 等硬件路径；具体能力由目标 BSP 验证 |
| NPU | `INpuBackend` 的 probe/load/describe/run/release/close 语义、错误映射 | DRP-AI、RKNN、VIPLite API、buffer 分配、cache 同步和 context 管理 |
| 运行库装载 | 公共装载流程、启动校验和错误报告 | YAML 指定 backend 模块、厂商库路径、依赖搜索目录、运行环境及预期 ABI/runtime 版本 |
| 模型适配 | 同一 YOLOv5 解码/NMS 实现、同一 Detection 结构与类别语义 | native layout、dtype、量化、激活边界由模型 manifest 和 tensor adapter 描述 |
| 故障处理 | Service 统一的操作 deadline、失败锁定、进程监督和报告 | runtime 原始码、取消/复位能力、设备健康探测 |
| 判定与指标 | 相同计算口径、错误分类和非零工作量要求 | 已验证的每板 FPS/timeout 门限、可选利用率/频率采样接口 |

Camera 适配和 NPU 适配独立组合，业务引擎不依赖 `/dev/video*` 编号、vendor tensor 类型或某一 SDK 的 DMA 类型。若各 BSP 的 GStreamer 均能满足采集要求，优先共用 GStreamer 采集实现并使用板级 pipeline；确需直接 V4L2 或 vendor 路径时，也只替换 `ICameraSource` 的实现。

对共同使用 GStreamer 的平台，pipeline 差异直接由 YAML 解决，无需为每个 SoC 复制一个 Camera 实现。共同取帧出口需要约定 appsink 名称、实际协商格式及 Frame 描述；公共层应验证配置对应的实际输出，而不是仅相信配置声明。

YAML 决定“选哪个 backend、加载哪些库、使用哪些模型和参数”；backend 实现“如何调用该厂商 API”。库路径可配置，不要求各厂商导出相同函数或使用相同 tensor/buffer 类型。适配层把这些调用差异转换为公共接口，构成同一程序的一部分。

建议在现有板级配置中补齐以下字段类别，最终键名沿用主程序现有 Schema，不另建一套配置体系：

| YAML 配置类别 | 内容 | 程序启动时验证 |
| --- | --- | --- |
| Camera | pipeline、sink 名称、采集参数、buffer 上限、必要的板级初始化 profile | pipeline 可建立，协商格式有效，必需 Camera 均出首帧 |
| Backend | 类型、适配模块路径、适配接口版本 | 模块可加载，接口版本和必需操作齐全 |
| Vendor runtime | 按用途标识的库路径、依赖搜索目录、进程环境、预期 runtime/driver 版本 | 指定库及依赖可解析，必需符号和版本匹配；路径配置实际生效 |
| 模型 | 模型/标签/预处理资源路径、hash、模型语义 manifest 或其引用 | 实际 tensor 与声明相符，数值语义可接入共享后处理 |
| 硬件执行参数 | profile 中允许的 core mask、频率、设备选择、内存/预处理方式 | 所选 backend 支持对应参数，配置被应用；不静默忽略 |

运行库路径需区分适配模块与 vendor runtime，装载器必须实际使用 YAML 指定值。若使用进程环境设置动态链接器搜索路径，应在启动 worker 前设置，不能把“已经加载后再改环境变量”当作重新选择库。不要要求在源码中更改库名、路径或平台宏来完成部署切换。模块内使用的 vendor API 调用约定属于适配代码，硬件参数和资源位置属于 YAML。

运行报告保存实际生效的配置快照及可获取的库路径、版本、模型 hash 和协商 Camera 参数。所配置的版本与实际加载结果冲突时在启动阶段报错，避免配置表面一致、实际运行环境不同。

模型 manifest 和 tensor adapter 的职责是把平台表示接入同一数值语义，不能把三个 vendor 示例各自的完整后处理保留下来成为三套行为。含 sigmoid 与不含 sigmoid 的 head 均可通过显式语义适配进入共享解码；量化支持必须按实际类型描述，不能默认所有量化都只有一个 scale/zero-point。模型资源存在差异，不改变上层算法及测试流程。

建议交付结构：

```text
同一稳定性测试产品版本
  ├─ 同一 npu_inference 插件 / Service 控制逻辑
  ├─ 同一 stability-test-npu-worker 执行引擎
  ├─ 平台 backend 模块：Renesas / RKNN / VIPLite 对应 ABI
  └─ 板级资源：profile + runtime 依赖说明 + model manifest + 模型
```

worker 按受信任板级 YAML 指定的 backend、模块和厂商库按需装载，主程序和 worker 公共入口不直接强链接全部 vendor 库。这样不会因为某板没有其他厂商的 SDK 库而连主程序都无法启动。加载模块失败要转为明确的依赖/兼容错误，不能静默忽略；支持矩阵中的目标平台必须具备并通过对应模块。Linux `dlopen` 提供按需装载共享库的机制，但不自动解决 SDK ABI 兼容。[动态装载接口](https://man7.org/linux/man-pages/man3/dlopen.3.html)

公共引擎内部可使用 C++ 接口。若 backend 作为独立 `.so` 构建，推荐用带 `abi_version`、结构大小和明确内存归属的窄 C ABI 跨模块，不跨不同工具链传递 STL、异常或 vendor 对象。一个 worker 只装载所选平台 runtime。本次进一步核查的 Allwinner Model Zoo 为 T527/A733 提供公共 VIP 头文件与封装，两种 API 库均导出所需 `vip_*` 符号，优先共用 backend 并以 YAML 选择库。实际 BSP 如存在 ABI 差异，再增加局部适配或模块构建目标，仍随同一产品版本发布，不能仅因库名不同就预设必须拆模块。

“统一产品实现”和“一个 ELF 文件跨所有 BSP 原样运行”需要区分：前者可以由以上结构保证；后者还取决于 CPU ABI、glibc/libstdc++ 和系统依赖，须沿用主程序现有构建发布规则核实。如果 BSP 要求不同工具链，则从同一提交构建对应产物，不产生独立业务源码、独立功能版本或按平台维护的程序分支。

所有目标平台的**必需能力**统一为：真实 Camera 连续采集、所要求的单/双摄调度、真实 NPU 执行、目标检测、结构化结果/指标、正常停止清理、异常有界判定。硬件预处理、零拷贝、NPU 利用率、多 context 属于可选能力；可选能力不同不能导致基本功能消失。CPU 预处理/后处理可作为共同路径，但不能用整个模型的纯 CPU 推理冒充 NPU 测试。

兼容性结论至少需要五列矩阵逐项取证：RZ/V2H、RZ/V2N、RK3576、T527、A733。每列记录指定 BSP/runtime/模型组合及 Camera profile；每行覆盖加载、tensor 语义、单/双摄、超时、清理、Retest、并发及报告。当前均未在本次评审中完成真机验收，不能用某个平台的 demo 或相同芯片架构替代其他列的验证。

**二、平台核查与落地判断**

| 平台 | 已核查依据 | 对实现的影响 |
| --- | --- | --- |
| RZ/V2H | 官方模型表验证 YOLOv5s 640×640；多摄应用使用一个推理线程，逐路调用同一 runtime | 作为当前接口验证环境；参考代码只进入适配层，公共生命周期与判定保持统一 |
| RZ/V2N | 官方验证同一模型，但区分 Balanced System 与 AI-Centric 总线模式 | 不能直接沿用 V2H 的性能门限；记录总线配置、频率、BSP 和预留内存 |
| RK3576 | RKNN Model Zoo 明确支持 YOLOv5；原生 API 有状态码、tensor 描述及 timeout 字段 | 接口适配路径清楚，但必须核实模型的 sigmoid 边界、core mask 和目标 BSP 的实际 timeout 行为 |
| T527 / A733 | 旧 SDK 及更新的 Model Zoo 均有参考路径；所查 Model Zoo 明确支持两个导出 target，库分别为 VIPlite/VIPuser 与 NBGlinker/VIPhal | 共用 VIP backend 的路径已有源码和符号依据；分别配置并验证模型/运行库组合，业务执行引擎保持相同 |

依据：[RZ/V2H 模型表](https://github.com/renesas-rz/rzv_drp-ai_tvm/blob/main/docs/model_list/Model_List_V2H.md)、[RZ/V2N 模型表](https://github.com/renesas-rz/rzv_drp-ai_tvm/blob/main/docs/model_list/Model_List_V2N.md)、[Renesas 多摄源码](https://github.com/Ignitarium-Renesas/rzv_ai_apps/blob/main/14_Multi_camera_vehicle_detection/src/multi_camera_vehicle_detection.cpp)、[RKNN YOLOv5](https://github.com/airockchip/rknn_model_zoo/blob/main/examples/yolov5/README.md)、[RKNN API](https://github.com/airockchip/rknn_model_zoo/blob/main/3rdparty/rknpu2/include/rknn_api.h)、[Radxa 平台版本表](https://docs.radxa.com/en/cubie/a7s/app-dev/npu-dev/cubie-acuity-usage)。

Renesas 有三个需要落实到设计的细节：

1. 当前 `multi_camera_vehicle_detection` 使用 **YOLOX-l**，可以证明多摄串行调度有参考实现，不能证明其模型、预处理与后处理满足本项目的 YOLOv5s 契约。当前 README 也区分 V2H/V2N 的 MIPI 路数及具体相机方案；仍要用实际传感器、设备树和 media graph 验证双摄。[多摄 README](https://github.com/Ignitarium-Renesas/rzv_ai_apps/blob/main/14_Multi_camera_vehicle_detection/README.md)
2. V2N 官方表在其指定配置下给出 YOLOv5s Balanced 43.0 ms、AI-Centric 20.7 ms；表中还注明扩展 DRP-AI 内存区域。这些数字只能说明配置对性能影响很大，不能作为当前 SBC 的验收门限。并发 Camera/VPU/DDR 压测应首先使用产品实际配置。[V2N 测量条件](https://github.com/renesas-rz/rzv_drp-ai_tvm/blob/main/docs/model_list/Model_List_V2N.md)
3. Renesas YOLOv5 导出说明提供截取三个卷积输出的路径，支持统一 raw-head 的方向；需要固定权重、导出版本与节点映射，而不是仅检查文件名包含 `yolov5s`。[官方转换说明](https://github.com/renesas-rz/rzv_drp-ai_tvm/blob/main/docs/model_list/how_to_convert/How_to_convert_yolov5_onnx_models.md)

关于设计第 6.3 节的 A733 待确认项：公开资料已提供可进一步验证的路径。Radxa 指定 A733 使用 `source env.sh v3`；其链接的 SDK 脚本将 v3 映射为 `VIP9000NANODI_PLUS_PID0X1000003B`。可以把“尚无明确 target”收敛为“核对当前项目 SDK 是否采用该 target，并验证生成模型”；不能据此认定所有历史 BSP 均兼容。[Radxa 编译流程](https://docs.radxa.com/en/cubie/a7s/app-dev/npu-dev/cubie-acuity-usage)、[SDK target 配置](https://github.com/ZIFENG278/ai-sdk/blob/main/scripts/pegasus_setup.sh)

进一步核查的 Model Zoo 转换脚本已显式接受 `a733` 和 `t527`，无需继续把“A733 没有明确 target”作为公开资料层面的阻塞项。实际下载包的预处理和输出布局也与旧 `ai-sdk` 不同，应由模型 profile 描述；下载镜像与网页标注版本的差异及源码证据详见文档 20。[Model Zoo YOLOv5 用法](https://docs.radxa.com/en/cubie/a7a/app-dev/npu-dev/model-zoo/yolov5)

**三、实现前应修正的问题**

**P0-1：模型契约缺少数值语义，现有共享 sigmoid 会与部分平台模型冲突。**

位置：需求第 5 节；设计第 6.2、7.1、7.3 节，尤其 `18_npu_stress_detailed_design.md:244`。

三个 head 的尺寸相同，不代表都输出 logits。Rockchip 官方 FAQ 明确指出，其 YOLO demo 的类别置信度预期已经经过 sigmoid，源码也直接使用这些数值计算分数。若再统一做一次 sigmoid，会改变置信度和检测结果；程序仍可能每帧正常返回，因此单靠“无错误”难以发现。[官方 FAQ](https://github.com/airockchip/rknn_model_zoo/blob/main/FAQ.md)、[YOLOv5 后处理源码](https://github.com/airockchip/rknn_model_zoo/blob/main/examples/yolov5/cpp/postprocess.cc)

建议为每个模型包提供 manifest，至少固定：

- 输入颜色、layout、dtype、数值范围、mean/scale，以及归一化由应用还是模型/runtime 完成。
- resize 插值、letterbox 填充值、padding/舍入规则、是否允许放大；输入 YUV 的色彩矩阵和范围。
- 每个输出的名称或显式映射、shape、stride/padding、dtype、量化方式及量化轴。
- xywh、objectness、class 各字段是否已激活；anchor 数值及排列、decode 版本、类别顺序。
- 置信度计算公式、NMS 是否按类别、候选数及最终框数上限。
- compiler/runtime 兼容范围、源模型及导出配置、模型目录逐文件 hash、测试向量和容差。

共享后处理应作为统一实现的要求，根据 manifest 选择明确的语义分支，确保 sigmoid 只执行一次。不要凭输出是否落在 [0,1] 来猜测，也不要用反 sigmoid 恢复量化 logits。

另外，统一逻辑输出不必强制每帧生成一个完整 FP32 拷贝。按本文契约计算，三个 head 共 2,142,000 个元素，FP32 占 8.17 MiB；10 次/秒仅写入这份缓冲就有约 85.68 MB/s 数据量，尚未计入读、重排和后处理。先使用可复用缓冲验证正确性，实测需要时再采用 tensor view、按候选反量化等优化。

**补充建议：明确端到端负载的测量口径，独立诊断模式可后置。**

位置：需求第 3 节；设计第 4.2 节及 `target_total_inference_fps`。

串行推理可行，但当前一条执行链包含预处理、runtime、输出转换及 NMS，任一 CPU 阶段变慢都会让 NPU 等待。若目标 FPS 是限速值，也会主动降低负载。例如假设单次 runtime 调用 20 ms、总计每秒 10 次，runtime 调用时间仅覆盖约 20% 墙钟时间；这是示意计算，并非硬件利用率。

第一版优先完成 `camera_npu_e2e`。后续如果需要区分采集与推理故障，或验证最大 NPU 负载，可在同一插件/worker 中增加互补的内部诊断模式：

| 模式 | 数据路径 | 主要回答的问题 |
| --- | --- | --- |
| `camera_only` | 实际单/双摄持续采集，预览可开关 | 摄像头、CSI/USB、ISP 与采集驱动是否稳定 |
| `npu_soak` | 内存中的已验证 tensor 集持续推理，抽样做参考输出校验 | runtime/NPU 能否持续执行，避免被实时采集及每帧绘框限速 |
| `camera_npu_e2e` | 当前双摄 → 预处理 → 推理 → 后处理 → 预览 | 完整业务链路与资源竞争是否稳定 |

Camera 实时检测是正式执行路径。可选 NPU 模式只增加诊断覆盖，其结果不作为 Camera 测试结果；本轮不要求实现独立输入模式切换或组合压测调度。

第一版先明确限速 FPS 与最低验收 FPS 的区别，分开记录预处理、runtime、输出转换、后处理与端到端时延。没有真实硬件计数器时只报告可测量指标，不把 runtime 时间占比叫作 NPU utilization。若以后需要最大吞吐模式，再增加 `load_mode: max_throughput|fixed_rate`。

Allwinner 可先借 `vpm_run` 的循环、timeout 和关闭结果文件输出选项验证平台基线，再接入统一 worker。正式结果仍需结构化进度和输出校验，不能仅靠解析命令行平均 FPS。[Radxa vpm_run](https://docs.radxa.com/en/cubie/a7s/app-dev/npu-dev/cubie-vpm-run)

**P0-2：超时检测、终止进程、设备恢复需要三个独立结论。**

位置：设计第 5、11 节；需求验收场景 5、8、9。

文档已有独立进程与 watchdog，方向正确，但 `run(..., timeout)` 并不能自动让 vendor API 可取消。Renesas 的 `MeraDrpRuntimeWrapper::Run()` 也是 `void`，文档只特别指出 Allwinner 的 `awnn_run()`，应补上 Renesas 的适配策略。[Renesas Runtime Wrapper](https://github.com/renesas-rz/rzv_drp-ai_tvm/blob/main/docs/Runtime_Wrap.md)

补充核查：RKNN 有 `rknn_run_extend.timeout_ms`，VIPLite 有 `VIP_NETWORK_PROP_SET_TIME_OUT` 与取消接口，宜与外部 deadline 配合。新 Allwinner Model Zoo 的 `network_run()` 已返回 int，但会合并部分错误且 cache flush 失败仅打印；正式 backend 仍需完整传播错误。不能将旧 `awnn_run()` 的局限泛化为所有新 SDK 均无状态返回，具体接口及依据见文档 20。

建议：

- Service 按 `run_id / worker_generation / operation_id / stage / start_time / deadline` 独立监督；worker 的 IPC 心跳存活不能替代推理进度。
- 超时覆盖模型加载、等待首帧、预处理、推理、取输出、后处理和清理，不能只盯 `Run()`。
- 上层 `Result` 接口继续保留。底层 `void` 调用的可捕获异常、进程异常退出、超时、无效输出分别映射；没有 vendor 原始码时明确“不可获得”，不包装成伪造的成功码。
- 优先使用已验证的 vendor timeout/cancel；其次请求停止、等待宽限期，再由 Service 终止整个 worker。禁止在线程外释放仍被执行中的 context/buffer。
- 杀进程不等于复位硬件。内核存在不可中断等待，用户态监督不能保证所有驱动卡死均立即退出；需要 `DEVICE_UNHEALTHY / RESET_REQUIRED` 等状态，禁止在未恢复的设备上无限 Retest。[Linux 等待机制](https://docs.kernel.org/driver-api/basics.html#wait-queues-and-wake-events)
- 正常停止要求资源释放；故障路径保证 Service 有界判定并保留证据，设备能否恢复按 BSP 能力验证。板级 reset/reboot 是显式恢复策略，不能把原失败改为 PASS。
- buffer 销毁顺序按 runtime 所有权确定。设计第 11 节固定“先释放全部 DMA，再销毁 context”不能适用于所有外部绑定内存的后端。

RKNN 头文件存在 `timeout_ms` 和 `RKNN_ERR_TIMEOUT`，可优先适配；字段存在仍不等于当前板上所有阻塞阶段都有该超时保护。[RKNN API](https://github.com/airockchip/rknn_model_zoo/blob/main/3rdparty/rknpu2/include/rknn_api.h)

**后续 UI 集成注意：预览双槽的原子切换没有保护读者，可能读到被覆盖的像素。**

位置：`18_npu_stress_detailed_design.md:379`。该项在预览集成时解决，当前不要求先实现共享内存传输；程序内部 capture 与 inference 的帧所有权仍需先保证正确。

可构造的时序：GUI 开始读取 A；worker 写 B 并切换；GUI 因调度变慢还没读完 A；worker 下一帧开始写 A。此时每次 active index 切换都是原子的，GUI 仍可能获得撕裂帧。加成三槽只会降低概率，不能独立解决问题。

建议固定容量预览池配套明确的读者 lease/释放协议：GUI 通过控制 IPC 取得已提交槽的使用权，复制到自身纹理或私有缓冲后释放；writer 仅写没有读者的槽，池满时丢预览帧。像素仍只读映射，不经过 Service 转发。

协议要定义 generation、读写同步、GUI 卡住/退出和重连处理；不能只因 lease 超时就覆盖一个仍可能被活跃 GUI 读取的槽。若选择 sequence 校验后重读的方案，还需证明共享内存访问符合目标实现的同步/内存模型，不能只增加一个整数序号。

**P1-1：latest-frame 之外，GStreamer 和驱动缓冲也必须有界。**

位置：设计第 4.1、8.1、10 节。

应用只保留一帧，并不保证 appsink、上游 queue 或 vendor 插件不积压。GStreamer 官方说明 appsink 内部有队列，`max-buffers=0` 表示无限；新版本的 `leaky-type` 从 1.28 才提供。[appsink 官方文档](https://gstreamer.freedesktop.org/documentation/app/appsink.html)

建议板级 pipeline 显式设置 caps、队列容量和丢旧帧策略；旧 BSP 按其版本使用 `drop` 等实际支持的属性，不照搬新版本字段。采用有超时的取帧与 bus ERROR/EOS 处理；推理线程无新帧时等待事件/短超时，避免空转。

另需规定 capture 帧的所有权：长期持有 GstBuffer/V4L2 buffer 可能耗尽有限采集池。第一版优先用小容量自有缓冲池及时归还驱动 buffer，再评估复制开销。整个采集路径不能因预览缩放、GUI IPC 或推理而阻塞。

headless worker 的采集 profile 建议按 sensor、格式、分辨率和 BSP 组织；设计中用 `x11/wayland` 选择采集 pipeline 的必要性应核实。

**P1-2：Camera 健康与主动少做推理必须分开统计。**

位置：需求第 6.1、9 节；设计第 4 节。

应用接到帧后自行递增 sequence，只能证明回调发生；上游重复帧、旧帧积压或驱动错误不一定暴露。应保留能够取得的驱动 sequence、PTS、错误标志、实际协商格式/帧率，并增加每路 capture FPS、最大到帧间隔及帧龄。取得不到的信息标为 unsupported。V4L2 定义了 sequence、timestamp 与 error 标志，可作为适用平台的观测来源。[V4L2 buffer 文档](https://docs.kernel.org/userspace-api/media/v4l/buffer.html)

至少区分源端丢帧/错误帧、pipeline 丢帧、latest-frame 覆盖、推理主动跳帧和预览跳帧。30 fps 采集、5 fps 推理产生的大量未推理帧是设计行为，不能直接判为 Camera 故障。像素不变也不能直接判定停滞，因为真实场景可以静止。

帧 timeout 应由独立监控判断，不能依赖串行推理线程轮到该 Camera。报告同时使用单调时钟计算间隔，并标明 PTS 的时钟域；不能直接拿媒体时间戳减系统墙钟。

**P1-3：round、统计窗口和判定口径存在歧义。**

位置：需求第 4 节、设计第 3.2、4.2、8.2 节。

建议 `round` 表示轮询各路一次，`window` 表示例如 300 秒的统计区间，`run` 表示完整生命周期。“同轮取帧”当前不等于同时曝光：代码是处理每路前取该路最新帧。对本稳定性测试通常足够，不必引入同步相机的复杂度；若需要共同快照则另行明确时间差。

使用 Run 级单调累计计数和窗口快照差值，避免多线程窗口边界清零漏计。明确按完成时间归属窗口、跨窗口 in-flight 数、启动预热及提前停止规则；成功率分母定义为已完成的成功/失败操作，超时操作结算为失败，未完成项单列。不能把跨窗口提交数直接当成本窗口完成数。

每路必须有最小有效工作量与进度 deadline：`0/0` 成功率不能成为 PASS；两个 minimum 字段为 null 时仍需强制非零有效完成量。分别统计 Camera、预处理、runtime 和整链路成功率，防止“从未进入 NPU 的失败帧”被排除后仍显示 100%。

补全 `target_total_inference_fps` 是限速还是验收目标、暖机期、窗口间 interval 是否持续施压，以及连续失败达到阈值时是 `>=` 还是 `>`。静态自检关闭时不校验空图片路径，解决设计第 8.3 节的无条件文件校验与可选自检之间的冲突。

**P1-4：有限输出并不能证明计算正确，应引入离线参考向量。**

启动自检目前可选、现场也允许无目标；若模型输出始终为零或沿用上一帧结果，shape/NaN 检查可能都通过。

研发阶段保留可选接口合理；正式交付的模型包建议附带少量固定图片/输入 tensor 和每平台参考结果，运行前做已知答案检查，长稳中低频轮换校验并单独计数。不要求跨平台逐字节相同，以量化容差、指定框/类别等定义通过标准。tensor 向量验证 runtime，图片向量同时验证预处理；两者覆盖不同。

**P1-5：资源与全板故障证据需成为实际观测项。**

需求提到内存泄漏，但报告没有完整观测方案。建议记录 RSS/PSS、FD/线程数，以及平台能提供的 DMA/CMA/NPU 内存统计；在暖机后评估增长趋势与停止后的回收情况，不能只比较起止 RSS。资源缺少可读接口时明确标记。

失败时保存有界内核日志、OOM/driver reset/timeout、温度和可获取的频率/降频信息。所有压力项共用 Run ID 和时间基准。若把整板失联/重启也纳入验收，应由板外主机或串口保存心跳和最后日志；板内报告无法保证在系统挂死时成功落盘。

**四、建议的实现结构与取舍**

沿用两份文档中的 Service 插件模型，以全部目标平台的公共契约实现同一套单/双摄连续检测逻辑：

```text
Service：生命周期、资源占用、独立 deadline、失败锁定、报告
  └─ 同一 worker 执行引擎（按板级配置装载 backend）
       ├─ CameraSource × 1/2 → 有界最新帧池
       ├─ 公平轮转 → 预处理 → 单个 NPU context
       ├─ 输出校验/语义适配 → YOLO 解码/NMS
       ├─ ResultSink → 帧引用、Camera ID、序号、时间、检测结果
       └─ 结构化进度、每路指标及错误
```

这里是职责划分，不要求新增三个独立进程或三个插件。第一版 Camera 与 NPU 同处一个 worker 可接受；runtime 崩溃会一起结束双摄，应据实记录。只有后续确有“推理崩溃后 Camera 仍要继续”的需求，再拆 Camera 进程。

当前核心逻辑的最小闭环是：probe/load → 启动各路 Camera 并等待首帧 → 持续公平轮转检测 → 按窗口汇总判定 → stop/cleanup。模型及 Camera 在 Run 内复用；一路失流单独计错，无新帧时不反复推理旧帧；超时由 Service 独立监督。结果接口规定 frame handle 的有效期与持有上限，慢消费者不能阻塞采集和推理。开发时可用日志/结果文件消费结果接口，Qt 后续接入同一接口。

`ICameraSource`、`INpuBackend` 和模型语义适配从一开始就是明确边界；调度、状态机、后处理、指标和报告只维护一套。先对照全部 SDK 确认每个接口均有落地方式，再在 Renesas 上实现和验证公共引擎，并尽早以其他平台的最小适配验证接口。不要先写 Renesas 专用流程再事后抽取，也无需扩展到首版未要求的任意模型、异步队列和 DMA-BUF 框架。

Renesas wrapper 的 `GetOutput()` 主要返回 dtype、指针及元素数，且不同 runtime 分支可提供的元数据不同。因此 `validateContract()` 要定义真实信息来源：扩展 wrapper/查询图元数据，结合已校验 manifest；不能只有元素总数却声称完整校验了 shape/layout。对模型目录进行整体可复现的文件清单校验。[Wrapper 实现](https://github.com/renesas-rz/rzv_drp-ai_tvm/blob/main/apps/MeraDrpRuntimeWrapper.cpp)

共享后处理可以先采用 CPU 参考实现；Renesas 硬件 PreRuntime 的 letterbox/归一化语义逐项比对后再替换。需要测量证据时再增加预处理流水线、量化域筛选或 DMA-BUF。普通 worker 不需要显示环境，但某些硬件转换插件的实际依赖仍由 BSP 验证。

GUI 需要区分“最近检测结果”和“当前帧检测结果”，按采集时间计算结果年龄；过期清除只解决残留框，不能消除新画面叠旧框的时间错位。预览是否真的显示过，只能由 GUI 反馈；worker 发布帧数不可直接命名为显示帧数。

**五、建议的实施顺序与阶段出口**

1. **先核查板级 YAML 与多平台兼容矩阵。** 对全部目标板记录 BSP/kernel、runtime/driver、工具链、模型格式、Camera profile 和依赖包；将 pipeline、库路径、模型及硬件参数明确映射到现有配置 Schema。Renesas 额外记录 AI SDK/RUHMI、DRP-AI 预留内存、总线模式与频率；Allwinner 明确 v1.13/v2.0 的 ABI 和模型边界。统一产品版本与 CI 构建入口。
2. **定义并核验公共契约。** 完成 Frame、Tensor、Detection、Result、capability、模块 ABI 和模型 manifest；逐平台列出信息来源、操作映射、所有权及故障处理方式。验证 YAML 选择到的 pipeline/runtime/model 组合满足同一接口。接口缺口在实现公共流程前暴露，不以未来移植替代当前兼容评估。
3. **做各后端最小验证。** 从同一测试入口验证 load/run/output、参考结果、异常传播及释放；当前先上 Renesas，但其他 backend 的编译与契约验证尽早进行。确认真实 NPU 子图执行，识别 CPU 回退，不把 runtime 名字视为硬件执行证据。[Renesas profiling 说明](https://github.com/renesas-rz/rzv_drp-ai_tvm/blob/main/docs/Runtime_Wrap.md)
4. **实现共享连续检测引擎。** 单/双摄采集、latest-frame、公平轮转、预处理语义、共享 decode/NMS、窗口判定、报告只实现一次；各平台只接入采集/NPU 适配和板级资源。先以日志消费结果，不依赖 Qt 卡片完成。
5. **使用同一测试集验证故障路径。** mock 注入阻塞、崩溃、慢阶段、无效输出和持续不推进；覆盖帧所有权、慢消费者、窗口边界、Stop/Retest。各板用支持的方式制造失流；USB 热拔测试不能照搬到 MIPI。GUI 验证后置。
6. **在每个目标板逐步加压。** 持续检测、并发其他压力项、启动/停止耐久逐项取证，记录总线/温控/调度变化。门限值允许按板基线配置，计算口径与失败语义必须一致；临时诊断工具不成为分叉的正式实现。
7. **以全平台矩阵作为统一版本出口。** 同一产品版本在五个目标平台均完成必需能力验收。任何一列未验证就记录为未完成，不宣称全平台兼容，不通过另发专用版绕开。厂商 SDK 依赖仅由对应模块引入，并验证缺少其他厂商库时公共程序仍可启动。

第一版程序逻辑验收至少应证明：空场景正常通过；每路实际完成推理；单路失流准确定位；各阶段工作量可观测且达到配置门限；超时由 Service 判定；无进度不能通过；慢结果消费者不破坏帧数据或拖住执行；故障后不复用失效 context；报告能复现板端配置并关联内核错误。Qt 卡片显示、共享预览与重连在后续集成阶段验收。

**六、两份原文的建议修改位置**

| 原文位置 | 建议修改 |
| --- | --- |
| 需求 §1、§2；设计 §2、§5、§6、§8、§12 | 明确同一产品/引擎及全平台验收；沿用板级 YAML，补库装载/版本/模型语义字段与实际生效校验；核查 Camera/tensor/模块 ABI 边界 |
| 需求 §3、§7、§11 | 明确 Camera 连续检测为首版主路径，区分工作量与性能门限；独立 Camera/NPU 模式仅作为可选诊断建议 |
| 需求 §5；设计 §6.2、§7 | 补模型 manifest、sigmoid/anchor/归一化语义及参考向量 |
| 需求 §6、§9；设计 §3、§8 | 明确工作量、分母、窗口归属、帧龄、丢帧分类及可选自检规则 |
| 设计 §5、§6.1、§11 | 补 Renesas void API 适配、阶段 deadline、进程终止与设备恢复边界、buffer 所有权 |
| 设计 §8.1、§9、§10 | 当前先补 pipeline 背压和内部帧所有权；预览读者协议与 GUI 重连后续集成时落实 |
| 设计 §6.3、§12、§13、§14 | 更新 A733 可验证 target、Renesas 示例模型类型、V2N 总线模式；Renesas 先验证，全部平台使用同一契约且逐项验收 |

首要评估是“所有目标平台能否用同一执行引擎完成同一子项”，其次验证持续工作、检测语义、故障定位与停止。兼容边界和各板验证收口后，现有方向可以支撑统一首版；四摄、多 context、全链路零拷贝和 UI 优化均可后置。
