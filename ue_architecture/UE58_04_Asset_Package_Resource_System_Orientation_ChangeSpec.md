# UE5.8 资源与资产系统源码地图 Change Spec

> 本文档面向 thomas。这是"第 4/5 个文档任务：Unreal Engine 5.8 资源与资产系统源码地图"的变更规格（change spec），用于约束主文档 [`UE58_04_Asset_Package_Resource_System_Orientation.md`](<D:/UE/Docs/UE58_04_Asset_Package_Resource_System_Orientation.md>) 的产出边界与验收标准。
>
> 本文档遵循阿卡姆剃刀：只描述本次任务必需的范围与判据，不展开资产序列化的具体实现细节。
>
> **证据约定**：标 `【事实】` 的内容由本机目录/文件名/`.Build.cs`/轻量 grep（含 `file:line`）直接验证；标 `【推理】` 的内容是 **逻辑分析推理(无事实依据)**，未通读实现，需后续读代码确认。

---

## 1. 目标（Goals）

1. 为 thomas 建立一张 **Unreal Engine 5.8 资源/资产系统源码地图**，目标是"看懂 UObject / Package / AssetRegistry / 加载引用 / Cook / DDC 的地图、在资产体系里不迷路"，**不**做任何一条资产序列化实现的逐行精读。
2. 讲清七个核心边界对象的 **落点与职责**：`UObject`、`UPackage`、`IAssetRegistry` / `FAssetData`、`UAssetManager`、`FSoftObjectPath`、`FStreamableManager`、Cook（`UCookOnTheFlyServer`）、DerivedDataCache（DDC）。
3. 讲清一条主链路：**磁盘资产文件 → Package → UObject → AssetRegistry 索引 → Soft/Hard Reference → Load/Stream 加载 → Cook/DDC 产出**。
4. 说明本系统与 **大世界（WorldPartition）、流送、插件、编辑器框架** 的衔接点。
5. 全程区分 **事实依据** 与 **逻辑分析推理(无事实依据)**。

## 2. 范围（Scope）

只读调查，且只允许触达以下目录级、有限深度的结构与少量代表性文件：

- `D:\UE\5.8.0r\Engine\Source\Runtime\CoreUObject`（模块根 + `Public\UObject` / `Public\AssetRegistry` / `Public\Cooker` 子域 + `CoreUObject.Build.cs`）
- `D:\UE\5.8.0r\Engine\Source\Runtime\AssetRegistry`（模块根 + `Public\AssetRegistry` + `AssetRegistry.Build.cs`）
- `D:\UE\5.8.0r\Engine\Source\Runtime\Engine\Classes\Engine`（`AssetManager.h`、`StreamableManager.h`、`AssetManagerTypes.h`、`AssetManagerSettings.h`、`Engine.h` 的轻量 grep）
- `D:\UE\5.8.0r\Engine\Source\Developer\DerivedDataCache`（模块根 + `Public` 头清单，仅定位）
- `D:\UE\5.8.0r\Engine\Source\Developer\AssetTools`（仅定位 `.Build.cs`）
- `D:\UE\5.8.0r\Engine\Source\Editor\UnrealEd\Classes\CookOnTheSide\CookOnTheFlyServer.h` 与 `...\Private\CookOnTheFlyServer.cpp`（仅定位类声明与代表性函数行号）
- `D:\UE\5.8.0r\Engine\Source\Runtime\CookOnTheFly`、`D:\UE\5.8.0r\Engine\Source\Developer\CookOnTheFlyNetServer`（仅定位）

`Generated/`、`Intermediate/`、`Binaries/`、`DerivedDataCache/`（磁盘缓存数据目录）、`Saved/` 等生成产物，以及 `.uasset` 二进制内容 **只做排除说明，不进入内容**。

## 3. 非目标（Non-Goals）

1. 不做 Unreal Engine 全量源码扫描，不逐行通读任何 `.cpp/.h` 完整实现，不精读资产序列化格式（`PackageFileSummary`、`FLinkerLoad` 内部状态机仅做"它在哪"的定位）。
2. 不深入 Nanite / WorldPartition / HLOD / 渲染 / 动画 任何一条实现链路（已由前序文档覆盖，本文只做衔接指引）。
3. 不读取凭据、会话、个人配置、压缩包（`D:\UE\UnrealEngine-5.8.0-release.zip`）、生成产物（`Binaries`、`Intermediate`、`DerivedDataCache`、`Saved`），不读取任何 `.uasset` 二进制内容。
4. 不修改 `D:\UE\5.8.0r` 下任何源码、项目文件、生成产物或压缩包，不触发构建。
5. 不覆盖已有文档 `D:\UE\Docs\UE58_Source_Hierarchy_Orientation*.md`、`D:\UE\Docs\UE58_Nanite_WorldPartition_HLOD_*.md`、`D:\UE\Docs\UE58_Rendering_Animation_DeepDive*.md`。

## 4. 调查路线（Investigation Route）

| 步骤 | 动作 | 范围 | 证据形态 |
| --- | --- | --- | --- |
| S1 | 列举 CoreUObject 模块结构与 `Public` 子域 | `Runtime\CoreUObject` | 目录条目 + `CoreUObject.Build.cs` 依赖行 |
| S2 | 列举 AssetRegistry 模块结构 | `Runtime\AssetRegistry` | 目录条目 + `AssetRegistry.Build.cs` 依赖行 |
| S3 | 定位核心类型声明 | `Object.h` / `Package.h` / `SoftObjectPath.h` / `AssetData.h` / `IAssetRegistry.h` | `class/struct` 声明 `file:line` |
| S4 | 定位 AssetManager / StreamableManager 入口 | `Runtime\Engine\Classes\Engine` | `UAssetManager`、`FStreamableManager`、`GetStreamableManager`、`GetAssetRegistry`、`LoadPrimaryAssets` 行号 |
| S5 | 定位加载机制 | `LinkerLoad.h` / `Linker.h` / `Package.h` | `FLinkerLoad`、`CreateLinker`、`GetPackageLinker`、`UPackage::GetLinker` 行号 |
| S6 | 定位 DDC 模块 | `Developer\DerivedDataCache\Public` | 接口头清单（`DerivedDataCacheInterface.h` 等） |
| S7 | 定位 Cook 入口 | `Editor\UnrealEd` Cook 头/实现 | `UCookOnTheFlyServer`、`ECookMode`、`StartCookByTheBook`、`BlockOnAssetRegistry` 行号 |

## 5. 产物（Deliverables）

1. 本 change spec：`D:\UE\Docs\UE58_04_Asset_Package_Resource_System_Orientation_ChangeSpec.md`。
2. 主文档：`D:\UE\Docs\UE58_04_Asset_Package_Resource_System_Orientation.md`，必须包含：
   - "thomas 先读这段"：资源/资产地图如何帮助不迷路
   - ASCII 总览图：磁盘资产 → Package → UObject → AssetRegistry → Soft/Hard Reference → Load/Stream → Cook/DDC
   - Mermaid 1：资产发现与注册图（AssetRegistry 扫描/索引）
   - Mermaid 2：软引用加载图（FSoftObjectPath → StreamableManager → 异步加载）
   - Mermaid 3：Cook / DDC 高层关系图
   - 七个边界对象的职责与落点：`UObject` / `UPackage` / `AssetRegistry`+`FAssetData` / `UAssetManager` / `FSoftObjectPath` / `FStreamableManager` / `Cook` / `DDC`
   - 与大世界、流送、插件、编辑器框架的衔接
   - 与前三份文档的衔接
   - 阿卡姆剃刀检查
   - 局限性与潜在风险提示

## 6. 验收标准（Acceptance Criteria）

1. 主文档所有 Windows 路径均为完整绝对路径；可点击链接目标为正斜杠完整绝对路径并置于尖括号 `<...>` 中。
2. 主文档含 **至少 3 个 mermaid 图**：资产发现与注册图、软引用加载图、Cook/DDC 高层关系图。
3. 主文档含 **一个 ASCII 总览图**，覆盖 磁盘资产 → Package → UObject → AssetRegistry → Soft/Hard Reference → Load/Stream → Cook/DDC 主链路。
4. 主文档为七个边界对象各给出 **完整绝对路径 + 至少一个 `file:line` 事实**。
5. 主文档逐处标注 **事实依据** 与 **逻辑分析推理(无事实依据)**；复杂描述主谓宾完整。
6. 主文档说明"机器绝对路径只是本机定位路径，工程内引用应使用 Unreal 路径 API（`FSoftObjectPath`/`FPackagePath`/`FPaths`）或模块相对包含路径，不得硬编码 `D:\UE\...`"。
7. 各份文档对同一路径/职责/模块边界/口径的描述不得互相矛盾（最后一步读取复核）。
8. 全程未读取范围外文件、未读取凭据/压缩包/生成产物/`.uasset` 二进制、未修改任何引擎源码，未覆盖已有六份文档。

## 7. 局限性与潜在风险提示

- 本研究只看目录名、文件名、`.Build.cs` 与少量声明/函数 grep 行，**未通读任何实现**；"加载时序、Cook 阶段流转、DDC 命中逻辑"等运行时行为多为 **逻辑分析推理(无事实依据)**，需后续读代码或读 Epic 文档验证。
- 类型职责描述部分基于类名、头文件名与目录结构推断，文件名与真实职责可能不完全一致。
- 文档中的绝对路径绑定本机 `D:\UE\5.8.0r` 布局，换机或换引擎版本即失效；这是为满足 thomas"完整绝对路径"硬性要求而做的取舍，与"代码/文档不硬编码绝对路径"的通用准则存在张力，已在主文档显式声明并给出工程内正确引用方式。
