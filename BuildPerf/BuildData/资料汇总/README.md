# UE 构建讨论资料汇总

文件路径：`D:\UE\Docs\Build\资料汇总\README.md`

## 目的

`D:\UE\Docs\Build\资料汇总` 汇集 6 个 Codex session 的讨论范围、关键结论、数据矩阵和原始证据入口。

6 个 session 分别是：

| Session ID | 标题 | 主题 |
|---|---|---|
| `019f2b0f-faab-7b81-b4da-c15b18739168` | `5.7.4r build.bat 分析` | `D:\UE\5.7.4r` 的 `BuildEditor.bat`、PCH/SharedPCH、XGE/IB 构建耗时与日志归档 |
| `019f2c30-202a-7582-82f1-18a286c013e8` | `检查 5.4.4r Build Config` | `D:\UE\5.4.4r` 的 BuildConfiguration、MSVC 14.38 固定、NoSharedPCH 与 SharedPCH 对照 |
| `019f2d8f-5402-78c1-98f0-dbc8c4929ac3` | `5.8.0r NoSharedPCH对比` | `D:\UE\5.8.0r` 的 NoSharedPCH 与 SharedPCH/XGE/IB 数据对比 |
| `019f2fbc-5fa5-7e50-9c32-c9e50ef63a5e` | `评估局域网带宽瓶颈` | 交换式局域网、10Gbps 升级、IncrediBuild 分布式编译流量瓶颈判断 |
| `019f2fac-c36a-7fa1-b40e-03cd789577b8` | `分析冷启动任务PCH依赖` | `C:\ProgramData\IncrediBuild\BuildData\BuildDB_42.db`、`C:\ProgramData\IncrediBuild\BuildData\BuildDB_43.db` 的冷启动 task 与 PCH/SharedPCH 归因，以及 UBA 单 action/多 tier 验证 |
| `019f34d3-e918-7f23-b84e-72e6524140e6` | `澄清 SharedPCH 与 CAS 机制` | UBT SharedPCH 失效判定、UBA 普通 CAS/custom CAS key 边界、IB/UBA 同 XML 与网卡流量实测 |

## 汇总文件

| 文件 | 用途 |
|---|---|
| [00_六个session范围总览.md](<D:/UE/Docs/Build/资料汇总/00_六个session范围总览.md>) | 6 个 session 的统一范围图和结论总表 |
| [01_5.7.4r_BuildBat分析.md](<D:/UE/Docs/Build/资料汇总/01_5.7.4r_BuildBat分析.md>) | `D:\UE\5.7.4r` 脚本、失败修复、PCH/SharedPCH 与构建耗时 |
| [02_5.4.4r_BuildConfig检查.md](<D:/UE/Docs/Build/资料汇总/02_5.4.4r_BuildConfig检查.md>) | `D:\UE\5.4.4r` 配置、编译错误修复、Build39/40/41 对照 |
| [03_5.8.0r_NoSharedPCH对比.md](<D:/UE/Docs/Build/资料汇总/03_5.8.0r_NoSharedPCH对比.md>) | `D:\UE\5.8.0r` Build42/43 与 SharedPCH 冷态分发税 |
| [04_局域网带宽瓶颈评估.md](<D:/UE/Docs/Build/资料汇总/04_局域网带宽瓶颈评估.md>) | 局域网吞吐、汇聚瓶颈和 10Gbps 升级判断 |
| [05_冷启动任务PCH依赖分析.md](<D:/UE/Docs/Build/资料汇总/05_冷启动任务PCH依赖分析.md>) | Build42/43 冷启动 remote task、PCH 依赖、PCH 3/4/5 体积模型与 UBA 抽样 |
| [06_SharedPCH与CAS机制澄清.md](<D:/UE/Docs/Build/资料汇总/06_SharedPCH与CAS机制澄清.md>) | SharedPCH action 失效、UBA CAS、IB/UBA 传输对照与机制边界 |
| [关键数据矩阵.csv](<D:/UE/Docs/Build/资料汇总/关键数据矩阵.csv>) | 跨版本构建核心指标表 |
| [数据来源索引.md](<D:/UE/Docs/Build/资料汇总/数据来源索引.md>) | 原始文档、CSV、JSON、日志、IncrediBuild 数据库索引 |
| `D:\UE\Docs\Build\资料汇总\原始材料副本` | 小型源文档、关键 CSV 和 JSON 的副本目录 |

## 主线关系

```text
D:\UE\5.7.4r
|  |
|  +--> D:\UE\5.7.4r\BuildEditor.bat 创建与修复
|  +--> Build32 / Build33 / Build34
|  +--> 证明 SharedPCH 冷态会显著放大远端任务耗时
|
+D:\UE\5.4.4r
|  |
|  +--> D:\UE\5.4.4r\Engine\Saved\UnrealBuildTool\BuildConfiguration.xml
|  +--> Build39 NoSharedPCH 成功
|  +--> Build40 SharedPCH enabled 失败
|  +--> Build41 SharedPCH 增量成功
|
+D:\UE\5.8.0r
|  |
|  +--> Build42 NoSharedPCH 成功
|  +--> Build43 SharedPCH enabled 完整执行但 IB 失败
|  +--> Build42/43 remote task 与 PCH 依赖归因
|  +--> SharedPCH.UnrealEd.Cpp20.h.pch 第 3/4/5 体积模型
|
+分布式执行机制层
|  |
|  +--> XGE / IncrediBuild 按自身机制分发输入
|  +--> UBA 使用内容 CAS 复用同 key 文件
|  +--> CAS 不能消除 SharedPCH 自身失效
|
+局域网与 IncrediBuild 解释层
   |
   +--> 单机 1Gbps 不是“共享总线”本身
   +--> 真正瓶颈是端到端最窄环节与多机汇聚
   +--> SharedPCH 大文件冷态分发会放大网络/缓存/调度压力
```

## 阿卡姆剃刀检查

- 这个汇总是否必须复制所有日志？不必须。`D:\UE\Docs\Build\资料汇总` 只复制小型高价值材料，大型日志与 `C:\ProgramData\IncrediBuild\BuildData` 数据库通过索引追溯。
- 这个汇总是否必须把每个 build action 逐条复写？不必须。`D:\UE\Docs\Build\资料汇总\关键数据矩阵.csv` 保留跨版本判断所需的核心指标。
- 这个汇总是否混淆事实与推断？不应混淆。文档中出现解释性判断时，使用 `逻辑分析推理(无事实依据)` 标注。

## 局限性与潜在风险

- `D:\UE\Docs\Build\资料汇总` 是二次整理，不替代原始日志、SQLite 数据库和构建产物。
- `D:\UE\Docs\Build\资料汇总\关键数据矩阵.csv` 的部分字段来自 session 中已汇总的统计结果；若要审计每个 task，需要回到 `C:\ProgramData\IncrediBuild\BuildData\BuildDB_32.db`、`C:\ProgramData\IncrediBuild\BuildData\BuildDB_33.db`、`C:\ProgramData\IncrediBuild\BuildData\BuildDB_34.db`、`C:\ProgramData\IncrediBuild\BuildData\BuildDB_39.db`、`C:\ProgramData\IncrediBuild\BuildData\BuildDB_40.db`、`C:\ProgramData\IncrediBuild\BuildData\BuildDB_41.db`、`C:\ProgramData\IncrediBuild\BuildData\BuildDB_42.db`、`C:\ProgramData\IncrediBuild\BuildData\BuildDB_43.db`、`C:\ProgramData\IncrediBuild\BuildData\BuildDB_56.db`、`C:\ProgramData\IncrediBuild\BuildData\BuildDB_57.db`、`C:\ProgramData\IncrediBuild\BuildData\BuildDB_58.db`、`C:\ProgramData\IncrediBuild\BuildData\BuildDB_59.db`、`C:\ProgramData\IncrediBuild\BuildData\BuildDB_60.db`、`C:\ProgramData\IncrediBuild\BuildData\BuildDB_61.db`、`C:\ProgramData\IncrediBuild\BuildData\BuildDB_62.db`、`C:\ProgramData\IncrediBuild\BuildData\BuildDB_63.db`。
