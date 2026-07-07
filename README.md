# UEDocs — UE 5.8.0 开放世界预研文档库

本目录收录 UE 5.8.0 主机级开放世界预研的技术文档与工作流指引。地形管线以 **主文档** 为唯一事实源，案例操作拆分为两份 **SOP**，公式集中在主文档、不在各处重复。

## 文档索引

| 文件 | 角色 | 描述 |
|------|------|------|
| [landscape_pipeline_master.md](./landscape_pipeline_master.md) | **地形管线主文档（唯一事实源）** | 公式、坐标系、推荐尺寸、UE 5.8 集成、性能验收路径、风险边界 |
| [sop_undini_uk_5km.md](./sop_undini_uk_5km.md) | **SOP** | UNDINI / UK LiDAR / 5 km / 4033 单张 Landscape 案例；公式引用主文档 |
| [sop_vertical_slice_1to2km.md](./sop_vertical_slice_1to2km.md) | **SOP** | 1~2 km² 垂直切片；默认 Landscape+Nanite+RVT+WP，Mesh/Nanite 为对照路线；公式引用主文档 |
| [ue58_bigworld_preresearch.md](./ue58_bigworld_preresearch.md) | 预研记录 | DCC→Runtime 性能闭环、源码锚点、核心判断、验收矩阵 |
| [houdini_lidar_to_ue_terrain.md](./houdini_lidar_to_ue_terrain.md) | 来源材料（已归档，不作为执行依据） | 原始 UNDINI 工作流整理，内容已并入主文档 + `sop_undini_uk_5km.md` |
| [dcc_to_ue_landscape_guide.md](./dcc_to_ue_landscape_guide.md) | 来源材料（已归档，不作为执行依据） | 原始垂直切片指引，内容已并入主文档 + `sop_vertical_slice_1to2km.md` |
| [ue58_architecture.md](./ue58_architecture.md) | 参考 | UE 5.8 引擎架构概览 |
| [ue58_toolsets.md](./ue58_toolsets.md) | 参考 | UE 5.8 工具链说明 |
| [unreal_actor_placement_sop.md](./unreal_actor_placement_sop.md) | 参考 | Actor 放置与编辑器操作 SOP |

## 阅读顺序

```
landscape_pipeline_master.md      ← 唯一事实源：公式 / 坐标系 / 尺寸 / 集成 / 验收 / 风险
        ├── sop_undini_uk_5km.md          ← 案例 SOP：UK 5 km / 4033
        └── sop_vertical_slice_1to2km.md  ← 案例 SOP：1~2 km² 垂直切片
ue58_bigworld_preresearch.md      ← 预研决策与源码锚点（背景）
```

地形管线的实际操作以主文档 + 两份 SOP 为准；`houdini_lidar_to_ue_terrain.md` 与 `dcc_to_ue_landscape_guide.md` 仅作来源材料归档，不作为执行依据。
