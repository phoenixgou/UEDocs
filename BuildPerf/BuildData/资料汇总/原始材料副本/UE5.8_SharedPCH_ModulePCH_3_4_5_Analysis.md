# UE5.8 SharedPCH / ModulePCH 体积构成与缩减分析

## 0. 结论摘要

本文记录 `D:\UE\5.8.0r` 中 `UnrealEditor Development` 配置下 SharedPCH 与 ModulePCH 的体积分析结果，重点回答两个问题：

- `D:\UE\5.8.0r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.Cpp20.h.pch` 内部第 3、4、5 部分各自贡献多少。
- 第 3、4 部分是否存在可行的减少路线。

核心结论：

| 分析对象 | 结论 |
| --- | --- |
| 第 3 部分：AST / 类型语义结构 | 代理估算(逻辑分析推理(无事实依据))约 `1444.23 MiB`，约占 `58.8%`。 |
| 第 4 部分：C++ 模板相关状态 | 代理估算(逻辑分析推理(无事实依据))约 `1011.33 MiB`，约占 `41.2%`。 |
| 第 5 部分：`/Z7` 调试信息对 `.pch` 本体的贡献 | 实测约 `0 MiB` 级别。 |
| 第 5 部分：`/Z7` 调试信息对 PCH 创建任务输出的贡献 | 实测主要体现在 `.obj` 净贡献，约 `70.61 MiB`。 |
| 第 3、4 部分是否可以减少 | 可以减少，但属于架构级 include/PCH 选择优化，不是单一编译参数优化。 |

逻辑分析推理(无事实依据)：如果目标是解释 IncrediBuild helper 的 cold start remote task 变慢，单个 `.pch` 的 `2.4 GiB` 主要来自第 3、4 部分；第 5 部分会增加 PCH 创建动作的 `.obj` 回传成本，但不是 `.pch` 本体膨胀的主因。

## 1. 证据范围与实验目录

### 1.1 原始 UE 构建证据

本文引用的原始 PCH 证据位于：

- `D:\UE\5.8.0r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development`
- `D:\UE\5.8.0r\Engine\Source\Runtime\Engine\Public\EngineSharedPCH.h`
- `D:\UE\5.8.0r\Engine\Source\Editor\UnrealEd\Public\UnrealEdSharedPCH.h`
- `D:\UE\5.8.0r\Engine\Source\Runtime\Engine\Engine.Build.cs`
- `D:\UE\5.8.0r\Engine\Source\Editor\UnrealEd\UnrealEd.Build.cs`
- `D:\UE\5.8.0r\Engine\Source\Programs\UnrealBuildTool\Platform\Windows\VCToolChain.cs`
- `D:\UE\5.8.0r\Engine\Source\Programs\UnrealBuildTool\Configuration\Rules\TargetRules.cs`
- `D:\UE\5.8.0r\Engine\Source\Programs\UnrealBuildTool\Configuration\UEBuildModuleCPP.cs`

### 1.2 实验输出目录

本文的对照实验输出位于：

- `D:\UE\PCHAnalysis\UnrealEdCpp20`
- `D:\UE\PCHAnalysis\UnrealEdCpp20\Layers`

这些实验文件没有修改 `D:\UE\5.8.0r` 的源码。

### 1.3 编译器版本

实验显式使用：

- `C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.44.35207\bin\Hostx64\x64\cl.exe`

原因是原始 `.rsp` 中外部 include 指向：

- `C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.44.35207\INCLUDE`

如果使用默认 `VsDevCmd.bat`，本机默认会选择：

- `C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.38.33130\bin\Hostx64\x64\cl.exe`

该版本会与 `14.44.35207` STL 头文件不匹配，并触发 `STL1001: Unexpected compiler version`。

## 2. 原始 PCH 规模

`D:\UE\5.8.0r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.Cpp20.h.pch` 的原始规模如下：

| 指标 | 数值 |
| --- | ---: |
| `.pch` 体积 | `2.398 GiB` |
| 传递 include 文件数 | `2339` |
| 传递 include 原始文件体积合计 | `29.84 MiB` |

这说明原始源码文本并不是 `.pch` 巨大的唯一原因。MSVC 把 include 闭包解析成可恢复的编译器内部状态后，会保存大量类型系统、声明、宏、模板和语义状态。

```text
少量 PCH wrapper
  |
  +--> 传递 include 闭包
  |       |
  |       +--> Core / CoreUObject / Slate / Engine / UnrealEd
  |
  +--> MSVC 前端语义状态
  |       |
  |       +--> AST / 类型系统 / UObject / Editor 类型
  |       +--> C++ 模板 / 容器 / Delegate / Slate 泛型状态
  |
  +--> .pch 二进制状态文件
```

## 3. PCH 第 5 部分：调试信息贡献

### 3.1 实验方法

本文从原始响应文件复制出三个对照：

- `D:\UE\PCHAnalysis\UnrealEdCpp20\full_z7.rsp`
- `D:\UE\PCHAnalysis\UnrealEdCpp20\no_debug.rsp`
- `D:\UE\PCHAnalysis\UnrealEdCpp20\zi_pdb.rsp`

三个对照使用同一份输入：

- `D:\UE\5.8.0r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.Cpp20.cpp`
- `D:\UE\5.8.0r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.Cpp20.h`

三个对照只改变调试信息策略：

| 对照 | 调试策略 |
| --- | --- |
| `full_z7` | 保留 `/Z7`。 |
| `no_debug` | 移除 `/Z7`。 |
| `zi_pdb` | 把 `/Z7` 改为 `/Zi`，并通过 `/Fd` 输出独立 PDB。 |

### 3.2 实测结果

| 对照 | `.pch` | `.obj` | `.pdb` |
| --- | ---: | ---: | ---: |
| `D:\UE\PCHAnalysis\UnrealEdCpp20\full_z7.pch` | `2455.50 MiB` | `70.66 MiB` | 无 |
| `D:\UE\PCHAnalysis\UnrealEdCpp20\no_debug.pch` | `2455.62 MiB` | `0.05 MiB` | 无 |
| `D:\UE\PCHAnalysis\UnrealEdCpp20\zi_pdb.pch` | `2395.62 MiB` | `8.01 MiB` | `70.51 MiB` |

推导结果：

| 指标 | 结果 |
| --- | ---: |
| `/Z7` 对 `.pch` 本体贡献 | `full_z7.pch - no_debug.pch = -0.12 MiB`，可视为 `0 MiB` 级别。 |
| `/Z7` 对 `.obj` 净贡献 | `full_z7.obj - no_debug.obj = 70.66 MiB - 0.05 MiB = 70.61 MiB`。 |
| `/Z7` 对 `.pch + .obj` 输出贡献 | `(full_z7.pch + full_z7.obj) - (no_debug.pch + no_debug.obj) = (2455.50 MiB + 70.66 MiB) - (2455.62 MiB + 0.05 MiB) = 70.49 MiB`。 |
| `/Zi` 外部调试信息载荷 | `zi_pdb.obj + zi_pdb.pdb = 78.52 MiB`。 |

`70.61 MiB` 是纯 `.obj` 净贡献口径；`70.49 MiB` 是 `.pch + .obj` 合并口径。两者差异来自 `D:\UE\PCHAnalysis\UnrealEdCpp20\no_debug.pch` 反而比 `D:\UE\PCHAnalysis\UnrealEdCpp20\full_z7.pch` 大 `0.12 MiB` 的测量和布局差异，本文按 `0 MiB` 级处理。`D:\UE\PCHAnalysis\UnrealEdCpp20\zi_pdb.pch` 比 `D:\UE\PCHAnalysis\UnrealEdCpp20\full_z7.pch` 小 `59.88 MiB`，说明 MSVC 的 `/Zi` 与 `/Z7` 会改变 `.pch`、`.obj`、`.pdb` 之间的数据布局，不能把 `/Zi` 的 `.pch` 体积下降直接解释为语义状态减少。

结论：PCH 第 5 部分不是 `SharedPCH.UnrealEd.Cpp20.h.pch` 本体膨胀的主因。PCH 第 5 部分主要体现在 PCH 创建动作旁边的 `.obj` 或 `.pdb`。

## 4. PCH 第 3 与第 4 部分：分层 PCH 代理实验

### 4.1 实验方法

MSVC `.pch` 是私有二进制格式，不能稳定地逐字节拆出 AST、模板和调试信息。因此第 3、4 部分采用分层 PCH 增量 + 源码标记模型做代理估算。

实验分层文件位于：

- `D:\UE\PCHAnalysis\UnrealEdCpp20\Layers\Layer.empty.h`
- `D:\UE\PCHAnalysis\UnrealEdCpp20\Layers\Layer.core.h`
- `D:\UE\PCHAnalysis\UnrealEdCpp20\Layers\Layer.coreuobject.h`
- `D:\UE\PCHAnalysis\UnrealEdCpp20\Layers\Layer.slate.h`
- `D:\UE\PCHAnalysis\UnrealEdCpp20\Layers\Layer.engine.h`
- `D:\UE\PCHAnalysis\UnrealEdCpp20\Layers\Layer.unrealed.h`

对应输出：

- `D:\UE\PCHAnalysis\UnrealEdCpp20\layer_empty.pch`
- `D:\UE\PCHAnalysis\UnrealEdCpp20\layer_core.pch`
- `D:\UE\PCHAnalysis\UnrealEdCpp20\layer_coreuobject.pch`
- `D:\UE\PCHAnalysis\UnrealEdCpp20\layer_slate.pch`
- `D:\UE\PCHAnalysis\UnrealEdCpp20\layer_engine.pch`
- `D:\UE\PCHAnalysis\UnrealEdCpp20\layer_unrealed.pch`

分层关系：

```text
empty
  |
  +--> CoreSharedPCH.h
        |
        +--> CoreUObjectSharedPCH.h
              |
              +--> SlateSharedPCH.h
                    |
                    +--> EngineSharedPCH.h
                          |
                          +--> UnrealEdSharedPCH.h
```

注意：这是代理实验关系，不代表 UnrealBuildTool 在真实构建中一定以该线性链条组织所有 SharedPCH。真实构建由 UBT 根据模块依赖和编译环境选择 SharedPCH template 与 variant。

### 4.2 分层 PCH 结果

| 层 | include 数 | 源文件体积 | PCH 体积 |
| --- | ---: | ---: | ---: |
| `empty` | `2` | `0.02 MiB` | `3.94 MiB` |
| `core` | `716` | `10.10 MiB` | `370.94 MiB` |
| `coreuobject` | `864` | `12.38 MiB` | `512.25 MiB` |
| `slate` | `1132` | `15.10 MiB` | `874.56 MiB` |
| `engine` | `2201` | `28.42 MiB` | `2224.31 MiB` |
| `unrealed` | `2339` | `29.84 MiB` | `2455.56 MiB` |

### 4.3 第 3/4 估算模型

第 3 部分表示：

- AST 节点。
- 类型系统。
- class / struct / enum / UObject / UCLASS / USTRUCT / UPROPERTY / UFUNCTION 等语义结构。

第 4 部分表示：

- `template <...>`。
- `typename`。
- `TArray<T>`、`TMap<K,V>`、`TSet<T>`。
- `TSharedPtr<T>`、`TFunction<T>`、`TOptional<T>`。
- Slate attribute / delegate / 泛型 helper。
- 标准库模板相关符号。

本文按每层新增 include 文件中的模板标记行数与语义标记行数，把该层 PCH 增量分摊到第 3 和第 4。

逻辑分析推理(无事实依据)：该模型无法代表 MSVC `.pch` 内部真实字节布局，但它能够提供一组可复验、可解释、方向稳定的估算。

### 4.4 第 3/4 分层估算

| 层 | PCH 增量 | 估算第 3 部分 | 估算第 4 部分 | 证据摘要 |
| --- | ---: | ---: | ---: | --- |
| `empty/base` | `3.94 MiB` | `3.94 MiB` | `0 MiB` | empty wrapper + SharedDefinitions |
| `core` | `367.00 MiB` | `121.84 MiB` | `245.16 MiB` | 新增 `705` 个文件，模板标记 `14865` 行，语义标记 `7372` 行 |
| `coreuobject` | `141.31 MiB` | `90.72 MiB` | `50.59 MiB` | 新增 `145` 个文件，模板标记 `1979` 行，语义标记 `3556` 行 |
| `slate` | `362.31 MiB` | `160.50 MiB` | `201.81 MiB` | 新增 `269` 个文件，模板标记 `3763` 行，语义标记 `2988` 行 |
| `engine` | `1349.75 MiB` | `917.83 MiB` | `431.92 MiB` | 新增 `1041` 个文件，模板标记 `11236` 行，语义标记 `23879` 行 |
| `unrealed` | `231.25 MiB` | `149.39 MiB` | `81.86 MiB` | 新增 `139` 个文件，模板标记 `1481` 行，语义标记 `2700` 行 |

汇总：

| Bucket | 体积 | 占 `D:\UE\PCHAnalysis\UnrealEdCpp20\layer_unrealed.pch` 比例 |
| --- | ---: | ---: |
| 第 3 部分：AST / 类型语义结构 | `1444.23 MiB` | `58.8%` |
| 第 4 部分：C++ 模板相关状态 | `1011.33 MiB` | `41.2%` |

上表是代理估算(逻辑分析推理(无事实依据))，不是 MSVC 内部精确尺寸映射。分层求和基于 `D:\UE\PCHAnalysis\UnrealEdCpp20\layer_unrealed.pch`，而 `D:\UE\PCHAnalysis\UnrealEdCpp20\no_debug.pch` 只作为同量级参照；两者差 `0.06 MiB`，远小于 `2.4 GiB` 主体。第 3、4 部分分桶存在 `0.01 MiB` 级舍入分配，目的是让两桶总和守恒为 `2455.56 MiB`。

## 5. PCH 第 3、4 部分能否减少

答案：可以减少，但减少第 3、4 部分是架构级 include/PCH 组织优化，不是一个简单编译参数。

### 5.1 可减少路径

```text
减少第 3/4 部分
  |
  +--> A. 减少 SharedPCH 输入头文件
  |
  +--> B. 让更多模块不要选到巨大的 UnrealEd / Engine SharedPCH
  |
  +--> C. 减少 RTTI / Exceptions / CppStandard 等编译环境差异导致的变体复制
```

### 5.2 预计收益上限

| 动作 | 可影响体积 | 可行性 |
| --- | ---: | --- |
| 缩小 `D:\UE\5.8.0r\Engine\Source\Editor\UnrealEd\Public\UnrealEdSharedPCH.h` | 最高约 `231 MiB` | 中等，可实验 |
| 让模块避开 UnrealEd SharedPCH，改用 Engine / Slate / CoreUObject | 约 `231 MiB`（改用 Engine）到约 `1943 MiB`（改用 CoreUObject）的单 PCH 选择差异 | 较可行 |
| 让模块避开 Engine SharedPCH，改用 Slate / CoreUObject / Core | 约 `1350 MiB`（改用 Slate）到约 `1853 MiB`（改用 Core）的单 PCH 选择差异 | 高风险，只适合不需要 Engine 的模块 |
| 移除 Slate 层 | 最高约 `362 MiB` | 只适合非 UI / 非 editor 模块 |
| 减少 RTTI / Exceptions / CppStandard 差异导致的 SharedPCH 变体 | 可省重复的 `2.4 GiB` 级文件 | 值得优先查 |
| 全局 `-NoSharedPCH` | 减少 SharedPCH cold distribution | 会增加前端编译 CPU，属于成本转移 |

表中的收益是单个 `.pch` 选择差异，不等同于整体构建净收益；模块改用更小 PCH 也可能提高前端编译 CPU 成本，需要结合 `*.obj.rsp` 的 `/Fp` 使用矩阵和实际构建耗时验证。

### 5.3 优先路线

优先级 1：统计哪些模块使用 `D:\UE\5.8.0r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.Cpp20.h.pch`。

做法：扫描 `D:\UE\5.8.0r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development` 下所有 `*.obj.rsp` 的 `/Fp` 参数。

优先级 2：审查 SharedPCH variant。

目标：减少 `D:\UE\5.8.0r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.RTTI.Cpp20.h.pch` 这类变体复制，优先量化 `2.4 GiB x N` 的重复状态。

优先级 3：审查这些模块的 `.Build.cs` 依赖。

目标：把不必要的 `UnrealEd`、`BlueprintGraph`、`GraphEditor`、`KismetCompiler` 等依赖从 public dependency 移到 private dependency，或拆到 editor-only 子模块。

优先级 4：审查 public header 中的重 include。

目标：能前置声明的 class / struct 使用前置声明，把 include 移到 `.cpp`。

优先级 5：做 slim PCH 对照实验。

目标：只在 `D:\UE\PCHAnalysis` 中复制实验，不直接修改 `D:\UE\5.8.0r` 的真实引擎头文件。

## 6. 对 Build42 / Build43 与 IncrediBuild 的含义

逻辑分析推理(无事实依据)：如果 IncrediBuild remote helper 需要获取或校验 `.pch`，那么 helper 冷启动的主要输入负担来自第 3、4 部分产生的 `2.4 GiB` `.pch`。如果 remote task 还需要回传 PCH 创建动作的 `.obj`，那么第 5 部分会额外增加约 `70 MiB` 的回传负担。

更准确的因果图：

```text
SharedPCH.UnrealEd.Cpp20.h.pch 巨大
  |
  +--> 第 3 部分：AST / 类型系统 / UObject / Editor 类语义   约 58.8%
  |
  +--> 第 4 部分：模板 / 容器 / Slate / delegate 状态          约 41.2%
  |
  +--> 第 5 部分：/Z7 调试信息对 .pch 本体贡献                 约 0%

PCH 创建动作输出变大
  |
  +--> /Z7 调试信息主要进入 .obj                               约 70.61 MiB
```

因此，若优化目标是 IB cold start：

1. 优先减少 remote task 对 `D:\UE\5.8.0r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.Cpp20.h.pch` 的依赖。
2. 优先减少 SharedPCH 变体数量。
3. 再考虑瘦身 `D:\UE\5.8.0r\Engine\Source\Editor\UnrealEd\Public\UnrealEdSharedPCH.h` 或 `D:\UE\5.8.0r\Engine\Source\Runtime\Engine\Public\EngineSharedPCH.h`。

## 7. 阿卡姆剃刀检查

- 是否必须直接修改 `D:\UE\5.8.0r\Engine\Source\Runtime\Engine\Public\EngineSharedPCH.h`？不必须。先统计模块选择与变体复制更稳。
- 是否必须精确解析 MSVC `.pch` 私有格式？不必须。第 5 部分可以通过控制变量实验实测，第 3/4 可以用分层 PCH 代理估计。
- 是否必须全局关闭 SharedPCH？不必须。全局 `-NoSharedPCH` 是成本转移，可能把分发成本换成更多前端编译 CPU。
- 是否应该优先处理 `SharedPCH.UnrealEd.RTTI.Cpp20.h.pch` 这类变体？应该。它会复制大量第 3、4 状态。

## 8. Codemaker 审核迭代记录

| 轮次 | 审核者配置 | 审核结论 | 已落盘修正 |
| --- | --- | --- | --- |
| 第一轮 | `netease-codemaker/claude-opus-4-8`，`max`，`DEBUG` 日志 | 需修正。第 3、4 部分需要明确代理估算属性；第 5 部分需要拆开 `.obj` 净贡献和 `.pch + .obj` 合并口径；分母差异需要写清；`/Zi`、`/Z7` 布局差异需要写清；SharedPCH variant 审查优先级需要提前。 | 已补充 `70.61 MiB` 与 `70.49 MiB` 的公式和口径；已标注代理估算(逻辑分析推理(无事实依据))；已说明 `D:\UE\PCHAnalysis\UnrealEdCpp20\layer_unrealed.pch` 与 `D:\UE\PCHAnalysis\UnrealEdCpp20\no_debug.pch` 差 `0.06 MiB`；已说明 `/Zi`、`/Z7` 改变 `.pch`、`.obj`、`.pdb` 数据布局；已把 SharedPCH variant 审查提前到优先级 2。 |
| 第二轮 | `netease-codemaker/claude-opus-4-8`，`max`，`DEBUG` 日志 | 需修正。第一轮修正点已通过；收益上限表中「改用 Engine / Slate / CoreUObject」和「改用 Slate / CoreUObject / Core」用相邻层收益当成最高收益，会低估优化上限。 | 已把收益改为区间口径：避开 UnrealEd SharedPCH 约 `231 MiB` 到约 `1943 MiB`；避开 Engine SharedPCH 约 `1350 MiB` 到约 `1853 MiB`；已精简第 4 节汇总表表头。 |

## 9. 局限性与潜在风险提示

第 5 部分是基于 `D:\UE\PCHAnalysis\UnrealEdCpp20\full_z7.rsp`、`D:\UE\PCHAnalysis\UnrealEdCpp20\no_debug.rsp`、`D:\UE\PCHAnalysis\UnrealEdCpp20\zi_pdb.rsp` 的实测对照。第 3、4 部分是基于分层 PCH 增量和源码标记的代理估算，因为 MSVC `.pch` 没有公开稳定的内部 size map，无法逐字节精确标注 AST 与模板状态边界。任何瘦身 SharedPCH 的真实改动都必须用完整 UnrealEditor 构建验证，否则容易把 PCH 分发成本换成更大的编译失败风险或前端编译成本。
