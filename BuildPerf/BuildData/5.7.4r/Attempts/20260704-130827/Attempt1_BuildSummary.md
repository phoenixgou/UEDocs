# `D:\UE\5.7.4r` BuildEditor Attempt 1 Summary

## Result

- Command: `D:\UE\5.7.4r\BuildEditor.bat`
- Target: `UnrealEditor Win64 Development`
- Result: `Succeeded`
- Retry: not executed because attempt 1 succeeded.
- Rebuild summary: `Rebuild All: 1 succeeded, 0 failed, 0 skipped`

## Time

- Start: `2026-07-04 13:08:27 +08:00`
- End: `2026-07-04 13:35:28 +08:00`
- Wall time: `1621.59` seconds, about `27m 01.59s`
- UBT total execution time: `1613.81` seconds, about `26m 53.81s`
- XGE executor time: `1384.55` seconds, about `23m 04.55s`
- Non-XGE UBT time: `229.26` seconds, about `3m 49.26s`

## Actions

- UBT declared action count: `7783`
- Parsed non-`Complete` action-style log lines: `7766`
- Parsing note: `[n/7783]` is a progress counter in the log output, not a stable unique action id. The final action count should use UBT's declared `7783`.

| Kind | Count | Sum seconds | Sum minutes | Average seconds | Max seconds |
| --- | ---: | ---: | ---: | ---: | ---: |
| `Compile` | 3551 | 383439.77 | 6390.66 | 107.98 | 1010.87 |
| `Link` | 4135 | 4778.18 | 79.64 | 1.16 | 16.32 |
| `Generate` | 46 | 326.48 | 5.44 | 7.10 | 34.96 |
| `Copy` | 28 | 7.49 | 0.12 | 0.27 | 0.40 |
| `WriteMetadata` | 2 | 0.86 | 0.01 | 0.43 | 0.57 |
| `GenerateTLH` | 2 | 2.60 | 0.04 | 1.30 | 1.51 |
| `Resource` | 2 | 5.26 | 0.09 | 2.63 | 2.75 |

`Sum seconds` is summed per logged action and therefore represents parallel action time, not wall-clock time.

## PCH

- PCH-related compile actions: `12`
- PCH-related summed action time: `322.78` seconds, about `5.38` minutes
- SharedPCH compile actions: `7`
- SharedPCH summed action time: `202.74` seconds, about `3.38` minutes
- PCH output file detail: `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827\Attempt1_PCH_Files.csv`

| Action | Seconds |
| --- | ---: |
| `SharedPCH.UnrealEd.RTTI.Cpp20.cpp` | 55.07 |
| `SharedPCH.UnrealEd.Cpp20.cpp` | 54.17 |
| `SharedPCH.Engine.Cpp20.cpp` | 50.68 |
| `PCH.UnrealEd.cpp` | 50.46 |
| `PCH.Engine.cpp` | 47.36 |
| `SharedPCH.Slate.Cpp20.cpp` | 18.28 |
| `SharedPCH.CoreUObject.Cpp20.cpp` | 11.04 |
| `PCH.CoreUObject.cpp` | 11.01 |
| `PCH.Core.cpp` | 6.89 |
| `SharedPCH.Core.Cpp20.cpp` | 6.82 |
| `SharedPCH.Core.RTTI.Cpp20.cpp` | 6.68 |
| `PCH.ResonanceAudio.cpp` | 4.32 |

## PCH File Outputs

- Total `.pch` files found under `D:\UE\5.7.4r`: `12`
- Total `.pch` size: `14049.93` MiB, about `13.72` GiB
- Current build timestamp window: `2026-07-04 13:08:27 +08:00` to `2026-07-04 13:35:28 +08:00`
- Observed `.pch` write window: `2026-07-04 13:12:47 +08:00` to `2026-07-04 13:13:30 +08:00`

| Kind | Count | Total MiB | Total GiB | Average MiB | Max MiB | Min MiB |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| `ModulePCH` | 5 | 5105.31 | 4.99 | 1021.06 | 2007.75 | 232.06 |
| `SharedPCH` | 7 | 8944.62 | 8.74 | 1277.80 | 2307.43 | 387.56 |

Largest `.pch` files:

| File | MiB |
| --- | ---: |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.RTTI.Cpp20.h.pch` | 2307.43 |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.Cpp20.h.pch` | 2304.06 |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\Engine\SharedPCH.Engine.Cpp20.h.pch` | 2158.50 |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\PCH.UnrealEd.h.pch` | 2007.75 |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\Engine\PCH.Engine.h.pch` | 1932.75 |

## Slowest Logged Compile Actions

| Action | Seconds |
| --- | ---: |
| `Module.UnrealEd.22.cpp` | 1010.87 |
| `Module.ResonanceAudio.cpp` | 935.96 |
| `Module.Engine.38.cpp` | 910.00 |
| `Module.UnrealEd.8.cpp` | 888.06 |
| `Module.Sequencer.2.cpp` | 882.84 |
| `Module.InterchangeFbxParser.cpp` | 880.29 |
| `Module.Sequencer.5.cpp` | 879.64 |
| `Module.MovieSceneTools.2.cpp` | 876.73 |
| `Module.Sequencer.7.cpp` | 875.15 |
| `Module.MovieSceneTracks.3.cpp` | 855.28 |

## Warnings And Errors

- Build system warnings: `9`
- Warning type: `Disabling Helper`
- Error-like lines matched by the parser: `0`

The `Disabling Helper` warnings mean XGE helpers were disabled when helper machines were no longer idle. This did not fail the build.

## Logs

- Stdout log: `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827\Attempt1_BuildEditor_stdout.log`
- Stderr log: `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827\Attempt1_BuildEditor_stderr.log`
- UBT text log: `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827\Attempt1_UBT_Log.txt`
- UBT JSON log: `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827\Attempt1_UBT_Log.json`
- Metadata file: `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827\Attempt1_meta.txt`
- XGE task graph: `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827\Attempt1_XGETasks.xml`
- UBT dependency scan: `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827\Attempt1_UnrealBuildTool_dep.csv`
- UBT derived build configuration: `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827\Attempt1_UnrealBuildTool_Env_BuildConfiguration.xml`
- PCH file inventory: `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-130827\Attempt1_PCH_Files.csv`

## Output Cross-Check

- `D:\UE\5.7.4r\Engine\Binaries\Win64\UnrealEditor.exe` exists.
- `D:\UE\5.7.4r\Engine\Binaries\Win64\UnrealEditor.modules` exists.
- `D:\UE\5.7.4r\Engine\Binaries\Win64\UnrealEditor.version` exists.
- `D:\UE\5.7.4r\Engine\Binaries\Win64\UnrealEditor.target` exists.

## Occam Check

- Is a second build attempt needed? No. Attempt 1 succeeded.
- Is the main bottleneck visible? Yes. Compilation dominates logged parallel action time.
- Is `Disabling Helper` the failure cause? No. It is a warning and the final result is `Succeeded`.
- Is editor launch verified? No. This report verifies build logs and output files only.
