# UBA PCH Sampling Report

- RunId: `full_20260706_01`
- Summary CSV: `summary.csv`
- Manifest CSV: `manifest.csv`
- Scope: `Engine/Intermediate/Build/Win64/x64/UnrealEditor/Development`
- Unique agent: `10.226.143.38`

## SharedPCH baseline vs NoSharedPCH

| sample | tier | baseline cold | noshared cold | cold delta | baseline warm | noshared warm | warm delta | baseline SendCas raw/comp | noshared SendCas raw/comp | baseline SendTotal | noshared SendTotal |
| --- | --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| `SharedPCH.UnrealEd_CADKernel_1` | `SharedPCH.UnrealEd` | 11.9s | 7.1s | 4.8s | 6.9s | 5.5s | 1.4s | 2.6gb / 349.5mb | 41.5mb / 14.5mb | 350.3mb | 15.4mb |
| `SharedPCH.Engine_BlueprintGraph_1` | `SharedPCH.Engine` | 12.9s | 17.2s | -4.3s | 8.2s | 14.7s | -6.5s | 2.4gb / 316.0mb | 56.9mb / 18.8mb | 316.9mb | 20.0mb |
| `SharedPCH.Slate_Chaos_1` | `SharedPCH.Slate` | 16.2s | 17.2s | -1s | 13.2s | 15.3s | -2.1s | 954.7mb / 134.6mb | 51.3mb / 17.1mb | 135.6mb | 18.2mb |
| `SharedPCH.CoreUObject_AssetRegistry_1` | `SharedPCH.CoreUObject` | 7.3s | 9.8s | -2.5s | 5.3s | 8.2s | -2.9s | 567.1mb / 83.8mb | 45.3mb / 15.6mb | 84.1mb | 16.0mb |
| `SharedPCH.Core_Analytics_1` | `SharedPCH.Core` | 2.2s | 4.8s | -2.6s | 1.4s | 3.5s | -2.1s | 416.6mb / 62.9mb | 39.6mb / 14.2mb | 63.1mb | 14.5mb |

## ModulePCH baseline

| sample | tier | cold | warm | cold SendCas raw/comp | cold SendTotal | warm SendTotal |
| --- | --- | ---: | ---: | ---: | ---: | ---: |
| `ModulePCH.Engine_Engine_1` | `ModulePCH.Engine` | 23.1s | 9.5s | 2.1gb / 282.2mb | 283.3mb | 1.0mb |
| `ModulePCH.UnrealEd_UnrealEd_1` | `ModulePCH.UnrealEd` | 15.7s | 4.6s | 2.2gb / 296.0mb | 297.1mb | 1.0mb |
| `ModulePCH.Core_Core_1` | `ModulePCH.Core` | 5.1s | 3.2s | 418.9mb / 63.4mb | 63.8mb | 350.1kb |
| `ModulePCH.CoreUObject_CoreUObject_1` | `ModulePCH.CoreUObject` | 8.7s | 5.6s | 583.7mb / 86.0mb | 86.3mb | 321.8kb |

## Cleanup

- Remote agent: `NO_REMOTE_UBA_AGENT`
- Local UBA processes: none
- Firewall rule: not found after cleanup
