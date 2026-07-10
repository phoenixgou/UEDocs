# UE 5.7.4r BuildEditor SharedPCH 构建报告

## 直接结论

- thomas 请求的第一次 `D:\UE\5.7.4r\BuildEditor.bat` 已执行完成；本次不需要第二次重试。
- `D:\UE\5.7.4r\BuildEditor.bat` 标准输出显示 `Result: Succeeded` 与 `[BuildEditor] DONE: build succeeded.`。
- 外层 wall time 为 `1561.326` 秒；UBT `Total execution time` 为 `1530.99` 秒；XGE executor time 为 `1307.58` 秒。
- UBT 使用 XGE 执行 `7783` 个 action，最多 `24` 个本地物理核心进程。
- UBT 日志显示 `Found 5 shared PCH headers`；最终磁盘扫描发现 `SharedPCH*.pch` 为 `7` 个，合计 `8944.62` MiB。
- 本次有 `22` 条 build system warning，均汇总为 `Disabling Helper (22 occurrences)`。
- 采集备注：`Start-Process` 记录到的 `ExitCode` 字段为空；Codex wrapper 返回码为 0，且 stdout 同时包含 UBT 成功与批处理成功行。

## 构建主线

```text
thomas 修改 build config，去掉 NoSharedPCH
  |
  v
D:\UE\5.7.4r\BuildEditor.bat
  |
  v
D:\UE\5.7.4r\Engine\Build\BatchFiles\Build.bat UnrealEditor Win64 Development -WaitMutex
  |
  v
D:\UE\5.7.4r\Engine\Programs\UnrealBuildTool\Log.txt: Found shared PCH headers
  |
  v
XGE executor: build actions
  |
  v
Result: Succeeded
```

## 耗时统计

| 指标 | 数值 |
| --- | ---: |
| Start | `2026-07-04T14:33:40.1683180+08:00` |
| End | `2026-07-04T14:59:41.4942621+08:00` |
| Wall time | `1561.326` 秒 |
| UBT Total execution time | `1530.99` 秒 |
| XGE executor time | `1307.58` 秒 |
| XGE action count | `7783` |
| Rebuild All | `1 succeeded, 0 failed, 0 skipped` |
| Result | `Succeeded` |
| Build system warnings | `22` |
| Disabling Helper occurrences | `22` |

## PCH 与 SharedPCH 对比

| Metric | Before Count | After Count | Delta Count | Before MiB | After MiB | Delta MiB |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| All .pch files | 0 | 12 | 12 | 0.00 | 14,049.93 | 14,049.93 |
| All SharedPCH* files | 484 | 539 | 55 | 0.08 | 9,195.42 | 9,195.34 |
| SharedPCH*.pch files | 0 | 7 | 7 | 0.00 | 8,944.62 | 8,944.62 |
| UnrealEditor scope .pch files | 0 | 11 | 11 | 0.00 | 13,817.87 | 13,817.87 |
| UnrealEditor scope SharedPCH* files | 18 | 73 | 55 | 0.00 | 9,195.35 | 9,195.35 |
| UnrealEditor scope SharedPCH*.pch files | 0 | 7 | 7 | 0.00 | 8,944.62 | 8,944.62 |

## SharedPCH*.pch 明细

| Full Path | MiB | LastWriteTime |
| --- | ---: | --- |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.RTTI.Cpp20.h.pch` | 2307.43 | 2026-07-04T14:38:36.1328402+08:00 |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.Cpp20.h.pch` | 2304.06 | 2026-07-04T14:38:36.1328402+08:00 |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\Engine\SharedPCH.Engine.Cpp20.h.pch` | 2158.5 | 2026-07-04T14:38:32.7724782+08:00 |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\Slate\SharedPCH.Slate.Cpp20.h.pch` | 867.25 | 2026-07-04T14:38:04.4544386+08:00 |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\CoreUObject\SharedPCH.CoreUObject.Cpp20.h.pch` | 531.62 | 2026-07-04T14:37:58.1024335+08:00 |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\Core\SharedPCH.Core.RTTI.Cpp20.h.pch` | 388.19 | 2026-07-04T14:37:54.2998738+08:00 |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\Core\SharedPCH.Core.Cpp20.h.pch` | 387.56 | 2026-07-04T14:37:54.0928360+08:00 |

## 全部 .pch 明细

| Full Path | MiB | IsSharedPCHPCH |
| --- | ---: | --- |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.RTTI.Cpp20.h.pch` | 2307.43 | True |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.Cpp20.h.pch` | 2304.06 | True |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\Engine\SharedPCH.Engine.Cpp20.h.pch` | 2158.5 | True |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\PCH.UnrealEd.h.pch` | 2007.75 | False |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\Engine\PCH.Engine.h.pch` | 1932.75 | False |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\Slate\SharedPCH.Slate.Cpp20.h.pch` | 867.25 | True |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\CoreUObject\PCH.CoreUObject.h.pch` | 545 | False |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\CoreUObject\SharedPCH.CoreUObject.Cpp20.h.pch` | 531.62 | True |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\Core\SharedPCH.Core.RTTI.Cpp20.h.pch` | 388.19 | True |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\Core\PCH.Core.h.pch` | 387.75 | False |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\Core\SharedPCH.Core.Cpp20.h.pch` | 387.56 | True |
| `D:\UE\5.7.4r\Engine\Plugins\Runtime\ResonanceAudio\Intermediate\Build\Win64\x64\UnrealEditor\Development\ResonanceAudio\PCH.ResonanceAudio.h.pch` | 232.06 | False |

## 证据文件

- `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-143236-BuildEditor-SharedPCHEnabled\Attempt1_stdout.log`
- `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-143236-BuildEditor-SharedPCHEnabled\Attempt1_stderr.log`
- `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-143236-BuildEditor-SharedPCHEnabled\Attempt1_UBT_Log.txt`
- `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-143236-BuildEditor-SharedPCHEnabled\Before_PCH_SharedPCH_Inventory.csv`
- `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-143236-BuildEditor-SharedPCHEnabled\After_PCH_SharedPCH_Inventory.csv`
- `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-143236-BuildEditor-SharedPCHEnabled\PCH_SharedPCH_Comparison.csv`
- `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-143236-BuildEditor-SharedPCHEnabled\Attempt1_ParsedMetrics.json`

## 逻辑分析推理(无事实依据)

- 本次 `SharedPCH*.pch` 从 0 个变为 7 个，且 UBT 日志出现 `Found 5 shared PCH headers`，因此去掉 `NoSharedPCH` 后 Shared PCH 机制已经重新参与本次 `UnrealEditor Win64 Development` 构建。
- `All SharedPCH* files` 的数量变化包含 `.obj`、`.cpp`、`.response`、`.json` 等伴生产物；判断真正的 Shared PCH 大文件，应优先看 `SharedPCH*.pch files`。
- 22 次 `Disabling Helper` warning 说明部分分布式 helper 被禁用，可能增加 wall time，但本次不是失败原因。

## 阿卡姆剃刀检查

- 本次是否需要修改 `D:\UE\5.7.4r` 源码：不需要。
- 本次是否需要第二次构建：不需要，因为第一次已经成功。
- 本次是否需要全树删除历史 `SharedPCH*`：不需要，因为用户要求是构建后统计生成情况。

## 局限性与潜在风险提示

- 本次统计反映 `2026-07-04T14:33:40+08:00` 到 `2026-07-04T14:59:41+08:00` 这一次构建；分布式 helper 状态、机器负载和缓存状态变化会影响后续可比性。
