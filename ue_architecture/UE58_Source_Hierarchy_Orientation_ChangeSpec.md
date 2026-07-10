# UE5.8 源码层级结构整体认知 Change Spec

> 本文档面向 thomas。这是"先建立源码层级整体认知、不迷路"任务的变更规格（change spec），用于约束主文档 [`UE58_Source_Hierarchy_Orientation.md`](<D:/UE/Docs/UE58_Source_Hierarchy_Orientation.md>) 的产出边界与验收标准。
>
> 本文档遵循阿卡姆剃刀：只描述本次任务必需的范围与判据，不展开任何子系统实现细节。
>
> **证据约定**：标 `【事实】` 的内容由本机目录/文件名/轻量 grep 直接验证；标 `【推理】` 的内容是 **逻辑分析推理(无事实依据)**，未通读实现，需后续读代码确认。

---

## 1. 目标（Goals）

1. 为 thomas 建立一张 **Unreal Engine 5.8 源码层级结构整体地图**，目标是"在源码里不迷路"，先看清四类源码域与模块边界，**不**先深入任何一条 Nanite / WorldPartition / HLOD 实现链路。
2. 解释 thomas 不熟悉的 UE 工程结构关键词：`Engine\Source`；`Runtime` / `Developer` / `Editor` / `Programs`；`Runtime\Engine` / `Runtime\Renderer` / `Runtime\RenderCore` 的区别；`Public` / `Private` / `Classes` 的模块边界意义；`.Build.cs` 如何表达模块依赖；`U/A/F/I/T` 前缀含义（标明是命名习惯而非编译器强制）；`API` 宏 / `MinimalAPI` / `UCLASS/USTRUCT/UINTERFACE/UENUM` / `GENERATED_BODY` 在"找定义和边界"中的作用；为什么很多关键实现在 `Private` 而非 `Public`。
3. 提供从"看到一个类名/函数名"到"定位所属模块、声明、实现、依赖"的读码路径。
4. 全程区分 **事实依据** 与 **逻辑分析推理(无事实依据)**。

## 2. 范围（Scope）

只读调查，且只允许触达以下目录级、有限深度的结构与少量代表性文件：

- `D:\UE\5.8.0r\Engine\Source`（顶层条目）
- `D:\UE\5.8.0r\Engine\Source\Runtime`（顶层模块清单）
- `D:\UE\5.8.0r\Engine\Source\Runtime\Engine`（模块根：`Classes/`、`Public/`、`Private/`、`Internal/`、`Engine.Build.cs`）
- `D:\UE\5.8.0r\Engine\Source\Runtime\Renderer`（模块根 + `Public/` 与 `Private/` 体量对比）
- `D:\UE\5.8.0r\Engine\Source\Runtime\RenderCore`（模块根 + `RenderCore.Build.cs`）
- `D:\UE\5.8.0r\Engine\Source\Developer`、`D:\UE\5.8.0r\Engine\Source\Editor`、`D:\UE\5.8.0r\Engine\Source\Programs`（顶层条目）
- 代表性 `.Build.cs`（`Renderer.Build.cs`、`RenderCore.Build.cs`）与代表性头文件宏（`Actor.h` 等）的轻量 grep

`Generated/`、`Intermediate/`、`Binaries/`、`DerivedDataCache/`、`Saved/` 等生成产物目录与 `ThirdParty` 第三方库 **只做排除说明，不进入内容**。

## 3. 非目标（Non-Goals）

1. 不做 Unreal Engine 全量源码扫描，不逐行通读任何 `.cpp/.h` 完整实现。
2. 不深入 Nanite / WorldPartition / HLOD 任何一条具体实现链路（这部分已由前一阶段文档覆盖，本文档只做衔接指引）。
3. 不读取凭据、会话、个人配置、压缩包（`D:\UE\UnrealEngine-5.8.0-release.zip`）或生成产物（`Binaries`、`Intermediate`、`DerivedDataCache`、`Saved`）。
4. 不修改 `D:\UE\5.8.0r` 下任何源码、项目文件、生成产物或压缩包，不触发构建。
5. 不覆盖已有文档 `D:\UE\Docs\UE58_Nanite_WorldPartition_HLOD_SourceMap.md` 与 `D:\UE\Docs\UE58_Nanite_WorldPartition_HLOD_ChangeSpec.md`。

## 4. 调查路线（Investigation Route）

| 步骤 | 动作 | 范围 | 证据形态 |
| --- | --- | --- | --- |
| S1 | 列举 `Engine\Source` 顶层 | `D:\UE\5.8.0r\Engine\Source` | 目录条目 + `*.Target.cs` 清单 |
| S2 | 列举四类源码域顶层 | `Runtime` / `Developer` / `Editor` / `Programs` | 各域条目数与代表性模块名 |
| S3 | 取单模块结构样本 | `Runtime\Engine`、`Runtime\Renderer`、`Runtime\RenderCore` | `Public/Private/Classes/Internal/*.Build.cs` 落点 |
| S4 | 取 Public/Private 体量对比 | `Runtime\Renderer\Public` 与 `...\Private` | 条目数对比（佐证"实现多在 Private"） |
| S5 | 取 `.Build.cs` 依赖表达样本 | `Renderer.Build.cs`、`RenderCore.Build.cs` | `PublicDependencyModuleNames` / `PrivateDependencyModuleNames` 行号 |
| S6 | 取宏存在性样本 | `Engine\Public\*.h`、`Engine\Classes\GameFramework\*.h` | `ENGINE_API`、`UCLASS/UINTERFACE/GENERATED_BODY/MinimalAPI` 行号 |

## 5. 产物（Deliverables）

1. 本 change spec：`D:\UE\Docs\UE58_Source_Hierarchy_Orientation_ChangeSpec.md`。
2. 主文档：`D:\UE\Docs\UE58_Source_Hierarchy_Orientation.md`，必须包含：
   - "thomas 先读这段：怎么使用这张地图"
   - 一句话总览：`Engine\Source` 是什么
   - ASCII 顶层地图（核心分支）
   - Mermaid 1：`Engine\Source` 分层图
   - 四类源码域：`Runtime` / `Developer` / `Editor` / `Programs`
   - 三个常见 Runtime 模块：`Engine` / `Renderer` / `RenderCore`
   - 单模块结构：`Public` / `Private` / `Classes` / `.Build.cs`
   - Mermaid 2：单模块 `Public/Private/Build.cs` 边界图
   - UE 命名与宏：`U/A/F/I/T`、API 宏、`GENERATED_BODY`
   - Mermaid 3：读码定位流程图（类名/函数名 → 模块 → 声明 → 实现 → 依赖）
   - 读码路线：从目录到模块到声明到实现到调用链
   - 与前一份 Nanite/WorldPartition/HLOD 文档的衔接
   - 阿卡姆剃刀检查
   - 局限性与潜在风险提示

## 6. 验收标准（Acceptance Criteria）

1. 主文档所有 Windows 路径均为完整绝对路径；可点击链接目标为正斜杠完整绝对路径并置于尖括号 `<...>` 中。
2. 主文档含 **至少 3 个 mermaid 图**：`Engine\Source` 分层图、单模块边界图、读码定位流程图。
3. 主文档含 **一个 ASCII 总览图**，展示 `D:\UE\5.8.0r\Engine\Source` 下核心分支。
4. 主文档解释第 1 节列出的全部关键词，且显式标明 `U/A/F/I/T` 前缀是 UE/C++ 命名习惯而非编译器强制规则。
5. 主文档逐处标注 **事实依据** 与 **逻辑分析推理(无事实依据)**；复杂描述主谓宾完整。
6. 主文档说明"机器绝对路径只是本机定位路径，工程内引用应使用模块相对包含路径或 Unreal 路径 API"。
7. 两份文档对同一路径/职责/模块边界/验证方式的描述不得互相矛盾（最后一步读取复核）。
8. 全程未读取范围外文件、未读取凭据/压缩包/生成产物、未修改任何引擎源码，未覆盖已有两份文档。

## 7. 局限性与潜在风险提示

- 本研究只看目录名、文件名、`.Build.cs` 与少量宏 grep 行，**未通读任何实现**；四类源码域的"打包/加载语义"等运行时行为多为 **逻辑分析推理(无事实依据)**，需后续读代码或读 UBT 文档验证。
- 模块职责描述部分基于模块名与目录结构推断，文件名与真实职责可能不完全一致。
- 文档中的绝对路径绑定本机 `D:\UE\5.8.0r` 布局，换机或换引擎版本即失效；这是为满足 thomas"完整绝对路径"硬性要求而做的取舍，与"代码/文档不硬编码绝对路径"的通用准则存在张力，已在主文档显式声明并给出工程内正确引用方式。
