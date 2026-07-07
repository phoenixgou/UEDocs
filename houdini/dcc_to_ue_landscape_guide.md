# DCC 到 UE Landscape 指南（归档来源）

> **归档提示**：本文件只记录原始调研来源，不作为当前执行依据。
> 当前唯一事实源见 `landscape_pipeline_master.md`。
> 1~2 km² 垂直切片 SOP 见 `sop_vertical_slice_1to2km.md`。

## 归档原因

原文件包含早期 DCC 到 UE Landscape 流程中的旧比例公式、旧路径示例和过长步骤描述。
这些内容已被拆分为一个事实源和两个案例 SOP。

为避免旧公式与当前规则并存，本文件不再保留可执行步骤、命令、坐标公式或路径示例。

## 保留价值

- 主题：从 DCC / GIS / Houdini 等工具生成 UE Landscape 可导入数据。
- 背景：用于解释垂直切片地形验证的历史来源。
- 当前结论：以 `landscape_pipeline_master.md` 的单位、尺寸、导入、验证规则为准。

## 当前阅读顺序

1. `landscape_pipeline_master.md`
2. `sop_vertical_slice_1to2km.md`
3. `sop_undini_uk_5km.md`
4. `ue58_bigworld_preresearch.md`

## 第一性原理口径

- DCC 负责生成或清洗高度数据。
- UE Landscape 负责按高度场导入和渲染。
- XY 覆盖范围、Z 高度范围和世界原点必须分别验证。
- 旧来源文档只解释背景，不承载当前执行标准。
