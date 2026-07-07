# UE 地形管线主文档（唯一事实源）

> 责任：本文件是 DTM → Houdini → UE Landscape 地形管线的唯一事实源，只收录公式、坐标系、推荐尺寸、UE 5.8 集成、性能验收路径、风险边界。
> 案例操作步骤见 SOP：`sop_undini_uk_5km.md`、`sop_vertical_slice_1to2km.md`。
> 预研背景、核心判断与完整源码锚点见 `ue58_bigworld_preresearch.md`；该文件是历史预研记录，机器路径需按当前工作区复核。

---

## 1. 术语

- **DTM**：数字地形模型，裸地高程，去除建筑/植被。地形基础用它。
- **DSM**：数字地表模型，含建筑/植被高度，用于提取负空间或资产散布参考。
- **DEM**：DTM 与 DSM 的统称。

## 2. 坐标系与单位

- UE 采用左手坐标系，Z 轴向上，长度单位为厘米：`1 m = 100 cm`。
- Landscape 高度用 16-bit 无符号整数（`0`–`65535`）表示。
- `Z Scale = 100` 时，垂直范围为 ±256 m（共 512 m），灰度 `32768` 对应 `Z = 0`。
- Landscape 默认 `1 quad = 1 m`（`XY Scale = 100`）。高度图 `N` 个顶点对应 `N − 1` 个 quad。

## 3. 换算公式（唯一定义处）

### 3.1 Z Scale（米制，唯一写法）

```
Z Scale = TotalHeightMeters / 512 * 100
```

`TotalHeightMeters = DTM Max − DTM Min`，在 Houdini Remap 归一化前记录。

### 3.2 XY Scale（按 quad 数，不按像素数）

```
XY Scale  = RealSpanMeters / QuadCount * 100
QuadCount = HeightmapVertices - 1
```

若 Houdini 已重采样到 `1 m/quad`，则 `XY Scale = 100`。

### 3.3 绝对海拔对齐（仅在需要海平面基准时使用）

```
MidHeightMeters      = (DTM Min + DTM Max) / 2
Landscape Z Location = MidHeightMeters * 100        (单位: cm)
```

默认导入把高度图中点对齐到 `Z = 0`。**仅当**要求 `Z = 0` 对应真实海平面 0 m 时，才把 Landscape Actor 沿 Z 抬高该值；否则保持 `Z Location = 0`。

## 4. 推荐高度图尺寸

优先以 UE 导入面板给出的合法 Section Size / Components 组合复核；下表为常用顶点数及其分解：

| 顶点数 | 分解 | quad 数 |
|--------|------|---------|
| 1009 | 8×126+1 | 1008 |
| 2017 | 16×126+1 | 2016 |
| 4033 | 32×126+1 | 4032 |
| 8129 | 32×254+1 | 8128 |

- `1 m/quad` 时：1009≈1 km、2017≈2 km、4033≈4 km、8129≈8 km。
- 尺寸越大，编辑器与 Cook 越慢；无明确覆盖需求时，取满足范围的最小合法尺寸。

## 5. UE 5.8 集成

- **Nanite 地形**：Landscape Details 勾选 `Enable Nanite`，引擎生成 `LandscapeNaniteComponent`，用 Nanite 代理替换 CPU LOD。
- **RVT**：新建 Runtime Virtual Texture 资产，关卡放置 RVT Volume 绑定，材质用 `Runtime Virtual Texture Sample` 读取，实现按坡度/高度自动分层。
- **World Partition**：用 Open World 模板或 `World Settings → World Partition` 开启；Landscape 自动注册，Grid Cell Size 建议 256/512 m。
- **PSO Precache**：确认 `r.PSOPrecache.Enable` 已开启，避免 Teleport/快速加载首帧编译尖峰。

源码锚点（工作区相对路径，完整表见 `ue58_bigworld_preresearch.md`）：

- `5.8.0\Engine\Source\Runtime\Landscape\Classes\LandscapeNaniteComponent.h`
- `5.8.0\Engine\Source\Runtime\Landscape\Public\LandscapeHLODBuilder.h`
- `5.8.0\Engine\Source\Runtime\Engine\Public\WorldPartition\WorldPartitionStreamingSource.h`
- `5.8.0\Engine\Source\Runtime\Engine\Public\PSOPrecache.h`
- `5.8.0\Engine\Plugins\Interchange\Runtime\Source\Pipelines\Public\InterchangeGenericMeshPipeline.h`

## 6. 性能验收路径

首轮三条主线：单机场景、穿越流送、战斗压力。

采集与输出（**不要**把运行时性能数据写到 `%APPDATA%\Unreal Engine\LocalBuildLogs`）：

- **CSV Profiler**：`csvprofile start` / `csvprofile stop`，输出到项目 `Saved\Profiling\CSV`。
- **Trace / Unreal Insights**：输出到项目 `Saved\Profiling`，或用 `-tracefile=` 显式指定路径。
- **GPU**：`stat gpu` / GpuProfilerTrace。

核查点：

- XY/Z Scale 经第 3 节公式验证，与真实尺度一致。
- `Enable Nanite` 已勾选；`showflag.NaniteVisualization 1` 确认地形走 Nanite。
- RVT Volume 覆盖全地形，无穿插/黑斑。
- 双轨验证：`P` 键显示碰撞与视觉贴合；`BuildNavigation` 覆盖预设穿越路线可行走区域。
- World Partition 分区正确；Teleport 无 PSO 首帧尖峰；30 min soak 观察 `Nanite.StreamingPoolSize` 漂移。

源码锚点：

- `5.8.0\Engine\Source\Runtime\Core\Public\ProfilingDebugging\CsvProfiler.h`
- `5.8.0\Engine\Source\Runtime\Core\Public\ProfilingDebugging\TraceAuxiliary.h`
- `5.8.0\Engine\Source\Runtime\RHI\Public\GpuProfilerTrace.h`

## 7. 风险边界

- **数据源**：用 DTM，不用 DSM（DSM 含建筑/植被噪点）。
- **双轨**：Nanite/Landscape 可视表达与碰撞/导航不共享数据，必须双向验证（看得到走得通、走得通看得到）。
- **Experimental**：`VirtualHeightfieldMesh`、`NaniteDisplacedMesh` 位于 `Experimental`，垂直切片验证通过前不进默认生产。
- **Houdini 许可**：能否输出目标高度图，以本机 Houdini 版本和许可实测为准；若无法输出目标高度图，则走 Mesh/Nanite 对照路线（见 `sop_vertical_slice_1to2km.md`）。
- **坐标系**：地理坐标（经纬度/度）必须先在 QGIS 重投影到米制 UTM 再进 Houdini，否则产生水平/垂直形变。
