# UE5.8 UBT/UHT/Build.cs/Target.cs 构建机制源码地图 Change Spec

> 本文档面向 thomas。这是第 3/5 个文档任务"建立 Unreal Engine 5.8 构建机制整体认知、读源码不迷路"的变更规格（change spec），用于约束主文档 [`UE58_03_UBT_UHT_Build_System_Orientation.md`](<D:/UE/Docs/UE58_03_UBT_UHT_Build_System_Orientation.md>) 的产出边界与验收标准。
>
> 本文档遵循阿卡姆剃刀：只描述本次任务必需的范围与判据，**不**做 UBT/UHT 实现精读。
>
> **证据约定**：标 `【事实】` 的内容由本机目录条目、文件名、`file:line` 或轻量 `rg` 直接验证；标 `逻辑分析推理(无事实依据)` 的内容基于 UBT/UE 通用约定推断，未通读实现，需后续读代码或读官方文档确认。

---

## 1. 目标（Goals）

1. 为 thomas 建立一张 **Unreal Engine 5.8 构建机制源码地图**，目标是看清 **编译入口 → 模块规则 → Target → UHT 生成代码 → Public/Private include 边界** 这一条主链，做到"改 / 读构建相关源码时不迷路"。
2. 解释 thomas 关心的关键概念：`*.Target.cs`（Target 角色与 `TargetType`）；`*.Build.cs`（`ModuleRules` 字段含义）；`UnrealBuildTool`（UBT，读规则决定编译/链接）；`UnrealHeaderTool`（UHT，扫描反射宏生成样板代码）；`GENERATED_BODY()` 与 `*.generated.h` 的关系；`Public` / `Private` 依赖如何影响跨模块 `#include`；为什么改 `*.Build.cs` 后必须重新生成项目文件并重新编译。
3. 提供"从一个 `.uproject` / 一个模块 / 一个 `UCLASS` 出发，定位它由谁编译、依赖谁、反射代码生成到哪里"的读码路径。
4. 全程区分 **事实依据** 与 **逻辑分析推理(无事实依据)**。

## 2. 范围（Scope）

只读调查，且只允许触达以下目录级、有限深度的结构与少量代表性文件：

- `D:\UE\5.8.0r\Engine\Source`（顶层条目 + 4 个 `*.Target.cs`）
- `D:\UE\5.8.0r\Engine\Source\Programs\UnrealBuildTool`（顶层条目 + `Configuration\` + `Configuration\Rules\` 条目）
- `D:\UE\5.8.0r\Engine\Source\Programs\Shared\EpicGames.UHT`（顶层条目 + `Exporters\CodeGen\` 条目）
- 代表性 `*.Target.cs`：`UnrealGame.Target.cs`、`UnrealEditor.Target.cs`、`UnrealClient.Target.cs`、`UnrealServer.Target.cs`
- 代表性 `*.Build.cs`：`Runtime\Core\Core.Build.cs`、`Runtime\Renderer\Renderer.Build.cs`（Renderer 行号与前序文档复核一致）
- 规则定义源：`Configuration\Rules\ModuleRules.cs`、`Configuration\Rules\TargetRules.cs`（仅对关键字段做定点 `rg`，不全文通读）
- 反射锚点样本：`Runtime\Engine\Classes\GameFramework\Actor.h` 等头文件中 `#include "*.generated.h"`、`UCLASS`、`GENERATED_BODY` 的 `rg` 命中
- 三种工程声明形态：`D:\UE\AnimationSamples\AnimationSamples.uproject`、`D:\UE\ProjectTitan\ProjectTitan.uproject`、`D:\UE\tutorial\tutorial.uproject`（只读查看 `Modules` / `Plugins` 声明片段）

`obj/`、`bin/`、`Intermediate/`、`Binaries/`、`DerivedDataCache/`、`Saved/` 等生成产物（含真实 `*.generated.h`、`*.gen.cpp`）与 `ThirdParty` **只做排除说明，不进入内容**。

## 3. 非目标（Non-Goals）

1. 不做 Unreal Engine 全量源码扫描，不逐行通读 UBT/UHT 的 C# 实现，不解释其内部算法。
2. 不深入任何 Nanite / WorldPartition / HLOD / Renderer 具体实现链路（已由前序文档覆盖，本文只做衔接指引）。
3. 不读取凭据、会话、个人配置、压缩包（`D:\UE\UnrealEngine-5.8.0-release.zip`）或生成产物（`Binaries`、`Intermediate`、`DerivedDataCache`、`Saved`、真实 `*.generated.h`）。
4. 不修改 `D:\UE\5.8.0r` 下任何源码、项目文件、生成产物或压缩包，不触发构建。
5. 不覆盖 `D:\UE\Docs` 下已有的 4 份文档（Nanite/WorldPartition/HLOD 两份、Rendering/Animation 两份、Source Hierarchy 两份中的任意一份）。

## 4. 调查路线（Investigation Route）

| 步骤 | 动作 | 范围 | 证据形态 |
| --- | --- | --- | --- |
| S1 | 列举构建入口 | `D:\UE\5.8.0r\Engine\Source` | 顶层条目 + 4 个 `*.Target.cs` 清单 |
| S2 | 读 Target 角色 | 4 个 `*.Target.cs` + `TargetRules.cs:21` 的 `TargetType` 枚举 | `Type=` 赋值与枚举值行号 |
| S3 | 读模块规则字段 | `Core.Build.cs`、`Renderer.Build.cs` + `ModuleRules.cs` 字段 | 字段使用样本 + 字段声明行号 |
| S4 | 列举 UBT 结构 | `Programs\UnrealBuildTool` + `Configuration\` + `Configuration\Rules\` | 目录条目（含 `UEBuildTarget/Module/Binary` 与规则文件） |
| S5 | 列举 UHT 结构 | `Programs\Shared\EpicGames.UHT` + `Exporters\CodeGen\` | 目录条目（代码生成器文件名） |
| S6 | 取反射锚点样本 | `GameFramework\*.h` | `#include "*.generated.h"`、`UCLASS`、`GENERATED_BODY` 的 `file:line` |
| S7 | 取工程声明样本 | 三个 `.uproject` | `Modules` / `Plugins` 声明片段 |

## 5. 产物（Deliverables）

1. 本 change spec：`D:\UE\Docs\UE58_03_UBT_UHT_Build_System_Orientation_ChangeSpec.md`。
2. 主文档：`D:\UE\Docs\UE58_03_UBT_UHT_Build_System_Orientation.md`，必须包含：
   - "thomas 先读这段：构建机制如何帮助读源码不迷路"
   - ASCII 总览图：`.uproject/.uplugin → Target.cs → Build.cs → UBT → UHT → Compile/Link`
   - Mermaid 1：构建输入输出图（谁吃谁、产出什么）
   - Mermaid 2：模块依赖图（`Public/Private` 依赖方向，基于 `Renderer.Build.cs` 实测）
   - Mermaid 3：UHT 生成代码定位图（从 `UCLASS/GENERATED_BODY` 到 `*.generated.h`/`*.gen.cpp`）
   - `*.Build.cs` 字段解释（`ModuleRules` 各依赖列表的语义与传播差异）
   - `*.Target.cs` 角色（`TargetType` 五类与 `BuildEnvironment`/`ExtraModuleNames`）
   - UHT 与 `GENERATED_BODY` / `*.generated.h` 的关系
   - `Public` / `Private` 依赖如何影响 `#include`
   - 为什么改 `*.Build.cs` 后要重新生成项目文件并重新编译
   - 与插件、测试、源码层级文档的衔接
   - 阿卡姆剃刀检查
   - 局限性与潜在风险提示

## 6. 验收标准（Acceptance Criteria）

1. 主文档所有 Windows 路径均为完整绝对路径；可点击链接目标为正斜杠完整绝对路径并置于尖括号 `<...>` 中。
2. 主文档含 **至少 3 个 mermaid 图**：构建输入输出图、模块依赖图、UHT 生成代码定位图。
3. 主文档含 **一个 ASCII 总览图**，按 `.uproject/.uplugin → Target.cs → Build.cs → UBT → UHT → Compile/Link` 串起主链。
4. 主文档解释第 1 节列出的全部关键概念，且 Build.cs 字段含义、Target 角色、UHT 与 `GENERATED_BODY`/`*.generated.h` 关系、Public/Private 对 `#include` 的影响、改 Build.cs 后为何重编译，五点齐备。
5. 主文档逐处标注 **事实依据** 与 **逻辑分析推理(无事实依据)**；复杂描述主谓宾完整。
6. 主文档说明"机器绝对路径只是本机定位路径，工程内引用应使用模块名/模块相对包含路径或 Unreal 路径 API，不得硬编码 `D:\UE\...`"。
7. 本文档与前序 5 份文档对同一路径/职责/模块边界/`Renderer.Build.cs` 行号的描述不得互相矛盾（最后一步读取复核）。
8. 全程未读取范围外文件、未读取凭据/压缩包/生成产物、未修改任何引擎源码，未覆盖已有文档。

## 7. 局限性与潜在风险提示

- 本研究只看目录条目、`*.Target.cs`/`*.Build.cs` 字段、`ModuleRules.cs`/`TargetRules.cs` 字段声明行与少量反射宏 `rg` 命中，**未通读 UBT/UHT 任何 C# 实现**；"UBT 如何调度编译"、"UHT 如何生成具体代码"、"PublicDependency 如何向上游传播"等运行时/构建期行为多为 **逻辑分析推理(无事实依据)**，需后续读 UBT/UHT 源码或官方构建文档验证。
- 真实的 `*.generated.h` / `*.gen.cpp` 位于 `Intermediate` 生成产物目录，本任务 **未读取**；主文档对其内容与落盘位置的描述基于 UE 通用约定推断。
- 文档中的绝对路径绑定本机 `D:\UE\5.8.0r` 布局，换机或换引擎版本即失效；这是为满足 thomas"完整绝对路径"硬性要求而做的取舍，与"代码/文档不硬编码绝对路径"通用准则存在张力，已在主文档显式声明并给出工程内正确引用方式。
