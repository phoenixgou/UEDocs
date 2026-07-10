# UE5.8 插件机制源码地图 Change Spec

> 本文档面向 thomas。这是「Unreal Engine 5.8 插件机制源码地图（第 1/5 篇文档任务）」的变更规格（change spec），用于约束主文档 [`UE58_01_Plugin_Mechanism_Orientation.md`](<D:/UE/Docs/UE58_01_Plugin_Mechanism_Orientation.md>) 的产出边界与验收标准。
>
> 本文档遵循阿卡姆剃刀：只描述本次任务必需的范围与判据，不展开任何插件实现精读。
>
> **证据约定**：标 `【事实】` 的内容由本机目录/文件名/`.uplugin`/`.uproject`/`.Build.cs`/头文件 `file:line` 直接验证；标“逻辑分析推理(无事实依据)”的内容基于 UE 通用约定推断，未通读实现，需后续读代码确认。
>
> **路径表述约定**：正文与 ASCII 图中提及目录/文件一律用 **完整绝对路径**（Windows 原生反斜杠形式，反引号包裹）；mermaid 图节点为避免过长，使用 **以 `D:\UE\5.8.0r\Engine\Source` 或 `D:\UE\5.8.0r\Engine\Plugins` 为根的相对简写**（正斜杠风格），不代表磁盘真实分隔符。

---

## 1. 目标（Goals）

1. 为 thomas 建立一张 **Unreal Engine 5.8 插件机制源码地图**，目标是让他理解 **插件如何被发现、描述、加载、编译，以及如何区分 Runtime / Editor / Developer**，做到「读一个插件不迷路」，**不做实现精读**。
2. 解释 thomas 不熟悉的插件机制关键词：`.uplugin` / Plugin Descriptor（`FPluginDescriptor`）；Module Descriptor（`FModuleDescriptor`）；Module Type（`EHostType`：Runtime / Editor / Developer / Program 等）；Loading Phase（`ELoadingPhase`）；`.Build.cs`（`ModuleRules`）与模块的关系；插件类型（`EPluginType`：Engine / Project / Enterprise / External / Mod）；引擎插件与项目插件的区别；`.uproject` 如何启用插件。
3. 提供「拿到一个插件，怎么定位它的 descriptor、模块、`.Build.cs`、类型与加载时机」的读插件路径。
4. 全程区分 **事实依据** 与 **逻辑分析推理(无事实依据)**。

## 2. 范围（Scope）

只读调查，且只允许触达以下目录级、有限深度的结构与少量代表性文件：

- `D:\UE\5.8.0r\Engine\Plugins`（顶层分类条目，有限深度；分类目录下取代表性子目录名）。
- `D:\UE\5.8.0r\Engine\Source\Runtime\Projects`（插件机制的运行时核心模块：`Public\PluginDescriptor.h`、`Public\ModuleDescriptor.h`、`Public\PluginReferenceDescriptor.h`、`Public\ProjectDescriptor.h`、`Public\PluginManifest.h`、`Public\Interfaces\IPluginManager.h`、`Private\PluginManager.cpp`）。
- `D:\UE\5.8.0r\Engine\Source\Programs\UnrealBuildTool`（构建期插件/模块规则相关文件，仅定位：`Configuration\Descriptors\PluginDescriptor.cs`、`Configuration\Descriptors\ModuleDescriptor.cs`、`Configuration\UEBuildPlugin.cs`、`Configuration\Rules\ModuleRules.cs`、`System\Plugins.cs`）。
- 代表性 `.uplugin`（3 个不同类型）：`D:\UE\5.8.0r\Engine\Plugins\Runtime\CableComponent\CableComponent.uplugin`、`D:\UE\5.8.0r\Engine\Plugins\Editor\PluginBrowser\PluginBrowser.uplugin`、`D:\UE\5.8.0r\Engine\Plugins\Animation\AnimationWarping\AnimationWarping.uplugin`。
- 代表性插件模块 `.Build.cs`：`D:\UE\5.8.0r\Engine\Plugins\Runtime\CableComponent\Source\CableComponent\CableComponent.Build.cs`。
- 项目侧插件启用片段（只读，不修改）：`D:\UE\AnimationSamples\AnimationSamples.uproject`、`D:\UE\ProjectTitan\ProjectTitan.uproject`。

`Generated\`、`Intermediate\`、`Binaries\`、`DerivedDataCache\`、`Saved\` 等生成产物目录与 `ThirdParty` 第三方库 **只做排除说明，不进入内容**；不读取凭据、会话、个人配置与压缩包内容。

## 3. 非目标（Non-Goals）

1. 不做 Unreal Engine 全量源码扫描，不逐行通读 `FPluginManager`/UBT 任何完整实现。
2. 不深入 GameFeatures、Verse、平台扩展插件（platform extension）等具体子系统的实现链路。
3. 不读取凭据、会话、个人配置、压缩包（`D:\UE\UnrealEngine-5.8.0-release.zip`）或生成产物（`Binaries`、`Intermediate`、`DerivedDataCache`、`Saved`）。
4. 不修改 `D:\UE\5.8.0r` 下任何源码、项目文件、生成产物或压缩包，不触发构建；不修改 `D:\UE\AnimationSamples` 与 `D:\UE\ProjectTitan` 任何文件。
5. 不覆盖已有文档 `D:\UE\Docs\UE58_Source_Hierarchy_Orientation.md`、`D:\UE\Docs\UE58_Rendering_Animation_DeepDive.md`、`D:\UE\Docs\UE58_Nanite_WorldPartition_HLOD_SourceMap.md` 及其各自 ChangeSpec。

## 4. 调查路线（Investigation Route）

| 步骤 | 动作 | 范围 | 证据形态 |
| --- | --- | --- | --- |
| S1 | 列举 `Engine\Plugins` 顶层分类 | `D:\UE\5.8.0r\Engine\Plugins` | 79 顶层条目 + 5 个代表性分类内子目录名 |
| S2 | 定位插件机制运行时核心模块 | `Runtime\Projects` | `Public\*.h` / `Private\PluginManager.cpp` 落点 |
| S3 | 取 Module Type / Loading Phase 枚举 | `ModuleDescriptor.h` | `EHostType`、`ELoadingPhase` 枚举与注释行号 |
| S4 | 取 `.uplugin` 字段集 | `PluginDescriptor.h` | `FPluginDescriptor` 字段行号 |
| S5 | 取插件类型/来源枚举与发现/加载函数 | `IPluginManager.h`、`PluginManager.cpp` | `EPluginType`、`EPluginLoadedFrom`、`DiscoverAllPlugins`/`ConfigureEnabledPlugins`/`LoadModulesForEnabledPlugins` 行号 |
| S6 | 取 3 个代表性 `.uplugin` 与 1 个 `.Build.cs` | 见第 2 节列表 | descriptor JSON 字段 + `ModuleRules` 子类 |
| S7 | 定位 UBT 构建期描述符与模块规则 | `UnrealBuildTool` | `Descriptors\*.cs`、`Rules\ModuleRules.cs` 落点 |
| S8 | 取项目侧插件启用片段 | 两个 `.uproject` | `Plugins`/`Modules`/`AdditionalPluginDirectories` 字段 |

## 5. 产物（Deliverables）

1. 本 change spec：`D:\UE\Docs\UE58_01_Plugin_Mechanism_Orientation_ChangeSpec.md`。
2. 主文档：`D:\UE\Docs\UE58_01_Plugin_Mechanism_Orientation.md`，必须包含：
   - 「thomas 先读这段：插件机制如何帮助不迷路」。
   - ASCII 总览图：`.uplugin` → Plugin Descriptor → Modules → `.Build.cs` → Runtime/Editor/Developer → Loading。
   - 至少 3 个 mermaid 图：插件发现与加载图、插件模块边界图、读一个插件的定位流程图。
   - 插件目录地图、`.uplugin` 字段解释、Module Type 解释、插件与模块/`.Build.cs` 关系、插件与项目 `.uproject` 关系、引擎插件/项目插件区别。
   - 与已有源码层级、渲染动画、大世界文档的衔接。
   - 阿卡姆剃刀检查、局限性与潜在风险提示。

## 6. 验收标准（Acceptance Criteria）

1. 主文档所有 Windows 路径均为完整绝对路径；可点击链接目标为正斜杠完整绝对路径并置于尖括号 `<...>` 中。
2. 主文档含 **至少 3 个 mermaid 图**：插件发现与加载图、插件模块边界图、读一个插件的定位流程图。
3. 主文档含 **一个 ASCII 总览图**，展示 `.uplugin → Plugin Descriptor → Modules → .Build.cs → Runtime/Editor/Developer → Loading` 链路。
4. 主文档解释第 1 节列出的全部关键词，且显式标明 Module Type（`EHostType`）与 Loading Phase（`ELoadingPhase`）的来源 `file:line`。
5. 主文档逐处标注 **事实依据** 与 **逻辑分析推理(无事实依据)**；复杂描述主谓宾完整。
6. 主文档说明「机器绝对路径只是本机定位路径，工程内引用插件/模块应使用 `IPluginManager`、`FPaths`、`.Build.cs` 模块名或 `.uplugin/.uproject` 配置，不得硬编码 `D:\UE\...` 绝对路径」。
7. 与已有四份文档对同一路径/职责/模块边界的描述不得互相矛盾（最后一步读取复核）。
8. 全程未读取范围外文件、未读取凭据/压缩包/生成产物、未修改任何引擎源码或项目文件，未覆盖已有文档。

## 7. 局限性与潜在风险提示

- 本研究只看目录名、`.uplugin`/`.uproject` JSON、`.Build.cs` 与若干头文件/`.cpp` 的 `file:line`，**未通读 `FPluginManager` 与 UBT 完整实现**；插件发现顺序、加载阶段调度、cook 期取舍等运行时行为多为 **逻辑分析推理(无事实依据)**，需后续读代码或读 Epic 文档验证。
- 任务建议调查的 `D:\UE\5.8.0r\Engine\Source\Developer\PluginBrowser` 与 `D:\UE\5.8.0r\Engine\Source\Developer\Plugins` 在本机 **不存在**【事实】；PluginBrowser 实际是插件 `D:\UE\5.8.0r\Engine\Plugins\Editor\PluginBrowser`，插件机制运行时核心在 `D:\UE\5.8.0r\Engine\Source\Runtime\Projects`。主文档以本机实测落点为准。
- 任务文字与 `D:\UE\ProjectTitan\AGENTS.md` 写的 `Titan.uproject` 与本机实际文件名 `D:\UE\ProjectTitan\ProjectTitan.uproject` 不一致【事实】；主文档以本机实际文件名为准。
- 文档中的绝对路径绑定本机 `D:\UE\5.8.0r` 布局，换机或换引擎版本即失效；这是为满足 thomas「完整绝对路径」硬性要求而做的取舍，与「代码/文档不硬编码绝对路径」的通用准则存在张力，已在主文档显式声明并给出工程内正确引用方式。
