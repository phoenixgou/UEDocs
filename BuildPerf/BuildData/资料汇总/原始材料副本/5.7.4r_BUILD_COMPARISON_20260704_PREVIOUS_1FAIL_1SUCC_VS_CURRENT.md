# `D:\UE\5.7.4r` BuildEditor Comparison

## Scope

This report compares:

- Previous sequence: one failed `D:\UE\5.7.4r\BuildEditor.bat` run followed by one successful retry.
- Current sequence: one successful `D:\UE\5.7.4r\BuildEditor.bat` run stored under `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827`.

The previous failed run does not have a complete local log under `D:\UE\Docs\Build\5.7.4r`. Its numbers below come from thomas' terminal output in the conversation:

- `Result: Failed (OtherCompilationError)`
- `Total execution time: 1256.60 seconds`
- `Total time in XGE executor: 1047.06 seconds`
- `13 build system warning(s)`
- `BuildEditor` exit code `6`
- Last visible progress near `[4104/8228]`

## Build Flow

```text
Previous sequence
  |
  +--> Attempt A: failed
  |       |
  |       +--> result: Failed (OtherCompilationError)
  |       +--> UBT time: 1256.60 seconds
  |       +--> XGE time: 1047.06 seconds
  |       +--> warnings: 13
  |
  +--> Attempt B: succeeded
          |
          +--> result: Succeeded
          +--> UBT time: 203.19 seconds
          +--> XGE time: 195.14 seconds
          +--> actions: 4151

Current sequence
  |
  +--> Attempt 1: succeeded
          |
          +--> result: Succeeded
          +--> wall time: 1621.59 seconds
          +--> UBT time: 1613.81 seconds
          +--> XGE time: 1384.55 seconds
          +--> actions: 7783
```

## Evidence Files

| Scope | Evidence |
| --- | --- |
| Previous failed attempt | thomas' terminal output in this conversation; no complete local log found under `D:\UE\Docs\Build\5.7.4r` |
| Previous successful retry | `D:\UE\Docs\Build\5.7.4r\UnrealBuildTool_Log_latest.txt` |
| Previous successful retry JSON | `D:\UE\Docs\Build\5.7.4r\UnrealBuildTool_Log_latest.json` |
| Previous XGE task graph | `D:\UE\Docs\Build\5.7.4r\XGETasks_latest.xml` |
| Current stdout | `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827\Attempt1_BuildEditor_stdout.log` |
| Current stderr | `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827\Attempt1_BuildEditor_stderr.log` |
| Current UBT text log | `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827\Attempt1_UBT_Log.txt` |
| Current UBT JSON log | `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827\Attempt1_UBT_Log.json` |
| Current XGE task graph | `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827\Attempt1_XGETasks.xml` |
| Current metadata | `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827\Attempt1_meta.txt` |
| Current summary | `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827\Attempt1_BuildSummary.md` |
| Current PCH inventory | `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827\Attempt1_PCH_Files.csv` |

## Summary Table

| Metric | Previous failed attempt | Previous successful retry | Previous sequence total | Current successful attempt |
| --- | ---: | ---: | ---: | ---: |
| Final result | `Failed` | `Succeeded` | `Succeeded after retry` | `Succeeded` |
| BuildEditor exit code | `6` | not archived | not archived | `0` from process exit |
| UBT total seconds | `1256.60` | `203.19` | `1459.79` | `1613.81` |
| UBT total time | `20m 56.60s` | `3m 23.19s` | `24m 19.79s` | `26m 53.81s` |
| XGE seconds | `1047.06` | `195.14` | `1242.20` | `1384.55` |
| XGE time | `17m 27.06s` | `3m 15.14s` | `20m 42.20s` | `23m 04.55s` |
| Declared actions | `8228` | `4151` | not additive | `7783` |
| Parsed compile actions | not archived | `26` | not complete | `3551` |
| Parsed link actions | not archived | `4122` | not complete | `4135` |
| PCH actions | not archived | `0` | not complete | `12` |
| SharedPCH actions | not archived | `0` | not complete | `7` |
| PCH output files | not archived | not archived | not complete | `12` |
| PCH output size | not archived | not archived | not complete | `14049.93` MiB, about `13.72` GiB |
| Build system warnings | `13` | `0` | `13` | `9` |
| Error-like local log lines | not archived | `0` | not complete | `0` |

## Delta

| Comparison | UBT delta | XGE delta | Interpretation |
| --- | ---: | ---: | --- |
| Current attempt vs previous successful retry | `+1410.62` seconds | `+1189.41` seconds | Current attempt compiled far more from source. Previous retry mostly linked and completed leftover work. |
| Current attempt vs previous sequence total | `+154.02` seconds | `+142.35` seconds | Current one-shot success took slightly longer than the previous fail-plus-retry UBT total, but avoided a failed end state and second invocation. |
| Current warnings vs previous failed attempt | `-4` warnings | not applicable | Current run still had XGE helper warnings, but fewer than the failed attempt and no compilation failure. |

## Action Mix

| Metric | Previous successful retry | Current successful attempt |
| --- | ---: | ---: |
| Declared actions | `4151` | `7783` |
| Parsed command actions | `4150` | `7766` |
| Compile actions | `26` | `3551` |
| Link actions | `4122` | `4135` |
| Generate actions | `0` | `46` |
| Copy actions | `0` | `28` |
| WriteMetadata actions | `2` | `2` |
| PCH actions | `0` | `12` |
| SharedPCH actions | `0` | `7` |

`Parsed command actions` excludes `Complete` helper rows. UBT's declared action count is the authoritative total.

## Current PCH File Size

All `.pch` files found under `D:\UE\5.7.4r` have timestamps inside the current build window.

| Kind | Count | Total MiB | Total GiB | Average MiB | Max MiB | Min MiB |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| `ModulePCH` | 5 | 5105.31 | 4.99 | 1021.06 | 2007.75 | 232.06 |
| `SharedPCH` | 7 | 8944.62 | 8.74 | 1277.80 | 2307.43 | 387.56 |
| `Total` | 12 | 14049.93 | 13.72 | 1170.83 | 2307.43 | 232.06 |

Largest current `.pch` files:

| File | MiB |
| --- | ---: |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.RTTI.Cpp20.h.pch` | 2307.43 |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.Cpp20.h.pch` | 2304.06 |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\Engine\SharedPCH.Engine.Cpp20.h.pch` | 2158.50 |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\PCH.UnrealEd.h.pch` | 2007.75 |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\Engine\PCH.Engine.h.pch` | 1932.75 |

## Slowest Current Compile Actions

| Action | Seconds |
| --- | ---: |
| `Module.UnrealEd.22.cpp` | `1010.87` |
| `Module.ResonanceAudio.cpp` | `935.96` |
| `Module.Engine.38.cpp` | `910.00` |
| `Module.UnrealEd.8.cpp` | `888.06` |
| `Module.Sequencer.2.cpp` | `882.84` |

## Analysis

Current build succeeded on the first attempt. Previous sequence succeeded only after a failed attempt and a short retry.

The current attempt is more expensive than the previous successful retry because the previous retry reused artifacts produced before the failure. The current attempt includes PCH and SharedPCH compilation, thousands of compile actions, and a much larger compile workload.

The previous failed attempt's direct failure category was `OtherCompilationError`. Because the complete failed log is not present under `D:\UE\Docs\Build\5.7.4r`, this report cannot name the exact compiler diagnostic that caused the failure.

逻辑分析推理(无事实依据)：the previous retry's small compile count suggests it was an incremental completion after the failed attempt, while the current run behaved closer to a broader rebuild after cleaning or invalidating more build outputs.

## Occam Check

- Do we need to duplicate current logs in another directory? No. They already live under `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827`.
- Do we need to claim the previous failure root cause beyond `OtherCompilationError`? No. The complete failed log is not locally available.
- Does the comparison need editor-launch validation? No. The requested scope is build log archival and timing comparison.
- Is the main difference visible? Yes. Previous retry was mostly linking; current run performed broad compilation including PCH and SharedPCH.
- Is PCH size now visible? Yes. The current run generated `12` `.pch` files totaling about `13.72` GiB.
