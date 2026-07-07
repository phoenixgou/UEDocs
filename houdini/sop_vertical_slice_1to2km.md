# SOP：1~2 km² 垂直切片地形导入

> 公式（Z Scale、XY Scale、Z Location）见 `landscape_pipeline_master.md` 第 3 节，本文不重复。
> 预研背景与性能验收矩阵见 `ue58_bigworld_preresearch.md`。

---

## 路线选择

- **默认路线**：Landscape + Nanite + RVT + World Partition。适用 Houdini 能输出目标 16-bit 高度图时。
- **对照路线**：Mesh/Nanite（Heightfield → OBJ → Interchange 导入并 `Enable Nanite`）。仅在两种情形使用：Houdini Apprentice 无法输出目标高度图；或预研需要“补充负空间 / 纯 Nanite Mesh 地形”对照评估。许可能力以本机实测为准（见主文档第 7 节）。

## 1. 数据源

- 挪威 Hoydedata / LaserInnsyn、英国 `data.gov.uk`、美国 USGS Lidar Explorer，均选 **DTM 1 m**、**GeoTIFF**。
- 框选比目标略大（约 2×2 km 预留裁剪余量）。用 DTM，不用 DSM。

## 2. QGIS 预处理（地理坐标必做）

地理坐标单位为度，直接进 Houdini 会形变，必须重投影到米制：

1. **Merge**：`Raster → Miscellaneous → Merge`（跨瓦片拼接）。
2. **Reproject**：`Raster → Projections → Warp`，Target CRS 选米制 UTM（挪威 `UTM zone 32N` / 英国 `OSGB36 British National Grid` / 美国对应 UTM 带），Resampling `Bilinear`。
3. **Clip**：`Raster → Extraction → Clip Raster by Extent`，框选 1.5×1.5 或 2×2 km，另存 GeoTIFF（32-bit Float）。

## 3. Houdini（默认路线）

```
Heightfield File (GeoTIFF, Layer Name = height)
  └ Heightfield Resample (Set Target Resolution, Grid Spacing 1.0, Target 1009 或 2017)
     └ [可选] Heightfield Erode
        └ (Remap 前记录 height min/max)
        └ Heightfield Remap (Input min/max → Output 0/1)
           └ Heightfield Output (16-bit U16, Height Layer=height, Remap 0 to 1)
```

- `Grid Spacing 1.0` = `1 m/quad` → 导入时 `XY Scale = 100`（见主文档 3.2）。
- 目标尺寸取主文档第 4 节合法组合：1009≈1 km、2017≈2 km。
- **对照路线**（替换 Output）：`Convert HeightField`（Polygon Mesh，Division Method By Size = 1）→ [可选 PolyReduce] → `File` 存 `.obj`。

## 4. UE 导入

`Shift+2` → Landscape → Import from File：

- **Scale**：`X = Y = 100`（1 m/quad）；`Z` 按主文档 3.1，用记录的 min/max（示例 min 120.45 / max 840.85 → `Z Scale ≈ 140.70`）。
- **Location**：需要海平面基准时按主文档 3.3 填 `Z`（示例 Mid 480.65 → `48065` cm）；否则保持 `0`。
- **Material**：分配支持 RVT 混合的地形材质。
- **组件复核**：以面板显示的 Section Size / Components / Overall Resolution 与 Houdini 目标一致为准（如 1009 → 8×8 Components）。

**对照路线**：Interchange 导入 `.obj`，勾选 `Enable Nanite`，指定基础材质。

## 5. 集成与验收

- Nanite / RVT / World Partition / PSO Precache 配置：见主文档第 5 节。
- 性能验收（单机/穿越/战斗三主线，采集命令与输出路径）：见主文档第 6 节。
