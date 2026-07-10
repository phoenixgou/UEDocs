# UE5.8 渲染 × 动画 两条主线深入 Change Spec

> 本文档面向 thomas。这是"在已有源码层级整体认知之上、继续深入渲染与动画两条主线"任务的变更规格（change spec），用于约束主文档 [`UE58_Rendering_Animation_DeepDive.md`](<D:/UE/Docs/UE58_Rendering_Animation_DeepDive.md>) 的产出边界与验收标准。
>
> 本文档遵循阿卡姆剃刀：只描述本次任务必需的范围与判据，不展开任何算法实现细节，不通读源码。
>
> **证据约定**：标 `【事实】` 的内容由本机目录/文件名/`.Build.cs`/轻量 grep（含 `file:line`）直接验证；标 `【推理】` 的内容是 **逻辑分析推理(无事实依据)**，基于 UE 通用约定推断，未通读实现，需后续读代码确认。
>
> **路径表述约定**：正文与 ASCII 图中提及目录/文件一律用 **完整绝对路径**（Windows 原生反斜杠形式，反引号包裹）；mermaid 图节点为避免节点过长，使用 **以 `D:\UE\5.8.0r\Engine\Source` 为根的模块相对简写**（正斜杠风格），不代表磁盘真实分隔符。

---

## 1. 目标（Goals）

1. 在已有 [`UE58_Source_Hierarchy_Orientation.md`](<D:/UE/Docs/UE58_Source_Hierarchy_Orientation.md>)（层级整体认知）与 [`UE58_Nanite_WorldPartition_HLOD_SourceMap.md`](<D:/UE/Docs/UE58_Nanite_WorldPartition_HLOD_SourceMap.md>)（子系统地图）之上，为 thomas 建立 **渲染** 与 **动画** 两条主线的 **更详细但仍可读** 的源码地图与读码路线，目标是"在渲染与动画源码里不迷路"，而非通读全部实现。
2. 给出两条主线各自的：模块边界与依赖（事实由 `.Build.cs` 支撑）、源码目录职责、每帧高层路径、从一个具体对象（像素/Mesh/Shader/Nanite 对象；SkeletalMeshComponent/AnimBlueprint/AnimNode/Montage/BlendSpace）定位源码的读码路线。
3. 说明 **渲染与动画的交界**：`USkeletalMeshComponent` 最终如何走向渲染侧（哪些是事实，哪些是推理）。
4. 全程区分 **事实依据** 与 **逻辑分析推理(无事实依据)**，复杂描述主谓宾完整。

## 2. 范围（Scope）

只读调查，且只允许触达以下目录级、有限深度的结构与少量代表性文件（每个调查点最多 5 次搜索，超过则标注未知）：

**渲染主线**：

- `D:\UE\5.8.0r\Engine\Source\Runtime\Renderer`（模块根 + `Public/` 与 `Private/` 体量、`Private/` 子系统目录清单）
- `D:\UE\5.8.0r\Engine\Source\Runtime\RenderCore`（模块根 + `Public/` 清单）
- `D:\UE\5.8.0r\Engine\Source\Runtime\RHI`（模块根）
- 三个 `.Build.cs`：`Renderer.Build.cs`、`RenderCore.Build.cs`、`RHI.Build.cs`
- `D:\UE\5.8.0r\Engine\Shaders`（仅顶层目录说明，不全量展开）
- 渲染强相关代表性入口（仅文件名/轻量 grep 定位）：`Nanite`、`Lumen`、`GPUScene`、`SceneCulling`、`InstanceCulling`、`HZB`、`RenderGraph`、`MeshPassProcessor`/`MeshDrawCommands`、`BasePassRendering`、`DeferredShadingRenderer`、`ShadowRendering`/`VirtualShadowMaps`、`PathTracing`/`RayTracing`、`PostProcess`、`SceneRendering`/`SceneVisibility`

**动画主线**：

- `D:\UE\5.8.0r\Engine\Source\Runtime\Engine\Classes\Animation`
- `D:\UE\5.8.0r\Engine\Source\Runtime\Engine\Public\Animation`
- `D:\UE\5.8.0r\Engine\Source\Runtime\Engine\Private\Animation`
- `D:\UE\5.8.0r\Engine\Source\Runtime\AnimGraphRuntime`（模块根 + `Public/`）
- `D:\UE\5.8.0r\Engine\Source\Runtime\AnimationCore`（模块根 + `Public/`）
- AnimationBudgetAllocator：实测 **不在** `Runtime\` 源码目录，而在插件目录（见 §4 S3，需在主文档纠正任务假设）
- `D:\UE\5.8.0r\Engine\Plugins\Animation`（仅顶层插件目录清单，不全量展开）
- 三个 `.Build.cs`：`AnimGraphRuntime.Build.cs`、`AnimationCore.Build.cs`、`AnimationBudgetAllocator.Build.cs`

**交界主线**：

- `D:\UE\5.8.0r\Engine\Source\Runtime\Engine\Public\SkeletalRenderPublic.h`、`...\SkeletalMeshSceneProxy.h`、`...\GPUSkinCache.h`、`...\PrimitiveSceneProxy.h`
- `D:\UE\5.8.0r\Engine\Source\Runtime\Engine\Private\SkeletalRender*.h`
- `D:\UE\5.8.0r\Engine\Source\Runtime\Renderer\Private\Skinning`（目录定位）

## 3. 非目标（Non-Goals）

1. 不做 Unreal Engine 全量源码扫描，不逐行通读任何 `.cpp/.h` 完整实现。
2. 不解释渲染/动画的算法原理或数学推导（如 Nanite 簇剔除算法、姿势压缩算法、IK 求解推导）；只交付"源码在哪、谁连谁、怎么找"的地图与路线。
3. 不展开 `D:\UE\Docs\UE58_Nanite_WorldPartition_HLOD_SourceMap.md` 已覆盖的 Nanite/WorldPartition/HLOD 内部链路，只做衔接引用。
4. 不展开 `Engine\Shaders` 内部、不全量展开 `Engine\Plugins\Animation` 各插件内部；不展开 `DerivedDataCache`、`Binaries`、`Intermediate`、`Saved`、压缩包、凭据、会话或个人配置。
5. 不修改 `D:\UE\5.8.0r` 下任何源码、项目文件、生成产物或压缩包，不触发构建。
6. 不覆盖已有四份文档；不修改 `D:\UE\AnimationSamples`、`D:\UE\ProjectTitan`、`D:\UE\tutorial` 任何文件。

## 4. 调查路线（Investigation Route）

| 步骤 | 动作 | 范围 | 证据形态 |
| --- | --- | --- | --- |
| S1 | 列举渲染三模块根 | `Runtime\Renderer`、`Runtime\RenderCore`、`Runtime\RHI` | 目录条目（`Public/Private/Internal/*.Build.cs`） |
| S2 | 取渲染 `.Build.cs` 依赖 | `Renderer.Build.cs`、`RenderCore.Build.cs`、`RHI.Build.cs` | `Public/PrivateDependencyModuleNames`、`DynamicallyLoadedModuleNames` 行号 |
| S3 | 列举渲染 `Private`/`Public` 与 Shaders 顶层 | `Renderer\Private`、`Renderer\Public`、`RenderCore\Public`、`Engine\Shaders` | 条目数与子系统目录/文件名清单 |
| S4 | 轻量 grep 渲染关键类 | `SceneRendering.h`、`MeshPassProcessor.h`、`Engine\Public\PrimitiveSceneProxy.h` | `FSceneRenderer`/`FMeshDrawCommand`/`FPrimitiveSceneProxy` 的 `file:line` |
| S5 | 列举动画三处目录与两模块 | `Engine\Classes\Animation`、`...\Public\Animation`、`...\Private\Animation`、`AnimGraphRuntime`、`AnimationCore` | 条目数与代表文件名 |
| S6 | 取动画 `.Build.cs` 依赖、定位 BudgetAllocator | `AnimGraphRuntime.Build.cs`、`AnimationCore.Build.cs`、`Engine\Plugins\Animation`、AnimationBudgetAllocator | 依赖行号 + 插件目录清单 + BudgetAllocator 实际路径 |
| S7 | 轻量 grep 动画关键类 | `AnimInstance.h`、`AnimInstanceProxy.h`、`AnimationRuntime.h`、`SkeletalMeshComponent.h` | `UAnimInstance`/`FAnimInstanceProxy`/`FAnimationRuntime`/`USkeletalMeshComponent` 的 `file:line` |
| S8 | 轻量 grep 交界类 | `SkeletalRenderPublic.h`、`SkeletalRender*.h`、`SkeletalMeshSceneProxy.h` | `FSkeletalMeshObject` 家族与 `FSkeletalMeshSceneProxy` 的 `file:line` |

## 5. 产物（Deliverables）

1. 本 change spec：`D:\UE\Docs\UE58_Rendering_Animation_DeepDive_ChangeSpec.md`。
2. 主文档：`D:\UE\Docs\UE58_Rendering_Animation_DeepDive.md`，必须包含：
   - thomas 先读这段：如何用这份文档不迷路
   - 两条主线的 ASCII 总览图（渲染链路、动画链路）
   - 至少 4 个 mermaid 图：渲染模块边界与依赖图、渲染每帧高层路径图、动画模块边界与依赖图、动画每帧高层路径图
   - 渲染源码地图：Renderer / RenderCore / RHI / Engine 渲染入口 / Shaders 的职责边界
   - 动画源码地图：Engine Animation / AnimGraphRuntime / AnimationCore / AnimationBudgetAllocator / 插件目录的职责边界
   - 渲染读码路线（像素 / Mesh / Shader / Nanite 对象）
   - 动画读码路线（SkeletalMeshComponent / AnimBlueprint / AnimNode / Montage 或 BlendSpace）
   - 渲染与动画的交界（SkeletalMesh 如何走向渲染侧，事实与推理分清）
   - 与已有 `UE58_Source_Hierarchy_Orientation.md` 和 `UE58_Nanite_WorldPartition_HLOD_SourceMap.md` 的衔接
   - 阿卡姆剃刀检查（哪些故意不展开、为什么）
   - 局限性与潜在风险提示

## 6. 验收标准（Acceptance Criteria）

1. 主文档所有 Windows 路径均为完整绝对路径；可点击链接目标为正斜杠完整绝对路径并置于尖括号 `<...>` 中。
2. 主文档含 **至少 4 个 mermaid 图**（渲染边界图、渲染每帧路径图、动画边界图、动画每帧路径图）与 **两个 ASCII 总览图**（渲染链路、动画链路）。
3. 主文档分别给出渲染源码地图与动画源码地图，覆盖 §5 列出的全部模块/目录职责边界。
4. 主文档分别给出渲染与动画的读码路线，覆盖 §5 列出的全部起点对象。
5. 主文档显式说明渲染与动画的交界（`USkeletalMeshComponent` → `UAnimInstance`/`FAnimInstanceProxy` → 姿势 → `FSkeletalMeshObject` 蒙皮 → `FSkeletalMeshSceneProxy` → `Renderer`），并对每段标注事实/推理。
6. 主文档逐处标注 **事实依据** 与 **逻辑分析推理(无事实依据)**；复杂描述主谓宾完整。
7. 主文档说明"机器绝对路径只是本机定位路径，工程内引用应使用模块相对包含路径或 Unreal 路径 API"。
8. 主文档纠正任务原假设：AnimationBudgetAllocator 不在 `D:\UE\5.8.0r\Engine\Source\Runtime\AnimationBudgetAllocator`，而在插件目录 `D:\UE\5.8.0r\Engine\Plugins\Runtime\AnimationBudgetAllocator`。
9. 两份新文档与已有四份文档对同一路径/职责/模块边界/验证方式的描述不得互相矛盾（最后一步读取复核）。
10. 全程未读取范围外文件、未读取凭据/压缩包/生成产物、未修改任何引擎源码，未覆盖已有四份文档。

## 7. 局限性与潜在风险提示

- 本研究只看目录名、文件名、`.Build.cs` 与少量 `file:line` grep，**未通读任何实现**；每帧高层路径与交界数据流多为 **逻辑分析推理(无事实依据)**，需后续读 `.cpp` 验证后才能当结论用。
- 模块/目录/文件职责描述部分基于命名与目录结构推断，文件名与真实职责可能不完全一致。
- AnimationSamples 项目使用的 Chooser、Mover 等插件，其在引擎 `D:\UE\5.8.0r\Engine\Plugins` 内的确切落点本次 **未逐一验证**，主文档只做"存在与衔接"层面的提示，不声明确切路径。
- 文档中的绝对路径绑定本机 `D:\UE\5.8.0r` 布局，换机或换引擎版本即失效；这是为满足 thomas"完整绝对路径"硬性要求而做的取舍，与"代码/文档不硬编码绝对路径"的通用准则存在张力，已在主文档显式声明并给出工程内正确引用方式。
