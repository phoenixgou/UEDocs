# Houdini LiDAR 真实地形 → Unreal Engine 完整工作流

> **视频来源**：[Import Real Landscapes into Unreal in under 30 minutes using Houdini and real world DTM data](https://www.youtube.com/watch?v=9v_7OmgNzIA)  
> **频道**：UNDINI (@UndiniTuts)  
> **时长**：34 分 39 秒  
> **整理时间**：2026-07-01  

---

## 一、核心思路

与其花几个月时间用程序化工具（Houdini Erode / World Machine / World Creator 等）去"仿造"自然地形，不如**直接使用真实世界的 LiDAR 数字地形模型（DTM）数据**。

| 方法 | 优势 | 劣势 |
|------|------|------|
| 程序化生成 | 完全自定义，可控 | 难以还原数千年自然侵蚀历史；山脉容易雷同 |
| **真实 DTM 数据（本方案）** | 1:1 还原地形，天然具有独特性格；可像"实景勘探"一样 Location Scout | 受数据覆盖范围限制 |

---

## 二、整体流程

```
Google Maps 选址
    ↓
从公开 LiDAR 数据库下载 DTM 瓦片（GeoTIFF）
    ↓
Houdini：Heightfield File 导入 → Heightfield Project 重采样 → Heightfield Remap 归一化
    ↓
Houdini：Heightfield Output 导出 16-bit PNG 高度图
    ↓
Unreal Engine：Landscape Import（按公式换算 XYZ Scale）
```

---

## 三、Unreal 关卡模板设置

| 要点 | 说明 |
|------|------|
| 推荐模板 | **Time of Day** 模板（UE 4.25+ / UE5 均有等效模板） |
| 已内置功能 | 体积云（Volumetric Clouds）、天空大气（Sky Atmosphere，支持瑞利散射）、可拖动太阳方向（`Ctrl + L`） |
| 作用 | 自动提供"远处物体偏蓝"等大气透视效果，减轻艺术家单独模拟尺度感的负担 |

---

## 四、LiDAR DTM 数据下载

### 4.1 公开数据源

| 地区 | 链接 / 搜索关键词 |
|------|-----------------|
| 英国（England，不含 Wales/Scotland）| [UK LiDAR Composite DTM](https://data.gov.uk/dataset/3fc40781-...) → 搜索 `uk lidar dtm` |
| 挪威 | `https://hoydedata.no/LaserInnsyn/` |
| 美国 | 搜索 `us open source height map lidar data` → USGS 数据库 |

> **注意**：如果使用非英国数据集，文件体积可能超过 Houdini 默认的 Max File Size 限制。  
> 解决方案参考作者的另一期短视频：[How To Import Larger Lidar Files to Houdini](https://www.youtube.com/watch?v=itCbS70wdzs)

### 4.2 推荐规格

- **分辨率**：1 米精度（1 meter per pixel）
- **数据类型**：DTM（Digital Terrain Model，纯净地表，去除建筑/植被）
  - ❌ 不要用 DSM（Digital Surface Model，包含建筑物和树冠）
- **高程精度**：32-bit GeoTIFF（HDR 高程，超出 \[0,1\] 范围，需在 Houdini 中重映射）
- **年份**：优先选最新（英国有 2020 年版本）

### 4.3 操作步骤（以英国为例）

1. 在 [UK DEFRA LiDAR Portal](https://data.gov.uk/dataset/3fc40781-...) 页面进入 **Survey Download** 交互地图
2. 先在 **Google Maps（卫星+3D 视图）** 中找到感兴趣的地形区域，记下附近地名或坐标
3. 在 Survey Download 地图中导航到同一位置，切换图层为 **DTM 2020 / 1m resolution**
4. 用多边形/矩形框选区域 → `Get Available Tiles`
5. 勾选目标瓦片（如 `NT81SE`），下载并解压

> 每张瓦片约 5000 × 5000 m，对应单张 ~5K 分辨率的 32-bit GeoTIFF。

---

## 五、Houdini 节点流详解

> 假设已下载并解压 `.tif` 文件。  
> 所用 Houdini 版本：Indie（学习版 Apprentice **不可用**——导出上限为 512×512，无法满足需求）

### 5.1 界面简化

关闭 Houdini 中不必要的面板，只保留：
- 右上：**参数面板（Parameters）**
- 右下：**网络视图（Network View）**
- 左侧：**3D 视口（Viewport）**

### 5.2 完整节点链

```
[Geometry 容器]
  └─ Heightfield File         ← 导入原始 DTM（.tif）
      └─ Heightfield           ← 空白目标网格（5000×5000m，4033 分辨率）
          └─ Heightfield Project  ← 将 5k 原始数据重采样到 4033×4033
              └─ Heightfield Remap ← 归一化高程到 [0,1]，记录 Min/Max
                  └─ Heightfield Output ← 导出 16-bit PNG
```

### 5.3 各节点关键参数

#### `Heightfield File`
| 参数 | 值 |
|------|----|
| File | 拖入下载的 `.tif` 文件 |

#### `Heightfield`（空目标网格）
| 参数 | 值 |
|------|----|
| Size X/Y | `5000` × `5000`（与真实范围匹配） |
| Division Mode | `By Axis` |
| Grid Resolution | `4033`（UE 推荐的 Landscape 规格之一） |

> 按 `I` 键可在视口中查看当前节点的分辨率信息。

#### `Heightfield Project`
| 输入口颜色 | 连接内容 |
|------------|---------|
| 🟢 绿色（左） | Heightfield（空目标网格）|
| ⬜ 白色（右） | Heightfield File（原始 DTM）|

#### `Heightfield Remap`
| 操作 | 说明 |
|------|------|
| 点击 `Compute Range` | 自动检测原始数据的高程范围 |
| 读取并记录 **Input Min** | 该区域最低海拔（如：**71 m**）|
| 读取并记录 **Input Max** | 该区域最高海拔（如：**851 m**）|
| 设置 Output Min → `0` | 将最低点映射为 0 |
| 设置 Output Max → `1` | 将最高点映射为 1 |

> ⚠️ **必须记录 Min 和 Max！** 后续在 UE 中计算 Z 轴缩放比例需要用到这两个值。

#### `Heightfield Output`
| 参数 | 值 |
|------|----|
| File | `$HIP/exports/lidar_height.png`（`$HIP` 自动指向项目保存目录）|
| Channel | Single（单通道）|
| Type | **16-bit fixed**（UE Landscape 标准格式）|
| Image Channel（Red） | `height` |
| 操作 | 点击 **Save to Disk** |

> ⚠️ 如果导出图像全白或全黑，说明 Remap 节点的 Output Min/Max 设置有误，请重新检查。

---

## 六、Unreal Engine 导入与比例换算

### 6.1 进入 Landscape 导入模式

```
快捷键 Shift+2  →  Landscape 模式  →  Import from File
```

选择刚才导出的 `.png` 文件，**先不要点 Import**，需要先填写正确的 Scale 值。

### 6.2 XY 轴比例换算公式

虚幻引擎默认 **1 像素 = 1 米 = Scale 100**。  
但我们的高度图是 4033px，覆盖的真实范围是 5000m：

$$\text{XY Scale} = \frac{\text{真实世界跨度 (m)}}{\text{高程图分辨率 (px)}} \times 100$$

$$\text{XY Scale} = \frac{5000}{4033} \times 100 \approx \mathbf{123.98}$$

### 6.3 Z 轴比例换算公式

UE 的 Landscape 默认 Z Scale = 100 时，高程范围为 **−256m ~ +256m = 共 512m 落差**。

1. 计算真实地形落差：
$$\text{高度落差} = \text{Max} - \text{Min} = 851 - 71 = 780 \text{ m}$$

2. 换算为 UE Z Scale：
$$\text{Z Scale} = \frac{\text{真实高度落差 (m)}}{512} \times 100 \approx \mathbf{152.34}$$

### 6.4 通用换算公式

| 轴 | 公式 |
|----|------|
| X、Y Scale | $\frac{\text{真实范围(m)}}{\text{高程图分辨率(px)}} \times 100$ |
| Z Scale | $\frac{(\text{DTM Max} - \text{DTM Min})}{512} \times 100$ |

### 6.5 填入并导入

在 UE Landscape Import 面板中：
- Scale X = `123.98`
- Scale Y = `123.98`  
- Scale Z = `152.34`

点击 **Import**，按 `Shift+1` 退出编辑模式即可看到 1:1 还原的真实地形。

---

## 七、重要避坑建议

| 建议 | 说明 |
|------|------|
| ❌ 不用 World Composition | 配置繁琐，容易引入边界断裂和性能问题，初学者尤其要避免 |
| ❌ 不用 8K Landscape | 8K + 多层 Layer Blend 会让编辑器和 Cook 极度缓慢，4K 级别足够 |
| ✅ 坚持单张大地形 | 即使是大型环境，单张 4K Landscape 也能满足绝大多数需求 |
| ✅ 用 Houdini Indie 授权 | Apprentice（免费学习版）导出限制 512×512，无法用于本流程 |
| ✅ 优先使用 DTM 而非 DSM | DSM 包含建筑/植被噪点，DTM 才是干净的地貌高程 |

---

## 八、Brushify 推荐（扩展资源）

视频中提到了 **Joe Garth 的 Brushify** 插件系列：
- 已将真实 LiDAR 数据转换为可在 UE 编辑器中**实时笔刷绘制**的山脉笔刷资产。
- 可在虚幻商城（Unreal Marketplace）找到大量相关内容。
- 适合需要在现有关卡中快速绘制山脉细节的工作流。

---

## 九、视频章节速查

| 时间码 | 内容 |
|--------|------|
| 00:00 | 介绍目标与方法论（真实数据 vs 程序化） |
| 02:15 | 公开 LiDAR 数据源介绍与搜索 |
| 04:15 | Google Maps 地形勘探 + 个人关于程序化地形局限性的长篇论述 |
| 07:50 | 论述结束 |
| 09:40 | 在 UK LiDAR 地图中框选并下载数据瓦片 |
| 12:26 | Houdini 获取（Indie vs Apprentice 授权说明） |
| 14:00 | 进入 Houdini 操作 |
| 15:05 | Heightfield File 节点导入 DTM |
| 17:56 | Heightfield 分辨率设置（4033×4033）|
| 21:53 | Heightfield Remap 归一化 → Heightfield Output 导出 16-bit PNG |
| 30:04 | 导入 Unreal Engine 并计算 XYZ 缩放比例 |
| 34:22 | 结束 |

---

## 十、参考链接

- 英国 LiDAR 数据库：https://data.gov.uk/dataset/3fc40781-...
- 挪威 LiDAR：https://hoydedata.no/LaserInnsyn/
- 美国 USGS LiDAR：https://prd-tnm.s3.amazonaws.com/Lida...
- Houdini 官方获取：https://www.sidefx.com/buy/
- UE Landscape 技术指南：https://docs.unrealengine.com/（搜索 Landscape Technical Guide）
- Brushify 频道：https://www.youtube.com/@Brushify

---

## 十一、QGIS 地理信息预处理（可选但推荐）

> **适用场景**：选区横跨多个瓦片、或使用非英国数据源（挪威/美国），原始 GeoTIFF 坐标系为地理坐标（经纬度/Degree 单位）时，必须先完成重投影，否则 Houdini 导入后水平尺度和垂直尺度会产生严重形变。

### 11.1 为何需要重投影

原始 DTM 通常以 **WGS84 经纬度（Degree）** 坐标系存储。Houdini 和 UE 均以**米（Meter）**作为内部单位。若直接导入经纬度坐标系的 TIFF，会导致：
- 水平方向以"度"为步长，1° ≈ 111 km，几何形状完全失真
- 垂直（高程）仍是米，产生极端的纵横比错误

### 11.2 操作步骤

#### 步骤 1：合并多瓦片（Raster Merge）
若框选区域跨越多张 DTM 瓦片：

```
Raster（栅格）→ Miscellaneous（杂项）→ Merge（合并）
```

勾选所有待拼接瓦片，输出为临时图层。

#### 步骤 2：重投影为 UTM 投影坐标系（Warp/Reproject）

```
Raster（栅格）→ Projections（投影）→ Warp (Reproject)（变形/重投影）
```

| 参数 | 说明 |
|------|------|
| Input layer | 合并后的图层 |
| Target CRS | 根据地理位置选对应 UTM 带（单位必须为**米**）：挪威 → `WGS 84 / UTM zone 32N`；英国 → `OSGB36 / British National Grid`；美国 → 对应 UTM 区带 |
| Resampling method | Bilinear（双线性）|

#### 步骤 3：按范围裁剪（Clip by Extent）

```
Raster（栅格）→ Extraction（提取）→ Clip Raster by Extent（按范围裁剪）
```

在画布上框选精确的矩形区域（建议比目标地形略大 5%，在 Houdini 中再精裁），另存为 **GeoTIFF（32-bit Float）**。

---

## 十二、Houdini Apprentice（免费版）许可限制与替代路线

> **许可限制摘要**：Apprentice 的 `Heightfield Output` 图片导出分辨率上限为 **1280×720**，无法满足 4033×4033 的高度图需求。此外 Apprentice 不支持 FBX / Alembic 格式导出。

### 12.1 替代方案：将 Heightfield 转为 OBJ 网格

此路线虽无法走 Landscape 系统，但能完成 **Nanite StaticMesh 地形**的性能评估（与预研中"补充负空间"路线一致）。

**Houdini 侧节点调整**（替换 `Heightfield Output`）：

```
Heightfield Remap
    └─ Convert HeightField       ← 转为多边形网格
        └─ [可选] PolyReduce     ← Nanite 不需要减面，可跳过
            └─ File（Save to Disk）← 输出 .obj
```

| 节点 | 关键设置 |
|------|---------|
| Convert HeightField | Convert to: **Polygon Mesh**；Division Method: **By Size**（Size = 1，保持 1 米/顶点） |
| File | 路径设为 `.obj` 后缀，Houdini Apprentice 允许自由导出 OBJ |

**UE Editor 侧导入**：

1. 用 **Interchange** 或旧版 StaticMesh Import 导入 `.obj`
2. 在导入设置中勾选 **Build Nanite**
3. 在材质槽中指定基础地形材质

---

## 十三、UE 5.8.0 特定集成指南

> **本节基于本地源码 `E:\UEWS\5.8.0` 锚点**，描述 Landscape 导入完成后，在 UE 5.8 开放世界垂直切片中需要额外完成的配置步骤。

### 13.1 激活 Nanite 地形（LandscapeNaniteComponent）

导入 Landscape 后，在 **Details 面板**（选中 Landscape Actor）中：

| 属性 | 操作 |
|------|------|
| **Enable Nanite** | 勾选 ✅ |

勾选后，引擎会生成对应的 `LandscapeNaniteComponent`（源码锚点：`E:\UEWS\5.8.0\Engine\Source\Runtime\Landscape\Classes\LandscapeNaniteComponent.h`），以 Nanite 简化代理替换传统 CPU LOD，彻底消除地形 DrawCall 随视角的线性增长。

> ⚠️ **双轨风险**：Nanite 可视表达与碰撞/导航网格数据不共享，必须分别验证"看得到走得通"和"走得通看得到"两侧的一致性。

### 13.2 配置 Runtime Virtual Texture（RVT）

RVT 是 Landscape + Nanite 地形获得正确材质混合的关键：

1. 在 Content Browser 中新建 **Runtime Virtual Texture** 资产
2. 在关卡中放置 **Runtime Virtual Texture Volume**，将 RVT 资产绑定其上
3. 在 Landscape 材质中使用 `Runtime Virtual Texture Sample` 节点读取 RVT 输出，实现基于坡度/高度的自动材质分层

### 13.3 启用 World Partition 分区

| 操作 | 说明 |
|------|------|
| 确认关卡为 World Partition 关卡 | 新建关卡时选择 **Open World** 模板，或在现有关卡中通过 **World Settings → World Partition** 开启 |
| Grid Cell Size | 垂直切片建议 **256m 或 512m**，与流送源（StreamingSource）配合 |
| Landscape 与 WP | 导入的 Landscape Actor 会自动注册到 World Partition，可在 **World Partition Editor** 中确认其所在 Cell |

源码锚点参考：
- `E:\UEWS\5.8.0\Engine\Source\Runtime\Engine\Public\WorldPartition\WorldPartitionStreamingSource.h`
- `E:\UEWS\5.8.0\Engine\Source\Runtime\Engine\Public\WorldPartition\WorldPartitionStreamingPolicy.h`

### 13.4 PSO Precache 协同验证

Landscape 材质较复杂时，若 PSO 未完全预缓存，Teleport 或快速加载后会出现首帧编译尖峰（hitch）。建议：

1. 在 Project Settings 中确认 **`r.PSOPrecache.Enable`** 已开启
2. 执行穿越测试（步行 → 载具 → 极速穿越）时，同步开启 CSV Profiler 采集帧时间：
   ```
   csvprofile start
   （执行穿越路线）
   csvprofile stop
   ```
3. 在输出的 CSV 文件中（路径见 `%APPDATA%\Unreal Engine\LocalBuildLogs\`）检查是否存在 `PSOPrecache` 相关尖峰帧

---

## 十四、1~2 km² 垂直切片性能验收清单

以下为本次预研首轮三条主线（单机场景、穿越流送、战斗压力）的验收核查点：

### 地形导入与渲染
- [ ] Landscape 导入后，**XY/Z Scale 数值经公式验证**，与真实地理尺度 1:1 匹配
- [ ] Nanite 已在 Landscape Details 中启用（`Enable Nanite = true`）
- [ ] 视口中切换 **Nanite 可视化模式**（`showflag.NaniteVisualization 1`），确认地形由 Nanite 处理
- [ ] RVT Volume 已覆盖整个地形范围，材质混合无穿插/黑斑

### 碰撞与导航（双轨验证）
- [ ] 在地形上方执行 `P` 键显示碰撞体，确认碰撞体与视觉表达贴合
- [ ] 执行 NavMesh 构建（`BuildNavigation`），确认 NavMesh 能覆盖预设穿越路线上所有可行走区域

### 流送与 PSO
- [ ] World Partition Editor 中确认 Landscape 已正确分区
- [ ] 执行 Teleport 测试，检查 `Saved\Logs\` 中是否存在 PSO FirstFrame 编译记录
- [ ] 执行 30 分钟 Soak 测试，采集 Nanite Streaming Pool 占用趋势（`Nanite.StreamingPoolSize`）

### 工具采集锚点
| 工具 | 命令 / 路径 |
|------|------------|
| GPU Profiler Trace | `stat gpu` / UnrealInsights |
| CSV Profiler | `csvprofile start/stop` |
| Trace Auxiliary | UnrealInsights 的 Memory / CPU / GPU 通道 |
