# `D:\UE\5.7.4r` Build.bat Clean PCH Report

## Scope

- Command: `D:\UE\5.7.4r\Engine\Build\BatchFiles\Build.bat UnrealEditor Win64 Development -Clean -WaitMutex`
- Working directory: `D:\UE\5.7.4r`
- Start: `2026-07-04 14:28:13 +08:00`
- End: `2026-07-04 14:28:23 +08:00`
- Wall time: `10.083` seconds
- Wrapper result: exited with code `0`
- Report directory: `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-142751-BuildCleanPCHCheck`

## Clean Output

The clean stdout contains:

```text
Using bundled DotNet SDK version: 8.0.412 win-x64
Running UnrealBuildTool: dotnet "..\..\Engine\Binaries\DotNET\UnrealBuildTool\UnrealBuildTool.dll" UnrealEditor Win64 Development -Clean -WaitMutex
Cleaning UnrealEditor binaries...
```

`D:\UE\5.7.4r\Engine\Programs\UnrealBuildTool\Log.txt` was not updated by this clean run. Its timestamp remained `2026-07-04 14:02:08 +08:00`, earlier than the clean start time. The copied files are therefore marked stale:

- `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-142751-BuildCleanPCHCheck\Stale_UBT_Log_NotUpdatedByClean.txt`
- `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-142751-BuildCleanPCHCheck\Stale_UBT_Log_NotUpdatedByClean.json`

## Summary

| Scope | Before count | Before MiB | After count | After MiB | Delta count | Delta MiB |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| All `D:\UE\5.7.4r` `.pch` files | 5 | 5105.31 | 0 | 0.00 | -5 | -5105.31 |
| UnrealEditor-scope `.pch` files | 4 | 4873.25 | 0 | 0.00 | -4 | -4873.25 |
| All `D:\UE\5.7.4r` `SharedPCH*` files | 484 | 0.08 | 484 | 0.08 | 0 | 0.00 |
| UnrealEditor-scope `SharedPCH*` files | 0 | 0.00 | 0 | 0.00 | 0 | 0.00 |

UnrealEditor scope:

`D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development`

## Before Clean PCH Files

| File | Kind | MiB | LastWriteTime |
| --- | --- | ---: | --- |
| `D:\UE\5.7.4r\Engine\Plugins\Runtime\ResonanceAudio\Intermediate\Build\Win64\x64\UnrealEditor\Development\ResonanceAudio\PCH.ResonanceAudio.h.pch` | `ModulePCH` | 232.06 | `2026-07-04 13:12:47 +08:00` |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\Core\PCH.Core.h.pch` | `ModulePCH` | 387.75 | `2026-07-04 13:12:50 +08:00` |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\CoreUObject\PCH.CoreUObject.h.pch` | `ModulePCH` | 545.00 | `2026-07-04 13:12:53 +08:00` |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\Engine\PCH.Engine.h.pch` | `ModulePCH` | 1932.75 | `2026-07-04 13:13:23 +08:00` |
| `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\PCH.UnrealEd.h.pch` | `ModulePCH` | 2007.75 | `2026-07-04 13:13:25 +08:00` |

## After Clean PCH Files

No `.pch` files remain under `D:\UE\5.7.4r`.

No `SharedPCH*` files remain under `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development`.

The remaining `484` `SharedPCH*` files under `D:\UE\5.7.4r` are historical non-`.pch` files outside the UnrealEditor clean scope, totaling only `0.08` MiB.

## Evidence Files

- Clean stdout: `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-142751-BuildCleanPCHCheck\BuildBat_Clean_UnrealEditor_stdout.log`
- Clean stderr: `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-142751-BuildCleanPCHCheck\BuildBat_Clean_UnrealEditor_stderr.log`
- Clean metadata: `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-142751-BuildCleanPCHCheck\BuildBat_Clean_UnrealEditor_meta.txt`
- Before all PCH inventory: `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-142751-BuildCleanPCHCheck\BeforeClean_AllPCH.csv`
- After all PCH inventory: `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-142751-BuildCleanPCHCheck\AfterClean_AllPCH.csv`
- Before all SharedPCH inventory: `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-142751-BuildCleanPCHCheck\BeforeClean_AllSharedPCHStar.csv`
- After all SharedPCH inventory: `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-142751-BuildCleanPCHCheck\AfterClean_AllSharedPCHStar.csv`
- Before summary: `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-142751-BuildCleanPCHCheck\BeforeClean_Summary.csv`
- After summary: `D:\UE\Docs\Build\5.7.4r\Attempts\20260704-142751-BuildCleanPCHCheck\AfterClean_Summary.csv`

## Interpretation

`Build.bat UnrealEditor Win64 Development -Clean -WaitMutex` removed the local `.pch` files used by the `UnrealEditor Win64 Development` build, including `PCH.Core`, `PCH.CoreUObject`, `PCH.Engine`, `PCH.UnrealEd`, and `PCH.ResonanceAudio`.

It did not remove the historical `SharedPCH*` non-`.pch` files outside the UnrealEditor target scope. Since UnrealEditor-scope `SharedPCH*` count was already `0` before clean, this run cannot show a decrease there.

逻辑分析推理(无事实依据)：the target clean path appears to remove PCH binary outputs relevant to the target, but it does not perform a broad workspace purge of every historical file whose name starts with `SharedPCH`.

## Occam Check

- Did target-level clean remove local `.pch` files? Yes, from `5` to `0`.
- Did target-level clean remove UnrealEditor-scope `.pch` files? Yes, from `4` to `0`.
- Did target-level clean change UnrealEditor-scope `SharedPCH*` files? No, it was already `0`.
- Did target-level clean remove all historical `SharedPCH*` files under `D:\UE\5.7.4r`? No.
