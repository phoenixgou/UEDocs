# UE5.8 编辑器自动化测试机制源码地图 Change Spec（任务 2/5）

> 本文档面向 thomas。这是"建立 Unreal Engine 5.8 编辑器自动化测试机制源码地图、不迷路"任务的变更规格（change spec），用于约束主文档 [`UE58_02_Editor_Automation_Testing_Orientation.md`](<D:/UE/Docs/UE58_02_Editor_Automation_Testing_Orientation.md>) 的产出边界与验收标准。
>
> 本文档遵循阿卡姆剃刀：只描述本次任务必需的范围与判据，不精读任何单个测试实现。
>
> **证据约定**：标 `【事实】` 的内容由本机目录/文件名/`.Build.cs`/轻量 `rg` 行号直接验证；标 `逻辑分析推理(无事实依据)` 的内容未通读实现，是基于 UE 通用约定的推断，需后续读代码确认。

---

## 1. 目标（Goals）

1. 为 thomas 建立一张 **Unreal Engine 5.8 编辑器自动化测试机制源码地图**，目标是看清测试 **如何注册、发现、调度、执行、报告** 五个环节落在哪些模块、哪些类、哪些文件，做到"在测试源码里不迷路"，**不**精读任何单个测试的实现。
2. 解释 thomas 可能不熟悉的关键词与组件：`Core/Public/Misc/AutomationTest.h` 中的测试宏（`IMPLEMENT_SIMPLE_AUTOMATION_TEST` 等）与核心类（`FAutomationTestFramework`、`FAutomationTestBase`）；`AutomationController`、`AutomationWorker`、`AutomationMessages`、`AutomationTest`（模块）、`AutomationWindow`、`SessionFrontend`、`FunctionalTesting`、`AutomationTool`、`Gauntlet`、`LowLevelTests` 各自的边界。
3. 说明四类测试（单元/编辑器/功能/集成）如何区分，以及"在哪里找测试"。
4. 说明本机制与 **插件机制、UBT/UHT、编辑器框架** 的衔接点。
5. 全程区分 **事实依据** 与 **逻辑分析推理(无事实依据)**。

## 2. 范围（Scope）

只读调查，且只允许触达以下目录级、有限深度的结构与少量代表性文件：

- `D:\UE\5.8.0r\Engine\Source\Runtime\Core\Public\Misc\AutomationTest.h`（宏与核心类声明）
- `D:\UE\5.8.0r\Engine\Source\Runtime\Core\Private\Misc\AutomationTest.cpp`（注册实现的少量行号）
- `D:\UE\5.8.0r\Engine\Source\Runtime\AutomationTest`（模块根 + `.Build.cs` + Public/Private 清单）
- `D:\UE\5.8.0r\Engine\Source\Runtime\AutomationWorker`（模块根 + `.Build.cs` + 消息端点 grep）
- `D:\UE\5.8.0r\Engine\Source\Runtime\AutomationMessages`（模块根 + Public 清单）
- `D:\UE\5.8.0r\Engine\Source\Developer\AutomationController`（模块根 + `.Build.cs` + Public/Private 清单 + 命令行入口 grep）
- `D:\UE\5.8.0r\Engine\Source\Developer\AutomationWindow`、`D:\UE\5.8.0r\Engine\Source\Developer\SessionFrontend`（模块根级存在性）
- `D:\UE\5.8.0r\Engine\Source\Developer\FunctionalTesting`（模块根 + `Classes/Public/Private` 清单 + `.Build.cs`）
- `D:\UE\5.8.0r\Engine\Source\Programs\AutomationTool`、`...\AutomationTool\Gauntlet`、`...\AutomationTool\LowLevelTests`（顶层定位）
- 代表性 `<Module>\Private\Tests` 目录与少量 `IMPLEMENT_*_AUTOMATION_TEST` 命中文件（佐证"测试在哪")

`Binaries`、`Intermediate`、`DerivedDataCache`、`Saved`、`Generated`、`obj`、`bin` 等生成产物目录 **只做排除说明，不进入内容**。

## 3. 非目标（Non-Goals）

1. 不做 Unreal Engine 全量源码扫描，不逐行通读任何测试实现或框架 `.cpp` 完整实现。
2. 不深入任何单个测试用例（如某个具体 `FileManagerTest` 的断言逻辑）。
3. 不读取凭据、会话、个人配置、压缩包（`D:\UE\UnrealEngine-5.8.0-release.zip`）或生成产物（`Binaries`、`Intermediate`、`DerivedDataCache`、`Saved`）。
4. 不修改 `D:\UE\5.8.0r` 下任何源码、项目文件、生成产物或压缩包，不触发构建。
5. 不覆盖已有文档：`D:\UE\Docs\UE58_Source_Hierarchy_Orientation.md`、`D:\UE\Docs\UE58_Source_Hierarchy_Orientation_ChangeSpec.md`、`D:\UE\Docs\UE58_Nanite_WorldPartition_HLOD_SourceMap.md`、`D:\UE\Docs\UE58_Nanite_WorldPartition_HLOD_ChangeSpec.md`、`D:\UE\Docs\UE58_Rendering_Animation_DeepDive.md`、`D:\UE\Docs\UE58_Rendering_Animation_DeepDive_ChangeSpec.md`。

## 4. 调查路线（Investigation Route）

| 步骤 | 动作 | 范围 | 证据形态 |
| --- | --- | --- | --- |
| S1 | 核验关键路径存在性 | `AutomationTest.h`、`AutomationController`、`AutomationWorker`、`FunctionalTesting`、`AutomationTool` | 目录/文件存在 + 真实落点（含 `AutomationWorker` 实为 `Runtime\` 而非 `Developer\` 的纠偏） |
| S2 | 宏与核心类定位 | `AutomationTest.h` | `IMPLEMENT_*` 宏行号、`FAutomationTestFramework`/`FAutomationTestBase` 行号 |
| S3 | 注册路径定位 | `AutomationTest.cpp` | 构造函数行号 + `RegisterAutomationTest` 调用行号 |
| S4 | Controller/Worker 依赖与通道 | 两模块 `.Build.cs` + `AutomationWorkerModule.cpp` 消息端点 | 依赖模块名 + `Handling<...>` 消息类型行号 |
| S5 | 命令行/编辑器入口 | `AutomationCommandline.cpp`、`AutomationWindow`、`SessionFrontend` | `FParse::Command("Automation")` 行号 + UI 模块存在 |
| S6 | 功能测试与外层框架边界 | `FunctionalTesting`（Classes/Build.cs）、`Gauntlet`、`LowLevelTests` | UObject 测试类清单 + C# `.csproj` 存在 |
| S7 | 测试落点取样 | `<Module>\Private\Tests` | 代表性 `Tests` 目录与命中文件 |

## 5. 产物（Deliverables）

1. 本 change spec：`D:\UE\Docs\UE58_02_Editor_Automation_Testing_Orientation_ChangeSpec.md`。
2. 主文档：`D:\UE\Docs\UE58_02_Editor_Automation_Testing_Orientation.md`，必须包含：
   - "thomas 先读这段：测试机制地图如何帮助不迷路"
   - 一句话总览 + **ASCII 总览图**：`Test Macro -> AutomationTestFramework -> Controller -> Worker -> Editor/Commandlet -> Report`
   - **Mermaid 1**：测试注册与发现图
   - **Mermaid 2**：Controller / Worker 调度图（含 MessageBus）
   - **Mermaid 3**：从测试名定位源码流程图
   - `AutomationTest.h` 宏与核心类解释（含 `WITH_AUTOMATION_WORKER` 静态实例自注册机制、`EAutomationTestFlags` 上下文/过滤位）
   - Editor 自动化 / Functional Test / Gauntlet / AutomationTool 的边界
   - "在哪里找测试" + 如何区分单元 / 编辑器 / 功能 / 集成测试
   - 与 **插件机制、UBT/UHT、编辑器框架** 的衔接点
   - 阿卡姆剃刀检查
   - 局限性与潜在风险提示

## 6. 验收标准（Acceptance Criteria）

1. 主文档所有 Windows 路径均为完整绝对路径；可点击链接目标为正斜杠完整绝对路径并置于尖括号 `<...>` 中。
2. 主文档含 **至少 3 个 mermaid 图**：测试注册与发现图、Controller/Worker 调度图、从测试名定位源码流程图。
3. 主文档含 **一个 ASCII 总览图**，展示 `Test Macro -> AutomationTestFramework -> Controller -> Worker -> Editor/Commandlet -> Report` 全链路。
4. 主文档逐处标注 **事实依据**（带 `file:line` 或目录证据）与 **逻辑分析推理(无事实依据)**；复杂描述主谓宾完整。
5. 主文档显式纠偏：`AutomationWorker` 模块真实落点是 `D:\UE\5.8.0r\Engine\Source\Runtime\AutomationWorker`，不在 `Developer\` 下；`FunctionalTesting` 真实落点是 `D:\UE\5.8.0r\Engine\Source\Developer\FunctionalTesting`。
6. 主文档说明"机器绝对路径只是本机定位路径，工程内引用应使用模块相对包含路径或 Unreal 路径 API"。
7. 两份文档与 `D:\UE\Docs` 下既有 6 份 UE58 文档对同一路径/职责/模块边界的描述不得互相矛盾（最后一步读取复核）。
8. 全程未读取范围外文件、未读取凭据/压缩包/生成产物、未修改任何引擎源码，未覆盖已有任何文档。

## 7. 局限性与潜在风险提示

- 本研究只看目录名、文件名、`.Build.cs` 与少量 `rg` 行，**未通读任何测试或框架完整实现**；"调度/执行/报告"的运行时时序多为 **逻辑分析推理(无事实依据)**，需后续读 `AutomationControllerManager.cpp`、`AutomationWorkerModule.cpp` 完整实现或读 UE 文档验证。
- 组件职责描述部分基于模块名、目录结构与依赖表推断，文件名与真实职责可能不完全一致。
- 绝对路径绑定本机 `D:\UE\5.8.0r` 布局，换机或换引擎版本即失效；这是为满足 thomas"完整绝对路径"硬性要求做的取舍，与"代码/文档不硬编码绝对路径"通用准则存在张力，已在主文档显式声明并给出工程内正确引用方式。
