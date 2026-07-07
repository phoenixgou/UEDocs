# SOP：UNDINI / UK LiDAR / 5 km / 4033 地形导入

> 案例来源：UNDINI《Import Real Landscapes into Unreal in under 30 minutes using Houdini and real world DTM data》。
> 公式（Z Scale、XY Scale、Z Location）见 `landscape_pipeline_master.md` 第 3 节，本文不重复。
> UE 5.8 集成与性能验收见主文档第 5、6 节。

---

## 适用

- 单张 UK LiDAR Composite DTM 瓦片，约 5000×5000 m，导出 4033×4033 的 16-bit 高度图，单张 Landscape。
- 覆盖范围受数据源限制，不做多瓦片拼接；跨瓦片/需重投影时走 `sop_vertical_slice_1to2km.md` 的 QGIS 流程。
- Houdini 需能输出 4033 高度图；许可能力以本机实测为准（见主文档第 7 节）。

## 1. 选址与下载

1. Google Maps 卫星/3D 视图定位目标地形，记地名或坐标。
2. UK LiDAR Portal（`data.gov.uk`，搜 `uk lidar dtm`）进入 Survey Download，切换到 DTM 1 m 图层。
3. 框选区域 → `Get Available Tiles` → 勾选瓦片（如 `NT81SE`）下载并解压得到 `.tif`。

- 用 DTM，不用 DSM。文件超过 Houdini Max File Size 时先裁剪。

## 2. Houdini 节点链

```
Heightfield File (.tif)
  └ Heightfield (Size 5000×5000, Division Mode By Axis, Grid Resolution 4033)
     └ Heightfield Project  (绿:目标网格 / 白:原始 DTM)
        └ Heightfield Remap (Compute Range → 记录 Input Min/Max → Output 0/1)
           └ Heightfield Output (16-bit fixed, Single/height, Save to Disk → .png)
```

- **必须记录 Input Min / Max**（示例 71 / 851），供 Z Scale 使用。
- 导出全白/全黑 = Remap 的 Output Min/Max 设错，重查。

## 3. UE 导入与比例

`Shift+2` → Landscape → Import from File，选 `.png`，**导入前**填 Scale：

- **XY**：5000 m 覆盖 4033 顶点高度图，水平间距按 **4032 quads** 计算（不是 4033 像素）。
  `XY Scale = 5000 / 4032 * 100 ≈ 124.01`（算式见主文档 3.2）。
- **Z**：`Total Height = 851 − 71 = 780 m`，`Z Scale = 780 / 512 * 100 ≈ 152.34`（算式见主文档 3.1）。
- **Z Location**：本案例不做海平面对齐，保持 `0`；需要绝对海拔时按主文档 3.3。

填 `X = Y ≈ 124.01`、`Z ≈ 152.34` → `Import` → `Shift+1` 退出，即得 1:1 还原地形。

## 4. 收尾约束

- 单张 4K 级 Landscape 足够：不用 World Composition，不用 8K（编辑器/Cook 极慢）。
- 大气透视用 Time of Day 类模板（体积云 + Sky Atmosphere）辅助尺度感，非管线必需。
