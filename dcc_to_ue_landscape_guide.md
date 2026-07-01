# UE 5.8.0 预研指引：从真实世界 DTM 数据到 UE Editor 场景地形导入全流程

本指引旨在为 **1~2 km² 的 DCC -> UE Editor 垂直切片**提供完整、严谨且可重复的落地操作步骤。主要内容基于 George Hulm (UNDINI 频道) 的主流地形管线实践，并结合本地源码环境进行适配，打通 **真实 DTM 下载 → QGIS 预处理 → Houdini 高度场处理与重采样 → 虚幻引擎 Landscape 导入与高度精确对齐** 的性能闭环。

---

## 1. 概念与软件准备

在进入具体步骤前，需厘清核心概念并准备相应软件：

### 1.1 核心概念
*   **DTM (Digital Terrain Model，数字地形模型)**：剔除了建筑物、桥梁和植被等人工与地表覆盖物，仅保留纯粹“裸地”高程的数据。适合直接作为游戏关卡地形基础。
*   **DSM (Digital Surface Model，数字地表模型)**：包含植被、建筑物等表面高度的数据。通常用于提取负空间（建筑群、森林边界）做资产散布。
*   **DEM (Digital Elevation Model，数字高程模型)**：在很多数据源中，DEM 是 DTM 和 DSM 的统称。

### 1.2 软件与插件要求
1.  **QGIS (3.x 或更高版本)**：开源地理信息系统，用于多瓦片拼接、投影坐标系转换及裁切。
2.  **Houdini (19.5 / 20.0+ / 20.5)**：
    *   **Commercial / Indie 版**：可无限制输出 $4K$ 级别的 16-bit 像素高度图。
    *   **Apprentice 版 (免费版)**：高度图输出有 $1280 \times 720$ 分辨率上限。本文第 4 节提供 Apprentice 转换为几何体（OBJ）导入的性能验证替代方案。
3.  **Unreal Engine 5.8.0**：用于最终的地形渲染、Nanite 激活及 World Partition 流送验证。

---

## 2. 第一阶段：获取真实世界 DTM 数据

以下为三个主流的公开高精度高程数据源获取路径：

### 2.1 挪威数据源 (Hoydedata / LaserInnsyn)
1.  访问 [Hoydedata Portal](https://hoydedata.no/LaserInnsyn/)。
2.  在地图上定位到目标区域（如挪威标志性的峡湾或山脉）。
3.  使用左侧的选择工具（Draw Polygon / Rectangle）框选目标区域（规划 1~2 km² 范围，建议框选约 $2\text{km} \times 2\text{km}$ 预留剪裁余量）。
4.  在右侧的 **Products** 面板中，选择 **DTM 1m**（即 1 像素代表 1 米的高精度地形模型）。
5.  格式选择 **GeoTIFF**，点击生成并下载。

### 2.2 英国数据源 (data.gov.uk)
1.  访问 [UK Lidar Composite DTM](https://data.gov.uk/dataset/3fc40781-7980-42fc-83d9-0498785c600c/lidar-composite-dtm-2019-1m)。
2.  根据提供的数据图层（National Grid Reference），搜索对应经纬度瓦片。
3.  下载对应的 1m 精度 DTM 压缩包（内含 `.tif` 格式的栅格高程图）。

### 2.3 美国数据源 (USGS Lidar Explorer)
1.  访问 [USGS Lidar Explorer](https://prd-tnm.s3.amazonaws.com/LidarExplorer/index.html#)。
2.  在搜索框输入目标山峰（如 Mt. Rainier 或 Yosemite），或者在右侧勾选 **DEMs (1 meter)** 覆盖层。
3.  框选感兴趣的区域，在下载列表中找到对应的 **GeoTIFF** 数据块下载。

---

## 3. 第二阶段：QGIS 地理信息预处理

真实地理数据下载后，其空间坐标系统（Coordinate Reference System, CRS）通常采用地理坐标（如 WGS84 经纬度单位）。直接导入 Houdini 会因为坐标单位为度（Degree）而不是米（Meter）导致严重的水平/垂直形变。因此必须进行重投影。

### 3.1 步骤 1：导入与合并 (Raster Merge)
1.  启动 QGIS，将下载的一个或多个 GeoTIFF 文件直接拖入图层面板（Layers）。
2.  若选区横跨多个瓦片，点击顶部菜单：**Raster (栅格) -> Miscellaneous (杂项) -> Merge (合并)**。
3.  在 **Input layers** 中勾选所有待拼接的 DTM 图层，点击运行，生成一个合并后的临时栅格图层。

### 3.2 步骤 2：重投影至 UTM 投影坐标系 (Reproject/Warp)
1.  点击顶部菜单：**Raster (栅格) -> Projections (投影) -> Warp (Reproject) (变形/重投影)**。
2.  **Input layer** 选择上一步拼接的图层。
3.  **Target CRS**（目标坐标系）：点击右侧地球图标，搜索目标区域所在的 **UTM 投影带**（例如挪威常用 `WGS 84 / UTM zone 32N`，英国常用 `OSGB36 / British National Grid`，美国使用对应的 UTM 区带）。投影坐标系的单位必须为**米（Meters）**。
4.  点击运行，生成投影转换后的新图层。

### 3.3 步骤 3：裁剪出垂直切片区域 (Clip by Extent)
1.  点击顶部菜单：**Raster (栅格) -> Extraction (提取) -> Clip Raster by Extent (按范围裁剪栅格)**。
2.  选择投影后的图层，在 **Clipping extent** 中选择 **Draw on Canvas (在画布上绘制)**。
3.  框选一个精确的 $1.5\text{km} \times 1.5\text{km}$ 或 $2\text{km} \times 2\text{km}$ 矩形区。
4.  在底部输出路径中，将裁剪结果另存为 **GeoTIFF** 格式（保存为 32-bit Float 或 16-bit Int，视精度要求而定）。

---

## 4. 第三阶段：Houdini 节点管线处理

将预处理好的 GeoTIFF 导入 Houdini 中，利用高度场（Heightfield）节点进行网格对齐、侵蚀细节增强以及格式导出。

### 4.1 Houdini 核心节点网格搭建

在 `/obj` 级创建一个 `Geometry` 节点，进入内部，建立如下节点链：

```
[hf_file] (导入 TIFF)
    │
[hf_clip] / [hf_crop] (精确修剪)
    │
[hf_resample] (重采样对齐 UE 规格)
    │
[hf_erode] (可选：模拟侵蚀增加微地貌)
    │
[hf_remap] (将绝对高度映射至 [0, 1] 范围)
    │
[hf_output] (导出 16-bit PNG 高度图)
```

### 4.2 各节点详细配置指南

#### 1. `Heightfield File` [NEW]
*   **作用**：将 GeoTIFF 栅格数据转换为 Houdini 的 2D 标量体层（Height 与 Mask）。
*   **核心配置**：
    *   **File**：指向 QGIS 裁剪出的 GeoTIFF 绝对路径。
    *   **Size**：保持默认或指定地理实际尺寸（若 TIFF 带有地理元数据，Houdini 会自动匹配）。
    *   **Layer Name**：设置为 `height`。

#### 2. `Heightfield Resample` [NEW]
*   **作用**：将地形分辨率和栅格间距调整为 Unreal Engine 建议的分辨率，避免引擎在导入时进行质量受损的拉伸。
*   **核心配置**：
    *   **Resolution Method**：选择 **Set Target Resolution**。
    *   **Grid Spacing**：设置为 `1.0`（这意味着 1 像素对应 1 米，完全匹配 Unreal 的默认 Quad 大小）。
    *   **Target Size**：根据 1~2 km² 的垂直切片需求，我们选择 Unreal 官方推荐的 Landscape 尺寸（遵循 $N \times 127 + 1$ 准则）：
        *   **1009 x 1009**：代表约 $1\text{km} \times 1\text{km}$ 的地形区块（总像素 101.8 万）。
        *   **2017 x 2017**：代表约 $2\text{km} \times 2\text{km}$ 的地形区块（总像素 406.8 万）。
        *   在节点中，将 Size 填入对应的像素大小，确保 Grid Spacing 为 1。

#### 3. `Heightfield Erode` (可选) [NEW]
*   **作用**：生成写实的自然沟壑、积水与流沙层（Flow, Sediments），为材质划分提供程序化 Mask。
*   **核心配置**：
    *   **Resolution**：保持与 Resample 一致。
    *   **Erosion Type**：建议选择 **Thermal / Precipitation**。
    *   **Visualization**：勾选以在视口预览侵蚀效果，可通过降水和侵蚀强度滑块调整山体风化程度。

#### 4. 获取绝对高度（中间数据记录）
*   **操作**：在 `hf_remap` 之前，鼠标中键点击当前计算节点。
*   **目标**：记录弹出信息中的 `height` 属性的最小值（`min`）和最大值（`max`）。
    *   *假设读出数据*：`min = 120.45`，`max = 840.85`。
    *   *计算总高差 (Height Range)*：$840.85 - 120.45 = 720.40$ 米。
    *   *记录此数值*，这对于第五阶段的 Unreal Z Scale 计算是**唯一且绝对**的关键输入。

#### 5. `Heightfield Remap` [NEW]
*   **作用**：将真实的物理高度（以米为单位）归一化映射到 `[0, 1]` 范围。如果不做归一化，直接输出 16-bit PNG 会在 0 以下截断，并在 1 以上严重过载。
*   **核心配置**：
    *   **Input Min**：填写上一步记录的 `min` 值（如 `120.45`）。
    *   **Input Max**：填写上一步记录的 `max` 值（如 `840.85`）。
    *   **Output Min**：填 `0.0`。
    *   **Output Max**：填 `1.0`。
    *   *此时，地形高程数据已被压缩在 [0, 1] 区间，为导出做好了准备。*

#### 6. `Heightfield Output` [NEW]
*   **作用**：将归一化后的数据作为 16-bit 灰度图写入本地磁盘。
*   **核心配置**：
    *   **Output Picture**：选择 `.png` 或 `.raw`（即 r16）。格式示例：`E:/VerticalSlice/Heightmaps/Terrain_1k_Resample1009.png`。
    *   **Data Type**：必须更改为 **16-bit Unsigned Integer (U16)**。
    *   **Height Layer**：选择 `height`。
    *   **Remap**：由于已在 `hf_remap` 中进行了精确归一化，此处 Remap Range 选择 **0 to 1**，不可再启用 Auto-remap，否则会造成高程数据偏移。

---

## 5. Houdini Apprentice 限制与验证替代方案

如果本地使用的是 Houdini Apprentice (非商业免费版)，将受到以下技术约束：

### 5.1 Apprentice 的两项核心约束
1.  **图片导出受限**：`Heightfield Output` 输出的图片分辨率上限为 **$1280 \times 720$**，无法导出完整的 $2017 \times 2017$ 的 16-bit 高度图。
2.  **HDA 加载受限**：生成的 `.hdanc` 文件无法在商业/Indie 版的 UE 插件中载入运行。

### 5.2 Apprentice 下的 1~2 km² 地形验证替代方案：Mesh 路线
为了在 Apprentice 许可下仍然能够验证 1~2 km² 的地形资产，我们可以将 Heightfield 转换为网格体（StaticMesh）并激活 Nanite，这也是项目研究中“补充负空间”和“纯 Nanite Mesh 地形评估”的重要闭环步骤。

#### 替代操作步骤（Houdini 侧）：
1.  在 `hf_remap` 节点下，连接 **`Convert HeightField`** 节点。
    *   **Convert to**：选择 **Polygon Mesh**。
    *   **Division Method**：选择 **By Size**（Size 设为 1，即保留 1 米一个顶点的精度）。
2.  连接 **`PolyReduce`** 节点（可选，如果需要轻量化，但由于开启 Nanite，可以直接跳过此步以保留高频细节）。
3.  由于 Apprentice 限制 FBX 输出，连接 **`File`** 节点或右键 `Convert` 节点选择 **Save Geometry**。
    *   将文件路径设置为绝对路径，以 `.obj` 后缀结尾：`E:/VerticalSlice/Meshes/Terrain_Slice.obj`。
    *   Houdini Apprentice 允许自由导出 OBJ 格式。
4.  在虚幻引擎中，利用 **Interchange** 直接导入该 OBJ 网格体，并在导入设置中勾选 **Enable Nanite**。

---

## 6. 第四阶段：导入 Unreal Engine 5.8.0 与数学对齐

本阶段的目标是将导出的 16-bit PNG 高度图导入虚幻引擎，并**精确还原**它在真实世界中的海拔坐标。

### 6.1 精确对齐的数学原理

虚幻引擎的 Landscape 系统使用 16-bit 无符号整数（`0` 到 `65535`）来映射高程：
*   当 Z 轴缩放（Z Scale）为 `100` 时，地形的总垂直范围是 **512 米**（范围为 $-256\text{m}$ 到 $+256\text{m}$，即 $-25600\text{cm}$ 到 $+25600\text{cm}$）。
*   灰度值 `32768`（中灰色）对应 Unreal 的 $Z = 0$ 坐标。

#### 1. Z Scale 计算公式
为了让归一化到 `[0, 1]` 的 16-bit 灰度图在虚幻中被拉伸回真实的高度差，Z 轴缩放比例的公式为：

$$\text{Z\_Scale} = \frac{\text{Total\_Height\_in\_Meters}}{512} \times 100$$

或者，当以厘米为单位进行直接乘积时：

$$\text{Z\_Scale} = \text{Total\_Height\_in\_Centimeters} \times \frac{1}{51200} \approx \text{Total\_Height\_in\_Centimeters} \times 0.001953125$$

*   **以我们的预估数据为例**：
    *   总高差 (Total Height) = $720.40$ 米。
    *   $$\text{Z\_Scale} = \frac{720.40}{512} \times 100 = 140.703125$$
    *   在导入地形时，Z 轴缩放比例必须填写：**`140.703125`**。

#### 2. Z Location (Z轴位置偏移) 计算公式
默认情况下，导入高度图后，虚幻引擎会将高度图的中点高度（灰度 `32768`）对齐到 $Z = 0$ 关卡。
在真实世界中，中点对应的绝对海拔高度是：

$$\text{Mid\_Height} = \frac{\text{Min\_Height} + \text{Max\_Height}}{2}$$

*   **以我们的数据为例**：
    *   $$\text{Mid\_Height} = \frac{120.45 + 840.85}{2} = 480.65\text{ 米}$$
*   这意味着，如果不做偏移，Unreal 坐标 $Z = 0$ 处的海拔对应的是 $480.65$ 米。
*   为了实现**绝对海拔对齐**（即 Unreal $Z = 0$ 对应海平面 $0$ 米），我们需要将整个 Landscape 演员（Actor）在 Unreal 世界中向上移动 $\text{Mid\_Height}$。
*   在 Unreal 中，Z 轴位置（Location Z）以厘米为单位，因此：

$$\text{Landscape\_Z\_Location} = \text{Mid\_Height} \times 100$$

*   **以我们的数据为例**：
    *   $$\text{Landscape\_Z\_Location} = 480.65 \times 100 = 48065.0$$
    *   在 Landscape 的 Location 属性中，Z 值填入：**`48065.0`**（单位：厘米）。

---

## 7. 第五阶段：虚幻编辑器 (UE Editor) 导入操作步骤

### 7.1 步骤 1：打开 Landscape 工具与载入高度图
1.  启动 Unreal Editor，打开您的预研关卡项目。
2.  在左上角模式切换菜单选择 **Landscape (地形)** 模式（快捷键 **`Shift + 2`**）。
3.  在 Landscape 管理面板中，选择 **Import from File (从文件导入)**。
4.  点击 **Heightmap File** 后方的 `...` 按钮，选择从 Houdini 导出的 16-bit 灰度 PNG 文件（如 `Terrain_1k_Resample1009.png`）。

### 7.2 步骤 2：应用计算出的缩放与位置
1.  **Scale (缩放)**：
    *   保持 X 和 Y 为 `100.0`（代表 1 米/像素）。
    *   将 Z 修改为我们计算出的精确缩放值（例如填入 **`140.703125`**）。
2.  **Location (位置)**：
    *   保持 X 和 Y 为 `0.0`（或根据项目坐标基准确定）。
    *   将 Z 修改为我们计算出的偏移厘米值（例如填入 **`48065.0`**）。
3.  **Material (材质)**：
    *   分配预先准备好的地形基础材质（建议使用支持 RVT 混合的材质）。

### 7.3 步骤 3：组件切片规格复核
在面板底部的 **Import** 之前，检查规格是否与 Houdini 预期一致：
*   例如，如果高度图是 `1009 x 1009`，Unreal 的参数应显示为：
    *   **Section Size**：63x63 Quads
    *   **Sections Per Component**：2x2 Sections (每个组件包含 126x126 像素)
    *   **Number of Components**：8x8 Components
    *   **Overall Resolution**：1009x1009
*   确认无误后，点击底部的 **Import** 按钮。

---

## 8. 第六阶段：垂直切片性能验证与下一步清单

导入完成后，为了实现项目指引中的“性能闭环反馈”，请按以下清单进行配置和指标检测：

### 8.1 性能与渲染配置闭环
*   [ ] **激活 Nanite 地形**：在 Landscape 属性面板中，勾选 **Enable Nanite**。这会生成对应的 Nanite 地形流送代理，避免传统的 CPU 几何体 LOD 开销。
*   [ ] **配置 RVT (Runtime Virtual Texture)**：创建 RVT 资产并在关卡中放置 `Runtime Virtual Texture Volume`。在 Landscape 材质中绑定 RVT 输出，用以在渲染时扁平化材质混合开销，维持 60 FPS 基线帧率预算。
*   [ ] **启用 World Partition 分区**：
    *   确认关卡支持 World Partition。
    *   为 $1\text{km}$ 垂直切片划分网格（例如 Grid Cell Size 设为 256m 或 512m），便于运行时分块流送（Streaming）。
*   [ ] **PSO 预热与 Trace 分析**：
    *   执行穿越流送测试，记录 `%APPDATA%\Unreal Engine\LocalBuildLogs\` 日志。
    *   使用 `CsvProfiler` 抓取流送过程中的每一帧耗时，重点核对是否存在因未完全 PSO Precache 而导致的首帧卡顿（hitch）。
