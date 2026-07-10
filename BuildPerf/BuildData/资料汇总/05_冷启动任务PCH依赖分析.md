# 冷启动任务 PCH 依赖分析

文件路径：`D:\UE\Docs\Build\资料汇总\05_冷启动任务PCH依赖分析.md`

## Session 范围

Session ID：`019f2fac-c36a-7fa1-b40e-03cd789577b8`

标题：`分析冷启动任务PCH依赖`

这个 session 的主语是 `IB Build42` 与 `IB Build43` 的 cold remote `cl.exe` task；宾语是“主要慢 task 依赖哪些 PCH/SharedPCH，以及这些 PCH 为什么巨大”。讨论范围后来扩展到 UBA 单 action 对照、多 tier 抽样和文档落盘。

## 主线

```text
C:\ProgramData\IncrediBuild\BuildData\BuildDB_42.db
C:\ProgramData\IncrediBuild\BuildData\BuildDB_43.db
  |
  +--> build_task remote cl.exe rows
  |       |
  |       +--> task name / duration / machine
  |       +--> cl.exe @response-file
  |       +--> response-file /Yu /FI /Fp
  |
  +--> PCH / SharedPCH 归因
  |       |
  |       +--> Build42: -NoSharedPCH，主要是 ModulePCH 冷态分发
  |       +--> Build43: SharedPCH enabled，SharedPCH + ModulePCH 同时放大 cold remote task
  |
  +--> UBA 验证
          |
          +--> D:\UEProject\Docs\XGE_PCH_Probe\uba_pch_sampling_full_20260706_01\summary.csv
          +--> D:\UEProject\Docs\XGE_PCH_Probe\uba_sharedpch_coldwarm_20260706_151339\ColdWarmReport.md
```

## Build42 / Build43 事实

| Build | 配置 | remote `cl.exe` rows | 最大 remote `cl.exe` | 慢 task 判断 |
|---|---|---:|---:|---|
| Build42 | `-NoSharedPCH` | `3428` | `431.345s` | 主要是 ModulePCH / 编译输入冷态分发 |
| Build43 | SharedPCH enabled | `3281` | `720.406s` | SharedPCH 和 ModulePCH 都进入 cold helper 分发链 |

Build42 的 UBT 日志 `D:\UE\5.8.0r\Engine\Programs\UnrealBuildTool\Log-backup-2026.07.04-14.45.13.txt` 明确带 `-NoSharedPCH`。该构建没有 SharedPCH action，主要涉及：

- `PCH.Core.h.pch`
- `PCH.CoreUObject.h.pch`
- `PCH.Engine.h.pch`
- `PCH.UnrealEd.h.pch`
- `PCH.ResonanceAudio.h.pch`

Build43 的 top remote task 中同时出现：

- `SharedPCH.Engine.Cpp20.h.pch`
- `SharedPCH.UnrealEd.Cpp20.h.pch`
- `PCH.Engine.h.pch`
- `PCH.UnrealEd.h.pch`

逻辑分析推理(无事实依据)：Build43 比 Build42 更慢，不是因为 helper 在远端编译 SharedPCH；SharedPCH action 已在本机完成。真正拖慢 remote `cl.exe` 的主因是 helper cold 状态缺少 `.pch`，需要发起机上行分发输入。

## PCH 3/4/5 体积模型

落盘文档：`D:\UEProject\Docs\UE5.8_SharedPCH_ModulePCH_3_4_5_Analysis.md`

实验目录：`D:\UE\PCHAnalysis\UnrealEdCpp20`

核心结果：

| Bucket | 体积 | 占 `.pch` | 证据属性 |
|---|---:|---:|---|
| 第 3 部分：AST / 类型语义结构 | 约 `1444.23 MiB` | 约 `58.8%` | 逻辑分析推理(无事实依据)，来自分层 PCH 代理估算 |
| 第 4 部分：C++ 模板相关状态 | 约 `1011.33 MiB` | 约 `41.2%` | 逻辑分析推理(无事实依据)，来自分层 PCH 代理估算 |
| 第 5 部分：`/Z7` 对 `.pch` 本体 | `0 MiB` 级别 | `0%` 级别 | 实测对照 |
| 第 5 部分：`/Z7` 对 PCH 创建 `.obj` | 约 `70.61 MiB` | 不计入 `.pch` 本体 | 实测对照 |

这修正了一个关键误解：`SharedPCH.UnrealEd.Cpp20.h.pch` 的 `2.398 GiB` 主体不是 `/Z7` 调试信息直接塞进 `.pch`，而主要来自 include 闭包被 MSVC 前端保存成 AST、类型系统、UObject/Editor 语义和模板状态。

## UBA 单 action 与多 tier 验证

### CADKernel 单 action

对照 action：`D:\UEProject\Docs\XGE_PCH_Probe\uba_cadkernel_compile_probe_nosharedpch\Module.CADKernel.1.cpp.nosharedpch.rsp`

| 模式 | Remote run | SendCas Raw/Comp | SendTotal | 结论 |
|---|---:|---:|---:|---|
| SharedPCH 冷 | `11.3s` | `2.6GB / 348.9MB` | `349.8MB` | cold helper 需要拿巨大 SharedPCH |
| SharedPCH 热 | `7.6s` | 已缓存 | `822.7KB` | warm 可复用远端缓存 |
| NoSharedPCH 冷 | `6.5s` | `41.5MB / 14.5MB` | `15.4MB` | CADKernel 单 action 明显受益 |
| NoSharedPCH 热 | `6.1s` | 已缓存 | `876.3KB` | 热态差距缩小 |

### 多 tier 抽样

落盘数据：`D:\UEProject\Docs\XGE_PCH_Probe\uba_pch_sampling_full_20260706_01\summary.csv`

| tier | baseline cold | NoSharedPCH cold | 结论 |
|---|---:|---:|---|
| `SharedPCH.UnrealEd` / `CADKernel` | `11.9s` | `7.1s` | NoSharedPCH 明显更快 |
| `SharedPCH.Engine` / `BlueprintGraph` | `12.9s` | `17.2s` | NoSharedPCH 更慢 |
| `SharedPCH.Slate` / `Chaos` | `16.2s` | `17.2s` | NoSharedPCH 略慢 |
| `SharedPCH.CoreUObject` / `AssetRegistry` | `7.3s` | `9.8s` | NoSharedPCH 更慢 |
| `SharedPCH.Core` / `Analytics` | `2.2s` | `4.8s` | NoSharedPCH 更慢 |

结论：NoSharedPCH 能显著降低 cold SendTotal，但不等于全局更快。除 `SharedPCH.UnrealEd` 的 CADKernel 样本外，其它 tier 的头文件解析成本超过了传输收益。

## 判断边界

- Build42 的历史 `.obj.rsp` 已被后续构建覆盖，且没有找到 `.rsp.old` 快照；逐 task PCH 映射不能按字节级证据强证。
- Build43 的 PCH 映射更强，因为当前 `.rsp` 与 UBT/IB 证据链能形成 `task -> response file -> /Fp`。
- UBA 多 tier 抽样每个 tier 只抽 1 个代表 action，适合指导下一轮筛选，不适合直接作为全局策略。

局限性与潜在风险提示：本文件是 session 二次汇总；精确审计应回到 `C:\ProgramData\IncrediBuild\BuildData\BuildDB_42.db`、`C:\ProgramData\IncrediBuild\BuildData\BuildDB_43.db`、`D:\UEProject\Docs\UE5.8_SharedPCH_ModulePCH_3_4_5_Analysis.md` 和 `D:\UEProject\Docs\XGE_PCH_Probe\uba_pch_sampling_full_20260706_01\summary.csv`。
