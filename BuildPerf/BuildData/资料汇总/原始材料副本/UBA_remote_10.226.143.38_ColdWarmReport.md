# UBA SharedPCH Cold/Warm Single-Agent Report

## Inputs

- EngineRoot: D:\UE\5.8.0r
- RspPath: D:\UEProject\Docs\XGE_PCH_Probe\uba_pch_sampling_full_20260706_01\samples\SharedPCH.UnrealEd_CADKernel_1\baseline\SharedPCH.UnrealEd_CADKernel_1.baseline.rsp
- Remote: phoen@10.226.143.38
- RemoteWorkRoot: C:\ProgramData\Epic\UbaAgent\Codex\ColdWarm
- LogCasBytes: 104857600

## Cold

| Step | Value |
| --- | --- |
| Remote run | 14.5s |
| Host SendCas | 241 files, 2.6s |
| Host SendCas Raw/Comp | 2.6gb   349.8mb |
| Agent ReceiveCas | 241 files, 15.2s |
| Agent ReceiveCas Raw/Comp | 2.6gb   349.8mb |
| Agent ReceiveCas Decompress | 111 calls, 68ms |
| Large PCH RecvCas | hint=D:\UE\5.8.0r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.Cpp20.h.pch cas=60ecfcbdaccd3b5e6d30d8b4a3407f645af97901 raw=2.6gb raw_bytes=2574778360 comp=337.0mb comp_bytes=337029231 recv=3.6s recv_ms=3562 decompressRecv=0ms decompress_recv_ms=0 |
| PCH memory map decision | file=D:\UE\5.8.0r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.Cpp20.h.pch alignment=65536 useStorage=1 allowMemoryMaps=1 decision=<1ms decision_ms=0 |
| PCH memory map large | file=C:\ProgramData\Epic\UbaAgent\Codex\ColdWarm\uba_sharedpch_coldwarm_20260706_151339\store\cas\60\60ecfcbdaccd3b5e6d30d8b4a3407f645af97901 hint=D:\UE\5.8.0r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.Cpp20.h.pch compressed=1 content=2.6gb content_bytes=2574778360 alignment=65536 allocMap=18ms alloc_map_ms=18 fillMemory=757ms fill_memory_ms=757 total=783ms total_ms=783 can_be_freed=0 mapping=^2-5m1g |
| CreateMmapFromFile summary | 132 calls, 1.8s |
| MappingBuffer summary | 1, 2.6gb |
| DecompressToMem summary | 132 calls, 722ms |
| MapViewOfFile summary | 94 calls, 24ms |
| MapViewOfFile3 summary | 30 calls, <1ms |

## Warm

| Step | Value |
| --- | --- |
| Remote run | 9.5s |
| Host SendCas |  |
| Host SendCas Raw/Comp |  |
| Agent ReceiveCas |  |
| Agent ReceiveCas Raw/Comp |  |
| Agent ReceiveCas Decompress |  |
| Large PCH RecvCas | CAS hit or no large PCH CAS transfer logged |
| PCH memory map decision | file=D:\UE\5.8.0r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.Cpp20.h.pch alignment=65536 useStorage=1 allowMemoryMaps=1 decision=<1ms decision_ms=0 |
| PCH memory map large | file=C:\ProgramData\Epic\UbaAgent\Codex\ColdWarm\uba_sharedpch_coldwarm_20260706_151339\store\cas\60\60ecfcbdaccd3b5e6d30d8b4a3407f645af97901 hint=D:\UE\5.8.0r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.Cpp20.h.pch compressed=1 content=2.6gb content_bytes=2574778360 alignment=65536 allocMap=11ms alloc_map_ms=11 fillMemory=551ms fill_memory_ms=551 total=563ms total_ms=563 can_be_freed=0 mapping=^2 |
| CreateMmapFromFile summary | 132 calls, 577ms |
| MappingBuffer summary | 1, 2.6gb |
| DecompressToMem summary | 132 calls, 513ms |
| MapViewOfFile summary | 94 calls, 18ms |
| MapViewOfFile3 summary | 30 calls, <1ms |

## Mermaid

~~~mermaid
sequenceDiagram
    participant Host as Host UbaCli
    participant Agent as Agent UbaAgent
    participant CAS as Agent CAS Store
    participant Map as Agent PCH MemoryMap
    participant CL as Remote cl.exe

    rect rgb(35, 50, 70)
    Note over Host,CL: COLD
    Host->>Agent: SendCas 241 files, 2.6s, 2.6gb   349.8mb
    Agent->>CAS: ReceiveCas 241 files, 15.2s, 2.6gb   349.8mb
    Note over Agent: Large PCH CAS: hint=D:\UE\5.8.0r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.Cpp20.h.pch cas=60ecfcbdaccd3b5e6d30d8b4a3407f645af97901 raw=2.6gb raw_bytes=2574778360 comp=337.0mb comp_bytes=337029231 recv=3.6s recv_ms=3562 decompressRecv=0ms decompress_recv_ms=0
    CL->>Agent: request .pch
    Agent->>Agent: PCH memory map decision: file=D:\UE\5.8.0r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.Cpp20.h.pch alignment=65536 useStorage=1 allowMemoryMaps=1 decision=<1ms decision_ms=0
    Agent->>Map: AllocAndMapView + fill: file=C:\ProgramData\Epic\UbaAgent\Codex\ColdWarm\uba_sharedpch_coldwarm_20260706_151339\store\cas\60\60ecfcbdaccd3b5e6d30d8b4a3407f645af97901 hint=D:\UE\5.8.0r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.Cpp20.h.pch compressed=1 content=2.6gb content_bytes=2574778360 alignment=65536 allocMap=18ms alloc_map_ms=18 fillMemory=757ms fill_memory_ms=757 total=783ms total_ms=783 can_be_freed=0 mapping=^2-5m1g
    Map-->>CL: MapViewOfFile summary 94 calls, 24ms, MapViewOfFile3 30 calls, <1ms
    CL-->>Host: Remote run 14.5s
    end

    rect rgb(35, 70, 45)
    Note over Host,CL: WARM
    Host->>Agent: SendCas , 
    Agent->>CAS: ReceiveCas , 
    Note over Agent: Large PCH CAS: CAS hit or no large PCH CAS transfer logged
    CL->>Agent: request .pch
    Agent->>Agent: PCH memory map decision: file=D:\UE\5.8.0r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.Cpp20.h.pch alignment=65536 useStorage=1 allowMemoryMaps=1 decision=<1ms decision_ms=0
    Agent->>Map: AllocAndMapView + fill: file=C:\ProgramData\Epic\UbaAgent\Codex\ColdWarm\uba_sharedpch_coldwarm_20260706_151339\store\cas\60\60ecfcbdaccd3b5e6d30d8b4a3407f645af97901 hint=D:\UE\5.8.0r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.Cpp20.h.pch compressed=1 content=2.6gb content_bytes=2574778360 alignment=65536 allocMap=11ms alloc_map_ms=11 fillMemory=551ms fill_memory_ms=551 total=563ms total_ms=563 can_be_freed=0 mapping=^2
    Map-->>CL: MapViewOfFile summary 94 calls, 18ms, MapViewOfFile3 30 calls, <1ms
    CL-->>Host: Remote run 9.5s
    end
~~~
