# UE5.8 编辑器框架源码地图 Change Spec

> 本文档面向 thomas。这是"第 5/5 个文档任务：Unreal Engine 5.8 编辑器框架源码地图"的变更规格（change spec），用于约束主文档 [`UE58_05_Editor_Framework_Orientation.md`](<D:/UE/Docs/UE58_05_Editor_Framework_Orientation.md>) 的产出边界与验收标准。
>
> 本文档遵循阿卡姆剃刀：只描述本次任务必需的范围与判据，**不**展开任何编辑器 UI 的实现精读。
>
> **证据约定**：标 `【事实】` 的内容由本机目录条目、文件名、`.Build.cs` 内容或轻量 `rg` 命中（含 `file:line`）直接验证；标 `【推理】` 的内容是 **逻辑分析推理(无事实依据)**，基于 UE 通用约定推断，未通读实现，需后续读代码确认。

---

## 1. 目标（Goals）

1. 为 thomas 建立一张 **Unreal Engine 5.8 编辑器框架源码地图**，目标是"知道编辑器的每一块代码在哪、谁连谁、从一个按钮怎么找回源码"，而 **不** 做任何编辑器 UI 控件的实现精读。
2. 讲清七个编辑器关键角色及其源码落点：`Editor` 模块域、`Slate`/`SlateCore` UI 框架、`Commands`/`ToolMenus` 命令与菜单扩展、`Details`（属性面板定制）/`AssetEditor`（资产编辑器工具包）、`EditorSubsystem`、`Commandlet`。
3. 讲清 `Editor` / `Developer` / `Runtime` 三个源码域在"编辑器"语境下的边界，以及 `Slate` 与 `SlateCore` 的上下层关系。
4. 给出一条 **从"编辑器里的一个按钮/菜单项"反向定位到源码声明、实现、所属模块** 的可机械执行流程。
5. 全程区分 **事实依据** 与 **逻辑分析推理(无事实依据)**，并与已有的"源码层级整体认知"文档口径一致衔接。

## 2. 范围（Scope）

只读调查，且只允许触达以下目录级、有限深度的结构与少量代表性文件（均已在本次调查中实测）：

- `D:\UE\5.8.0r\Engine\Source\Editor`（顶层 145 条目）
- `D:\UE\5.8.0r\Engine\Source\Developer`（顶层 138 条目，定位 `ToolMenus`、`AssetTools`、`Settings` 等）
- `D:\UE\5.8.0r\Engine\Source\Runtime\Slate` 与 `D:\UE\5.8.0r\Engine\Source\Runtime\SlateCore`（模块根三件套）
- 代表性 `.Build.cs`：`UnrealEd.Build.cs`、`LevelEditor.Build.cs`、`PropertyEditor.Build.cs`、`ContentBrowser.Build.cs`、`EditorSubsystem.Build.cs`、`Slate.Build.cs`、`SlateCore.Build.cs`、`ToolMenus.Build.cs`、`AssetTools.Build.cs`
- 代表性头文件/声明的轻量 `rg` 定位：`SWidget`、`FExtender`、`FUICommandList`、`UToolMenus`、`IDetailCustomization`、`FAssetEditorToolkit`、`UEditorSubsystem`、`UCommandlet`
- 单文件确认：`D:\UE\5.8.0r\Engine\Source\Editor\EditorSubsystem\Public\EditorSubsystem.h`（24 行，确认继承链）

`Generated/`、`Intermediate/`、`Binaries/`、`DerivedDataCache/`、`Saved/` 等生成产物目录与 `ThirdParty` 第三方库 **只做排除说明，不进入内容**。

## 3. 非目标（Non-Goals）

1. 不做 Unreal Engine 全量源码扫描，不逐行通读任何编辑器 UI `.cpp/.h` 完整实现。
2. 不讲解任何具体编辑器 UI 控件（如 `SLevelViewport`、`SContentBrowser`）的绘制/布局/事件实现细节。
3. 不读取凭据、会话、个人配置、压缩包（`D:\UE\UnrealEngine-5.8.0-release.zip`）或生成产物（`Binaries`、`Intermediate`、`DerivedDataCache`、`Saved`）。
4. 不修改 `D:\UE\5.8.0r` 下任何源码、项目文件、生成产物或压缩包，不触发构建。
5. 不覆盖 `D:\UE\Docs` 下任何已有 `UE58_*.md`（含 3 组既有文档及其 ChangeSpec）。

## 4. 调查路线（Investigation Route）

| 步骤 | 动作 | 范围 | 证据形态（本次已得） |
| --- | --- | --- | --- |
| S1 | 列举 `Engine\Source\Editor` 顶层 | `D:\UE\5.8.0r\Engine\Source\Editor` | 145 条目，含 `UnrealEd`/`LevelEditor`/`PropertyEditor`/`ContentBrowser`/`EditorSubsystem`/`EditorFramework`/`ToolMenusEditor`/`Kismet`/`BlueprintGraph` |
| S2 | 定位 `Developer` 中编辑器支撑模块 | `D:\UE\5.8.0r\Engine\Source\Developer` | 138 条目，确认 `ToolMenus`/`AssetTools`/`Settings`/`SettingsEditor`/`DeveloperToolSettings` |
| S3 | 取 Slate 上下层结构 | `Runtime\Slate`、`Runtime\SlateCore` | 各为 `Public/Private/*.Build.cs` 三件套 |
| S4 | 取依赖表达样本 | 9 个代表性 `.Build.cs` | `PublicDependency`/`PrivateDependency`/`CircularlyReferencedDependentModules` 行号 |
| S5 | 取关键类型声明落点 | 8 个核心类型 `rg` | 各类型 `file:line` 命中（见主文档 §8 证据表） |
| S6 | 确认 Subsystem 继承链 | `EditorSubsystem.h` | `UEditorSubsystem : public UDynamicSubsystem`，`UCLASS(MinimalAPI, Abstract)` |

## 5. 产物（Deliverables）

1. 本 change spec：`D:\UE\Docs\UE58_05_Editor_Framework_Orientation_ChangeSpec.md`。
2. 主文档：`D:\UE\Docs\UE58_05_Editor_Framework_Orientation.md`，必须包含：
   - "thomas 先读这段：编辑器框架地图如何帮助不迷路"
   - ASCII 总览图：`Editor Module -> Slate UI -> Commands/ToolMenus -> Details/AssetEditor -> Subsystems/Commandlets`
   - Mermaid 1：编辑器模块边界图（Editor/Developer/Runtime 三域 + 代表模块）
   - Mermaid 2：UI/命令扩展链路图（`FUICommandList`/`FExtender`/`UToolMenus` 如何挂到 Slate）
   - Mermaid 3：从一个编辑器按钮反向定位源码的流程图
   - `Editor`/`Developer`/`Runtime` 边界、`Slate`/`SlateCore` 关系
   - `UnrealEd`/`LevelEditor`/`PropertyEditor`/`ContentBrowser`/`AssetTools` 职责
   - `EditorSubsystem`/`Commandlet`/`FAssetEditorToolkit` 角色
   - 关键类型 `file:line` 证据表
   - 与插件、测试、资源系统、源码层级文档的衔接
   - 阿卡姆剃刀检查
   - 局限性与潜在风险提示

## 6. 验收标准（Acceptance Criteria）

1. 主文档所有 Windows 路径均为完整绝对路径（反引号包裹）；可点击链接目标为正斜杠完整绝对路径并置于尖括号 `<...>` 中。
2. 主文档含 **至少 3 个 mermaid 图**：编辑器模块边界图、UI/命令扩展链路图、按钮反查源码流程图。
3. 主文档含 **一个 ASCII 总览图**，链路为 `Editor Module -> Slate UI -> Commands/ToolMenus -> Details/AssetEditor -> Subsystems/Commandlets`。
4. 主文档讲清第 2 节列出的全部关键角色，且每个关键类型给出 **实测 `file:line` 事实**。
5. 主文档逐处标注 **事实依据** 与 **逻辑分析推理(无事实依据)**；复杂描述主谓宾完整。
6. 主文档说明"机器绝对路径只是本机定位路径，工程内引用应使用模块相对包含路径或 Unreal 路径 API"。
7. 主文档与已有"源码层级整体认知"文档对同一路径/域边界/`Public`-`Private` 语义的描述不得互相矛盾（最后一步读取复核）。
8. 全程未读取范围外文件、未读取凭据/压缩包/生成产物、未修改任何引擎源码，未覆盖任何已有 `UE58_*.md`。

## 7. 局限性与潜在风险提示

- 本研究只看目录名、文件名、`.Build.cs` 与少量 `rg` 声明命中行，**未通读任何编辑器实现**；各模块"运行时如何协作渲染编辑器界面"的行为多为 **逻辑分析推理(无事实依据)**，需后续读代码或读 UE 官方文档验证。
- 模块职责描述部分基于模块名、目录结构与 `.Build.cs` 依赖方向推断，文件名与真实职责可能不完全一致。
- `ToolMenus` 与 `AssetTools` 位于 `Developer` 域而非 `Editor` 域，是本任务"编辑器框架"语义与目录物理位置的一处张力，主文档需显式说明，避免 thomas 在 `Editor` 下找不到它们。
- 绝对路径绑定本机 `D:\UE\5.8.0r` 布局，换机或换引擎版本即失效；这是为满足 thomas"完整绝对路径"硬性要求而做的取舍，与"不硬编码绝对路径"通用准则的张力已在主文档显式声明并给出工程内正确引用方式。
