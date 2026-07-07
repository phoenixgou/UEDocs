# Houdini LiDAR 到 UE 地形管线（归档来源）

> **归档提示**：本文件只记录原始调研来源，不作为当前执行依据。
> 当前唯一事实源见 `landscape_pipeline_master.md`。
> UNDINi / UK / 5 km / 4033 案例 SOP 见 `sop_undini_uk_5km.md`。

## 归档原因

原文件包含早期调研中的旧公式、旧机器路径和旧执行假设。
这些内容已被重新校正并收敛到当前主文档中。

为避免同一主题存在多套可执行口径，本文件不再保留操作步骤、命令、坐标公式或路径示例。

## 保留价值

- 主题：从 Houdini / UNDINi 获取 LiDAR / DSM 数据，并导入 Unreal Engine Landscape。
- 背景：用于验证真实地形数据进入 UE 的技术路线。
- 当前结论：以 `landscape_pipeline_master.md` 的单位、尺寸、导入、验证规则为准。

## 当前阅读顺序

1. `landscape_pipeline_master.md`
2. `sop_undini_uk_5km.md`
3. `sop_vertical_slice_1to2km.md`
4. `ue58_bigworld_preresearch.md`

## 第一性原理口径

- Landscape 是高度场，不是完整网格建模系统。
- XY 方向由顶点间距决定。
- Z 方向由灰度高度范围、导入 Z Scale 和 Actor Z Location 共同决定。
- 地形文档只保留一个事实源；案例 SOP 只负责把事实源参数化。
