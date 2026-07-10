# UE 5.7.4r 构建日志索引

## 范围

本目录归档 `D:\UE\5.7.4r` 的构建证据。

归档目录：`D:\UE\Docs\Build\5.7.4r`

## 最新 Editor 构建

- 命令来源：`D:\UE\5.7.4r\BuildEditor.bat`
- 本次归档目录：`D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827`
- 本次构建结果：`Succeeded`
- 本次墙钟耗时：`1621.59` 秒
- 本次 UBT 总执行耗时：`1613.81` 秒
- 本次 XGE 执行器耗时：`1384.55` 秒
- 本次 UBT action 数量：`7783`
- 本次 `.pch` 文件数量：`12`
- 本次 `.pch` 文件总大小：`14049.93` MiB，约 `13.72` GiB
- 本次构建报告：`D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827\Attempt1_BuildSummary.md`
- 本次 PCH 文件清单：`D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827\Attempt1_PCH_Files.csv`
- 上次与本次对比报告：`D:\UE\Docs\Build\5.7.4r\BUILD_COMPARISON_20260704_PREVIOUS_1FAIL_1SUCC_VS_CURRENT.md`

## 上次成功 Editor 构建快照

- 命令来源：`D:\UE\5.7.4r\BuildEditor.bat`
- UBT 命令：`D:\UE\5.7.4r\Engine\Binaries\DotNET\UnrealBuildTool\UnrealBuildTool.dll UnrealEditor Win64 Development -WaitMutex`
- 结果：`Succeeded`
- 总执行耗时：`203.19` 秒
- XGE 执行器耗时：`195.14` 秒
- XGE action 数量：`4151`
- UBT 报告的并发上限：`24` 个进程

## 归档文件

| 归档文件 | 原始来源 | 用途 |
| --- | --- | --- |
| `D:\UE\Docs\Build\5.7.4r\UnrealBuildTool_Log_latest.txt` | `D:\UE\5.7.4r\Engine\Programs\UnrealBuildTool\Log.txt` | 最新 UBT 构建文本日志。 |
| `D:\UE\Docs\Build\5.7.4r\UnrealBuildTool_Log_latest.json` | `D:\UE\5.7.4r\Engine\Programs\UnrealBuildTool\Log.json` | 最新 UBT 构建结构化日志。 |
| `D:\UE\Docs\Build\5.7.4r\UnrealBuildTool_Log_GPF.txt` | `D:\UE\5.7.4r\Engine\Programs\UnrealBuildTool\Log_GPF.txt` | Generate Project Files 文本日志。 |
| `D:\UE\Docs\Build\5.7.4r\UnrealBuildTool_Log_GPF.json` | `D:\UE\5.7.4r\Engine\Programs\UnrealBuildTool\Log_GPF.json` | Generate Project Files 结构化日志。 |
| `D:\UE\Docs\Build\5.7.4r\XGETasks_latest.xml` | `D:\UE\5.7.4r\Engine\Intermediate\Build\XGETasks.xml` | 最新构建的 XGE action 图。 |
| `D:\UE\Docs\Build\5.7.4r\UnrealBuildTool_Env_BuildConfiguration.xml` | `D:\UE\5.7.4r\Engine\Intermediate\Build\UnrealBuildTool.Env.BuildConfiguration.xml` | UBT 环境派生构建配置。 |
| `D:\UE\Docs\Build\5.7.4r\UnrealBuildTool_dep.csv` | `D:\UE\5.7.4r\Engine\Intermediate\Build\UnrealBuildTool.dep.csv` | UBT 依赖扫描输出。 |

## 构建形态

最新归档构建是一次增量补完构建：

```text
D:\UE\5.7.4r\BuildEditor.bat
  |
  +--> UnrealBuildTool
        |
        +--> XGE executor
              |
              +--> 26 compile actions
              +--> 4122 link actions
              +--> 2 metadata writes
```

逻辑分析推理(无事实依据)：最新成功构建复用了前一次失败完整构建产生的大量产物，然后完成剩余 Chaos 编译 action 和链接 action。

## 备注

- `D:\UE\5.7.4r\Engine\Programs\UnrealBuildTool\Log.txt` 会被后续 UBT 运行覆盖。
- `D:\UE\Docs\Build\5.7.4r` 下的文件是复制快照，不会影响后续构建。
