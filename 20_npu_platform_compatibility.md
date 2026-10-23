# NPU / Camera 多平台兼容：示例源码与模型核查

核查日期：2026-10-23。本文补充 [总体评审](19_npu_camera_feasibility_review.md)，依据实际示例、转换配置、API 头文件及部分模型/ELF 元数据，收口 `17`、`18` 两份文档的实现边界。当前目标仍是一个 Qt 稳定性测试产品中的同一个子项，先完成程序逻辑。

本次核查结论已合入 [需求](17_npu_stress_requirements.md) 和 [详细设计](18_npu_stress_detailed_design.md)。本文保留来源与检查证据，具体接口/配置约定以详细设计为准。

**结论：保留“一套公共执行引擎 + 板级 YAML + 三类 NPU backend”的方案。公开资料已提供覆盖 RZ/V2H、RZ/V2N、RK3576、T527、A733 的接入路径，未发现需要分叉公共业务流程的架构性限制。** Pipeline、设备、库和模型资源继续由 YAML 选择。需要补齐的是模型数值语义、库装载契约和运行结果/生命周期接口。

这里的“可行”是源码与接口层面的判断。本次没有连接目标板，没有执行厂商二进制、模型转换或 NPU 推理；五个平台的 BSP、Camera、模型及长稳组合仍需分别验收。

## 1. 实际参考材料及版本

GitHub 材料固定到本次读取的提交，不把不同时间的 `main` 当成同一套 SDK：

| 材料 | 本次版本 | 用途 |
| --- | --- | --- |
| Renesas `rzv_ai_sdk` | [`ce4aed79`](https://github.com/renesas-rz/rzv_ai_sdk/tree/ce4aed79299f78183320f0435c96d6d041b90e75) | `R01_object_detection` 的 Camera → 检测应用行为、部署资源 |
| Renesas `rzv_drp-ai_tvm` | [`63c44ef7`](https://github.com/renesas-rz/rzv_drp-ai_tvm/tree/63c44ef7f14dd240ec4e2aa315ee84d06807f243) | Runtime Wrapper、链接依赖、YOLOv5 导出说明 |
| Rockchip `rknn_model_zoo` | [`837db8e9`](https://github.com/airockchip/rknn_model_zoo/tree/837db8e992815ff8647eadb709f07067f4bc3a71) | RK3576 支持、RKNN 调用、转换与后处理 |
| Radxa 链接的旧 Allwinner `ai-sdk` | [`fc90006d`](https://github.com/ZIFENG278/ai-sdk/tree/fc90006d0f6569da2f6726c2d8395877686f5aca) | 旧 `awnn_*` 包装、VIPLite 1.13/2.0、旧 YOLOv5 示例；不是 Allwinner 官方 GitHub 上游 |
| Radxa 分发的 Allwinner Model Zoo | 实际下载包 `awnpu_model_zoo-v0.9.0-20260116-83a67d4b` | 新预处理/后处理、T527/A733 导出 target、公共 VIP API、库依赖 |

Allwinner 材料有一个实测版本差异：[下载说明](https://docs.radxa.com/en/cubie/a7a/app-dev/npu-dev/model-zoo/model-zoo-download)展示 v1.0.0，但本次从其提供的[公开镜像](https://dl.radxa.com/cubie/allwinner-model-zoo.tar.gz)取得的目录及 `version` 文件均为 v0.9.0。本文关于包内源码的结论限定于实际下载包，不能直接套到未取得的 v1.0.0。

该归档 SHA-256：`f209bbcbc1222c52035ff0ea9796b9de6cdb96598a1c5cf7aaec2a20ab2673a5`。检查方法为源码静态阅读、ONNX protobuf 元数据读取及 `readelf` 检查，未运行包内程序。

## 2. 三类 backend 可以覆盖五个平台

| 目标 | 可复用的 backend 家族 | 模型与运行资源 | 结论 |
| --- | --- | --- | --- |
| RZ/V2H | `renesas_drpai` | 对应 SDK 编译的模型目录、Runtime Wrapper 和运行库 | 使用同一公共调用流程；板级资源选择平台/SDK 组合 |
| RZ/V2N | `renesas_drpai` | V2N 对应模型、运行库及硬件参数 | 与 V2H 共用适配逻辑方向成立；不能直接复制模型/性能门限 |
| RK3576 | `rockchip_rknn` | target 为 `rk3576` 的 `.rknn`、`librknnrt.so` | 官方示例明确支持；C API 映射直接 |
| T527 | `allwinner_viplite` | T527 `.nb`、`libVIPlite.so` → `libVIPuser.so` | 已核查包使用公共 `vip_*` 接口 |
| A733 | `allwinner_viplite` | A733 `.nb`、`libNBGlinker.so` → `libVIPhal.so` | 已核查包同样导出所需 `vip_*` 接口，可以复用 VIP 适配源码 |

Rockchip 平台支持和转换入口见[官方 YOLOv5 示例](https://github.com/airockchip/rknn_model_zoo/blob/837db8e992815ff8647eadb709f07067f4bc3a71/examples/yolov5/README.md)。Allwinner 的模型与平台用法见[板卡厂商示例说明](https://docs.radxa.com/en/cubie/a7a/app-dev/npu-dev/model-zoo/yolov5)；表中的 Allwinner 库依赖另由实际下载包的 ELF `DT_NEEDED` 核实。

**T527/A733 不必仅因库名不同就拆成两套 backend。** 本次检查的 `libVIPlite.so` 与 `libNBGlinker.so` 均导出 `vip_init`、`vip_create_network`、`vip_query_input/output`、`vip_set_network`、`vip_run_network`、`vip_cancel_network`、`vip_destroy` 等接口；该 Model Zoo 的公共头文件和封装也供两者使用。可先实现同一个 `VipApi` 函数表，由 YAML 选择导出 API 的库及依赖目录。

这证明了该软件包内的共用路径，不证明任意历史/未来 VIPLite 都有相同 ABI。若实际 BSP 的结构体或签名不同，只在 backend 内增加 ABI 适配构建，公共 worker 和产品版本不分叉。所查 T527/A733 路径均是 `vip_init(void)`；源码里的带内存参数形式由 `LIBVIP_VERSION_85X` 控制，不能误当成 T527 与 A733 的区别。

Renesas 的包装边界不同：[MeraDrpRuntimeWrapper.h](https://github.com/renesas-rz/rzv_drp-ai_tvm/blob/63c44ef7f14dd240ec4e2aa315ee84d06807f243/apps/MeraDrpRuntimeWrapper.h) 是 C++ 类，跨界包含 `std::string`、`std::tuple` 等；[构建文件](https://github.com/renesas-rz/rzv_drp-ai_tvm/blob/63c44ef7f14dd240ec4e2aa315ee84d06807f243/apps/CMakeLists.txt)链接 `mera2_runtime`、`mera2_plan_io`、`drp_tvm_rt`。建议将其封进自有 backend 模块，对公共 worker 提供窄 C ABI；YAML 仍决定模块及依赖位置。

一个产品版本不要求跨不兼容 BSP 原样复用一个 ELF。公共源码、控制协议、子项、判定和发布版本统一；构建产物沿用主程序已经采用的工具链/BSP 发布方式。

## 3. 最需要补齐的是模型 profile，而非另写平台业务流程

### 3.1 逻辑模型输入与实际提交格式分开描述

原需求的 `[1,3,640,640] / RGB / NCHW` 可以继续作为逻辑契约，但不能强制所有 runtime 都接收同一块 FP32/NCHW 内存。

| 已核查路径 | 示例向 runtime 提交的数据 | 归一化/转换边界 | 公共程序的处理 |
| --- | --- | --- | --- |
| Renesas Wrapper | `SetInput` 提供 FP32/FP16 指针接口 | 由选定模型及 PreRuntime 配置确定 | 查询输入类型，按模型包描述准备输入；不从其他平台继承归一化设置 |
| RKNN YOLOv5 | UINT8、NHWC、RGB；letterbox 填 114 | 转换脚本配置 mean=0、std=255；示例采用 runtime 输入转换路径 | 应用生成 UINT8 图像，不重复除以 255 |
| 旧 Allwinner `ai-sdk` | UINT8、平面 CHW、RGB | 示例未在 CPU 进行归一化；模型转换参数必须匹配 | 旧预处理实际 `copyMakeBorder` 填 0，不能把注释中的灰色当成执行结果 |
| 下载到的 Allwinner Model Zoo | UINT8、打包 RGB；letterbox 填 114 | 转换配置插入 `IMAGE_RGB` 预处理，mean=0、scale≈1/255 | 按该模型 binding 提交打包数据；不要继续使用旧示例的平面 CHW |

依据：[Renesas 输入接口](https://github.com/renesas-rz/rzv_drp-ai_tvm/blob/63c44ef7f14dd240ec4e2aa315ee84d06807f243/apps/MeraDrpRuntimeWrapper.h)、[RKNN 输入提交](https://github.com/airockchip/rknn_model_zoo/blob/837db8e992815ff8647eadb709f07067f4bc3a71/examples/yolov5/cpp/rknpu2/yolov5.cc)、[RKNN 转换配置](https://github.com/airockchip/rknn_model_zoo/blob/837db8e992815ff8647eadb709f07067f4bc3a71/examples/yolov5/python/convert.py)、[旧 Allwinner 预处理](https://github.com/ZIFENG278/ai-sdk/blob/fc90006d0f6569da2f6726c2d8395877686f5aca/examples/yolov5/yolov5_pre_process.cpp)。新 Allwinner 路径依据上述归档内 `examples/yolov5/yolov5_pre.cpp`、`convert_model/config_yml.py`。

建议分为 `logical_input` 与 `input_binding`：前者描述算法，后者描述本次 runtime API 实际接受的 dtype、内存顺序、转换模式及归一化执行方。特别是 RKNN，查询到的模型输入 layout 与 `rknn_inputs_set` 允许提交的 layout 并不必然一致。

公共预处理保留同一套 resize/letterbox 实现，通过 binding 选择最终打包方式。硬件预处理可后续替换其中步骤，但必须验证实际数值；CPU 参考路径不需要另造一套平台业务循环。

### 3.2 三个输出 head 并不自动等于同一种数值语义

| 模型路径 | 输出/解码依据 | 共享解码应采用的语义 |
| --- | --- | --- |
| Renesas 官方 YOLOv5 导出 | 截取三路检测卷积输出 | 对这条导出路径的 logits 执行 sigmoid |
| RKNN 官方优化 YOLOv5 | 输出布局示例为 `[1,255,H,W]`；后处理直接用输出计算 `2*x-0.5`、分数等 | 对已激活的数值直接解码，不能再次 sigmoid |
| 旧 Allwinner YOLOv5 | 示例按 anchor / H / W / 85 取值，并执行 sigmoid | 重排/反量化后按 logits 解码 |
| 下载到的 Allwinner Model Zoo | ONNX 输出 `[1,3,85,H,W]`；`detect_yolov5_rt` 使用对应索引并执行 sigmoid | 与旧版布局不同，仍是 logits 路径 |

依据：[Renesas 导出节点](https://github.com/renesas-rz/rzv_drp-ai_tvm/blob/63c44ef7f14dd240ec4e2aa315ee84d06807f243/docs/model_list/how_to_convert/How_to_convert_yolov5_onnx_models.md)、[RKNN 后处理](https://github.com/airockchip/rknn_model_zoo/blob/837db8e992815ff8647eadb709f07067f4bc3a71/examples/yolov5/cpp/postprocess.cc)、[旧 Allwinner 后处理](https://github.com/ZIFENG278/ai-sdk/blob/fc90006d0f6569da2f6726c2d8395877686f5aca/examples/yolov5/yolov5_post_process.cpp)。新 Allwinner 依据归档内 `examples/yolov5/yolov5_post.cpp` 和 `convert_model/yolov5s_rt.onnx`。

本次直接读取新 Allwinner ONNX 的元数据，得到：

```text
input:  images  FP32 [1,3,640,640]
output: 349     FP32 [1,3,85,80,80]
output: 497     FP32 [1,3,85,40,40]
output: 645     FP32 [1,3,85,20,20]
```

三路输出节点的 producer 为 `Reshape`；sigmoid 边界还结合示例解码确认。该 ONNX SHA-256 为 `fe69cda1852c7867dd7573acf013b415d98af1233a853ee92d39f1d8a579f4b9`。这是转换前 ONNX 的信息，转换后的 `.nb` 仍须查询实际输入/输出；`config_yml.py` 设置了插入预处理与输出反量化节点，外部 IO 可以因此改变。

网页运行日志出现六输出，而本次归档的 ONNX 与实际选用的后处理为三输出，二者不一致。不能据网页日志直接扩展正式契约，也不能直接认定新版本错误；先锁定实际采用的包和模型，再验证匹配关系。

共享解码建议接收 tensor view 和明确的 `activation`、量化、stride/anchor 映射。sigmoid 只做一次；统一 NMS、类别顺序和坐标还原。无需强制每帧先生成完整的 FP32 重排副本，也不应保留三家互不相同的整套后处理。

### 3.3 示例模型不能只凭名字混用

用户所指的 [R01_object_detection](https://github.com/renesas-rz/rzv_ai_sdk/blob/ce4aed79299f78183320f0435c96d6d041b90e75/R01_object_detection/README.md) 使用 YOLOv3、416 输入。它适合参考 Camera 与 runtime 的连接方式；本项目按原文继续采用 YOLOv5s 640 时，需使用相应模型资源与解码。

Renesas YOLOv5s 导出说明引用 v5.0 权重；Rockchip 提供优化模型及 ReLU 等不同资源。即便名字都叫 YOLOv5s，也不能假定权重、图变换和数值结果一致。模型包应记录源权重/ONNX hash、导出 recipe、compiler target 和参考结果。统一产品行为不要求量化结果逐位相同，但需要明确容差和验证基准。

## 4. 把现有 INpuBackend 补成可落地的共同契约

不必扩大为通用推理框架。现有 `probe/loadModel/validateContract/run/close` 可以保留，补上下面四项即可：

1. **描述实际 binding。** `loadModel` 返回模型逻辑说明、实际输入提交规格、每路输出描述和元数据来源。`shape/layout/dtype/stride/quantization` 与 `activation/anchors/classes` 分开：前一组尽可能 runtime 查询，后一组由经校验的模型 manifest 提供。
2. **明确 run 的完成条件。** 返回成功意味着本次输入已执行、输出已取回且可供 CPU 读取，包含必要 cache 同步；不是单纯“提交成功”。统一状态码保留失败阶段和可取得的原始错误码。
3. **明确输出所有权。** `OutputTensorSet` 使用 RAII lease 或等价释放协议；读取期间有效，释放后才允许复用有关内存。C ABI 跨模块时提供对应 release 回调，不让一侧释放另一侧分配的对象。
4. **固定 context 所属线程。** 首版由同一个推理线程完成初始化、运行、输出释放和关闭。停止请求由控制层发送，该线程执行清理；挂死则由独立 supervisor 处理。

各平台到共同操作的映射如下，省略了逐步错误清理代码：

| 操作 | Renesas | RKNN | VIPLite / NBGlinker |
| --- | --- | --- | --- |
| 加载 | `LoadModel(model_dir, address)` | `rknn_init` | `vip_init` → create/query network → create/set buffers → prepare |
| 元数据 | `GetInputDataType/GetNumOutput/GetOutput`，SDK 支持时补图/plan 元数据 | `rknn_query` 的 IO 数量及逐 tensor attr | `vip_query_network/input/output` |
| 输入与执行 | 按 dtype `SetInput` → `Run` | `rknn_inputs_set` → `rknn_run` | 写/同步输入 → `vip_run_network` |
| 输出 | `GetOutput`，保留对象与底层存储生命周期 | `rknn_outputs_get`，使用后 `rknn_outputs_release` | invalidate/cache 同步 → map/read；按 query 的量化信息解释 |
| 清理 | 由适配层完成 wrapper/context 清理 | 输出释放后 `rknn_destroy` | 按选定 SDK 顺序释放 network/buffer，最后 `vip_destroy` |

Renesas 当前 [wrapper 实现](https://github.com/renesas-rz/rzv_drp-ai_tvm/blob/63c44ef7f14dd240ec4e2aa315ee84d06807f243/apps/MeraDrpRuntimeWrapper.cpp)同时处理 `mera.plan` 与旧的 `deploy.json/.so/.params`。旧 runtime 分支的 `GetInputInfo/GetOutputInfo` 返回不支持；`GetOutput` 主要给 dtype、指针和元素数。此时应扩展查询或使用经 hash 固定的 manifest + 实际元素数校验，并如实记录来源，不能声称仅凭元素数验证了完整 shape。

Allwinner 的量化信息不只包含 affine scale/zero-point，旧 [封装](https://github.com/ZIFENG278/ai-sdk/blob/fc90006d0f6569da2f6726c2d8395877686f5aca/examples/libawnn_viplite/awnn_lib.c)还处理 dynamic fixed point。公共量化描述至少区分 `none / affine / dynamic_fixed_point`，按选定模型实际需要支持；遇到未实现类型明确拒绝。

Allwinner 公共 `vip_lite.h` 还要求 `vip_destroy` 与 `vip_init` 在同一线程，进一步支持首版采用一个 context 所属线程。无需把多个 Camera 变成多个 NPU context。

### 超时与错误传播的具体修正

- RKNN 头文件已有 `rknn_run_extend.timeout_ms`、`RKNN_ERR_TIMEOUT`；`rknn_outputs_get` 也可能阻塞。使用本地 timeout 的同时，独立 deadline 覆盖整个操作。[RKNN API](https://github.com/airockchip/rknn_model_zoo/blob/837db8e992815ff8647eadb709f07067f4bc3a71/3rdparty/rknpu2/include/rknn_api.h)
- VIPLite 头文件已有 `VIP_NETWORK_PROP_SET_TIME_OUT`，可在 prepare 前通过网络属性设置；还提供取消接口。应检查配置返回值，并在实际 BSP 上验证超时/取消行为，不能简单写成“只有进程 watchdog”。[旧版 1.13 头文件](https://github.com/ZIFENG278/ai-sdk/blob/fc90006d0f6569da2f6726c2d8395877686f5aca/viplite-tina/lib/aarch64-none-linux-gnu/v1.13/inc/vip_lite.h)
- 旧 `awnn_run()` 为 void；新 Model Zoo 的 `NetworkItem::network_run()` 已返回 int，但部分错误被合并为 -1，cache flush 失败仍只打印。正式 backend 应保留底层原始状态并让 cache 同步失败进入统一错误路径，不能直接把新 wrapper 的返回值当成所有步骤均成功。
- Renesas `Run` 仍为 void。可捕获异常、异常退出、超时与输出校验共同构成错误边界；原始状态码不可取得时明确标记，不伪造 vendor 成功码。

超时发生后能否重试、驱动是否恢复，仍需板端验证。进程结束与硬件恢复是两个独立结果。

## 5. YAML 的建议增量

沿用原设计的 `hardware.npu.devices` 和 `model_profiles`。以下为拟议字段与占位路径，展示配置职责，**不是可以直接部署的完整配置**：

```yaml
hardware:
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
            model: /opt/models/t527/yolov5s_rt_uint8_t527.nb
            sha256: "<packaged-model-sha256>"
            manifest: /opt/models/t527/yolov5s_manifest.yaml
```

同一已验证 VIP ABI 下，A733 profile 指向 `libNBGlinker.so`、包含 `libVIPhal.so` 的目录、A733 模型及其版本信息即可。无需更改业务入口、轮询逻辑、结果协议或判定算法。

模型 manifest 本身也可用 YAML，或内联进 `model_profiles`。建议补齐的内容为：

| 字段组 | 应表达的内容 |
| --- | --- |
| 来源/兼容 | 源模型与产物 hash、转换工具/参数、芯片 target、runtime 兼容组合 |
| `logical_input` | RGB、640、逻辑 layout、算法所需数值含义 |
| `input_binding` | 提交 dtype/layout、归一化执行方、应用或 runtime 转换模式 |
| 几何变换 | 插值、padding、取整及坐标还原参数 |
| `outputs` | 每个 head 的映射、实际维度顺序、stride、dtype、量化方式；激活前/后语义 |
| 检测语义 | anchors、类别顺序、confidence 公式、NMS 策略及输出上限 |
| 验证资源 | 固定输入/图片、参考结果、允许误差 |

库装载需要执行层配合：RKNN/VIP 可使用按已知头文件定义的函数表；Renesas bridge 按 SDK 编译。不要把函数签名、内存布局或任意调用脚本搬进 YAML，形成第二套解释执行语言。

Service 在启动 worker **之前**依据 YAML 设置依赖搜索环境，worker 再装载所选模块/API 库，记录实际加载结果。绝对路径只选定直接对象，其传递依赖仍有动态链接器搜索规则；配置了路径不等于所有依赖自动锁定。[Linux 动态装载规则](https://man7.org/linux/man-pages/man3/dlopen.3.html)

模型目录、预处理资源和依赖版本应作为一个经过验证的组合发布。发生缺库、符号或模型不匹配时统一报启动失败，不自动换另一套 runtime 或转成纯 CPU 推理。

## 6. Camera 与业务循环保持共享

建议继续由一个公共 GStreamer 采集实现读取板级 pipeline，以约定的 appsink 为出口；第一版采用可控复制和有界帧池。各 BSP 若已能协商为统一 RGB/BGR CPU 图像，公共预处理直接接入。需要特殊采集/硬件转换时在 Camera 或预处理适配内部增加路径。

公共采集层负责协商格式检查、图像 stride/存储生命周期、ERROR/EOS、首帧/失流 timeout、到帧时间及每路计数。appsink 本身有队列，不能仅限制应用层 latest-frame 槽；队列/丢弃属性按目标 BSP 的 GStreamer 版本配置并检查。[GStreamer appsink](https://gstreamer.freedesktop.org/documentation/app/appsink.html)

当前阶段的采集链路尽量独立于 X11/Wayland 和显示 sink；原 YAML 的显示环境 profile 可以保留作后续 UI 选择，但是否出帧不应依赖 Qt 卡片已创建。

程序逻辑仍然是：

```text
板级 YAML → 装载并验证 Camera / backend / 模型组合
                         ↓
各路持续采集 → latest-frame → 公平轮转取新帧
                         ↓
共享预处理 → backend.run → 输出语义适配 → 共享 YOLO 解码/NMS
                         ↓
Detection + Camera ID + frame sequence + 时间/指标 → ResultSink
```

厂商 demo 用于核对 API 顺序、模型格式和数值参考。将其图片读写、绘框显示、全局变量和整条主循环搬进产品，会失去统一生命周期与统计口径；正式 worker 直接调用 backend，Qt 后续消费同一 ResultSink。

## 7. 对原文的具体修订建议与落地顺序

| 位置 | 本次新增的具体修订 |
| --- | --- |
| 需求 §5；设计 §7 | 区分逻辑输入与 API binding；添加归一化执行方、输出激活及实际 layout；共享 sigmoid 改为按模型语义选择 |
| 设计 §5 | 补输出 lease、同步完成语义、context 所属线程和元数据来源 |
| 设计 §6.1 | 明确 Renesas C++ bridge 与模型目录版本；不能把 R01 的 YOLOv3 解码用于 YOLOv5 |
| 设计 §6.2 | RKNN 示例 UINT8/NHWC 提交及 runtime 转换；输出取回/释放属于完整操作 |
| 设计 §6.3 | T527/A733 在所查包内可共用 VIP API；更新不同 API 库与依赖名；区别旧 awnn 与新 wrapper 的错误传播 |
| 设计 §6.3 | 更新 A733 target：所查导出脚本明确将 `a733` 映射到 `VIP9000NANODI_PLUS_PID0X1000003B`，`t527` 映射到 `VIP9000NANOSI_PLUS_PID0X10000016`；与实际 BSP/compiler 组合验证 |
| 设计 §8 | 沿用板级 YAML，补模块/库装载配置、模型 manifest 及实际生效记录 |
| 设计 §11 | 同时采用已支持的 native timeout 和 Service deadline；cache 同步失败进入失败判定 |

建议按三个可检查的阶段推进：

1. **先固定接口与资源契约。** 三个 backend 家族共同审核 Frame/InputBinding/TensorView/Result/OutputLease；选定各板 BSP/runtime/模型资源。现有原型不需要 UI。
2. **用同一验证入口接通五个平台。** 相同固定图片/输入测试流程验证 load、真实 NPU 执行、参考检测结果、错误传播和释放。优先在现有 Renesas 板上执行，同时完成另外两类 backend 的构建与接口映射；没有板端验证的列继续标为待验收。
3. **接入同一连续 Camera 循环并做压力验收。** 每路新帧、公平调度、停止/重启、失流/超时、慢消费者及与其他压力项并发都走公共实现。五个平台达到同一功能和失败判定契约后，再接 Qt 卡片。

最小可行实现无需首先完成多 context、零拷贝、通用模型框架或 UI。公共引擎 + 三类薄适配 + 经验证的 YAML/模型包，已经有明确的逐平台实现依据。
