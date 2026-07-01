# UEDocs — UE 5.8.0 开放世界预研文档库

本目录收录 UE 5.8.0 主机级开放世界预研过程中产出的技术文档与工作流指引。

## 文档索引

| 文件 | 描述 | 更新时间 |
|------|------|---------|
| [ue58_bigworld_preresearch.md](./ue58_bigworld_preresearch.md) | **主预研记录**：DCC 到 Runtime 性能闭环全貌，含已验证源码锚点、管线总览图、核心判断、Nanite 资产优先级分析、性能验收矩阵及下一步垂直切片计划 | 2026-07-01 |
| [houdini_lidar_to_ue_terrain.md](./houdini_lidar_to_ue_terrain.md) | **DCC 管线操作指引**：真实 LiDAR/DTM 数据 → QGIS 预处理 → Houdini 高度场节点链 → UE Landscape 导入与 XYZ 比例精确换算；含 Apprentice 许可替代路线、UE 5.8 Nanite 地形/World Partition/PSO Precache 集成，以及 1~2 km² 垂直切片性能验收清单 | 2026-07-01 |
| [ue58_architecture.md](./ue58_architecture.md) | UE 5.8 引擎架构概览 | — |
| [ue58_toolsets.md](./ue58_toolsets.md) | UE 5.8 工具链说明 | — |
| [unreal_actor_placement_sop.md](./unreal_actor_placement_sop.md) | Actor 放置与编辑器操作 SOP | — |

## 文档关系

```
ue58_bigworld_preresearch.md       ← 顶层预研方向与决策
        │
        ├── houdini_lidar_to_ue_terrain.md   ← DCC 管线落地操作细节
        │         （DTM 下载 → QGIS → Houdini → UE Landscape → 验收清单）
        │
        ├── ue58_architecture.md             ← 引擎架构参考
        └── ue58_toolsets.md                 ← 工具链参考
```