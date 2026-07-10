# UE5.8 Nanite × WorldPartition × HLOD 源码结构研究 Change Spec

> 本文档面向 thomas。这是第一阶段的变更规格（change spec），用于约束第二阶段主文档 `D:\UE\Docs\UE58_Nanite_WorldPartition_HLOD_SourceMap.md` 的产出边界与验收标准。
>
> 本文档遵循阿卡姆剃刀：只描述本次任务必需的范围与判据，不展开实现细节。

## 1. 目标（Goals）

1. 为 thomas 建立一张可快速跳转的 **Unreal Engine 5.8 源码地图**，聚焦 Nanite、WorldPartition（大世界）、HLOD 三大子系统的源码落点。
2. 解释 thomas 不熟悉的 UE 工程结构关键词：`Engine\Source\Runtime`、`Runtime\Renderer`、`Runtime\Engine`、`Public`、`Private`、`F` 前缀类型命名习惯。
3. 用关系图与运行链路图，把以下七个概念串成一张图：大世界、资源管理、LOD、Nanite、流送、GPU 剔除、GPU 管线组装。
4. 全程区分 **事实依据**（由目录/文件名/轻量 grep 验证）与 **逻辑分析推理(无事实依据)**（未读取代码内部实现而做的推断）。

## 2. 范围（Scope）

只读调查，且只允许触达以下范围内的目录名、文件名与轻量 grep：

- `D:\UE\5.8.0r\Engine\Source\Runtime\Renderer\Private\Nanite`
- `D:\UE\5.8.0r\Engine\Source\Runtime\Engine\Public\WorldPartition`
- `D:\UE\5.8.0r\Engine\Source\Runtime\Engine\Private\WorldPartition`
- `D:\UE\5.8.0r\Engine\Source\Runtime\Engine\Public` 与 `D:\UE\5.8.0r\Engine\Source\Runtime\Engine\Private` 内 HLOD 相关目录/文件
- `D:\UE\5.8.0r\Engine\Source\Runtime\Renderer\Private` 下，仅用文件名或轻量 grep 定位 HLOD、GPUScene、InstanceCulling、HZB、RenderGraph、Nanite 相关入口

## 3. 非目标（Non-Goals）

1. 不做 Unreal Engine 全量扫描，不逐行通读上述任何 `.cpp/.h` 的完整实现。
2. 不读取凭据、会话、个人配置、压缩包（如 `D:\UE\UnrealEngine-5.8.0-release.zip`）或生成产物（`Binaries`、`Intermediate`、`DerivedDataCache`、`Saved`）。
3. 不修改任何引擎源码或项目文件，不触发构建。
4. 不解释 Nanite/WorldPartition/HLOD 的算法原理或数学推导；只交付"源码在哪、谁连谁"的地图。

## 4. 调查路线（Investigation Route）

| 步骤 | 动作 | 范围 | 证据形态 |
| --- | --- | --- | --- |
| S1 | 列举 Nanite 目录 | `...\Renderer\Private\Nanite` | 目录条目清单 |
| S2 | 列举 WorldPartition Public/Private 目录 | `...\Engine\Public\WorldPartition`、`...\Engine\Private\WorldPartition` | 目录条目清单 |
| S3 | 定位 HLOD 相关目录/文件 | `...\Engine\Public`、`...\Engine\Private` | 文件名清单 |
| S4 | 文件名定位渲染侧入口 | `...\Renderer\Private`（GPUScene/InstanceCulling/HZB/SceneCulling）+ RenderCore 模块（RenderGraph） | 文件名清单 |
| S5 | 轻量 grep 验证关键连接 | Nanite 对 GPUScene/SceneCulling 的 `#include`；BasePass 对 Nanite/InstanceCulling/RDG 的引用 | grep 行号命中 |

## 5. 产物（Deliverables）

1. 本 change spec：`D:\UE\Docs\UE58_Nanite_WorldPartition_HLOD_ChangeSpec.md`。
2. 主文档：`D:\UE\Docs\UE58_Nanite_WorldPartition_HLOD_SourceMap.md`，必须包含：
   - 快速定位表（路径 → 职责）
   - UE 工程结构导读（关键词解释）
   - 源码目录职责
   - 系统关系总览图（mermaid，≥1）
   - 关键运行链路图（mermaid，≥1）
   - 阅读路线
   - 局限性与潜在风险

## 6. 验收标准（Acceptance Criteria）

1. 主文档所有 Windows 路径均为完整绝对路径；可点击链接目标为正斜杠完整绝对路径并置于尖括号 `<...>` 中。
2. 主文档至少含 2 个 mermaid 图：一个系统关系总览图、一个关键运行链路图。
3. 主文档覆盖七个概念：大世界、资源管理、LOD、Nanite、流送、GPU 剔除、GPU 管线组装，并显式画出它们之间的关系。
4. 主文档逐处标注 **事实依据** 与 **逻辑分析推理(无事实依据)**。
5. 主文档解释了 `Engine\Source\Runtime`、`Runtime\Renderer`、`Runtime\Engine`、`Public`、`Private`、`F` 前缀六个关键词。
6. 两份文档对同一路径/职责/验证方式的描述不得互相矛盾（最后一步读取复核）。
7. 全程未读取范围外文件、未读取凭据/压缩包/生成产物、未修改引擎源码。

## 7. 局限性与潜在风险提示

- 本研究只看目录名、文件名与少量 `#include` 行，**未通读实现**；主文档中系统间数据流向多为 **逻辑分析推理(无事实依据)**，需后续读代码验证。
- 文档中的绝对路径绑定本机 `D:\UE\5.8.0r` 布局，换机或换引擎版本即失效；这是为满足 thomas 的"完整绝对路径"硬性要求而做的取舍，与"代码/文档不硬编码绝对路径"的通用准则存在张力，已在此显式声明。
