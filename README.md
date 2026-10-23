# dangku-test

NPU / Camera 稳定性测试的需求、设计与平台核查资料。当前目录尚无主程序源码或 backend 实现。

| 文档 | 用途 |
| --- | --- |
| [17 — 需求](17_npu_stress_requirements.md) | 同一产品覆盖五个平台的功能、判定与验收约束 |
| [18 — 详细设计](18_npu_stress_detailed_design.md) | 公共引擎、backend、YAML/模型契约及生命周期；实现以此为准 |
| [19 — 可行性评审](19_npu_camera_feasibility_review.md) | 评审过程、风险与取舍依据 |
| [20 — 平台兼容核查](20_npu_platform_compatibility.md) | 固定版本的厂商示例、API、模型和运行库证据 |

2026-10-23：已将评审结论合入 17、18。当前优先实现程序逻辑，Qt 卡片后续集成；板端配置和五平台验收仍需完成。
