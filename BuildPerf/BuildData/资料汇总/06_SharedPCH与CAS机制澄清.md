# SharedPCH 与 CAS 机制澄清

文件路径：`D:\UE\Docs\Build\资料汇总\06_SharedPCH与CAS机制澄清.md`

## Session 范围

Session ID：`019f34d3-e918-7f23-b84e-72e6524140e6`

标题：`澄清 SharedPCH 与 CAS 机制`

这个 session 的主语是 `D:\UE\5.8.0r` 的 UBT、XGE/IncrediBuild 与 UBA；宾语是 SharedPCH 是否每次重建、CAS 是否能减少 PCH 传输、IB 与 UBA 在相同或相近 action 下的 cold/warm 行为差异。

## 机制结论

```text
UBT 启动
  |
  +--> 判断 SharedPCH action 是否过期
       |
       +--> 未过期：不重新生成 .pch
       |
       +--> 已过期：cl.exe /Yc 重新生成 .pch
            |
            +--> XGE / IncrediBuild：按各自远端执行和缓存机制分发
            |
            +--> UBA：对文件内容生成 CAS key，按 key 存储和复用
```

关键边界：

- UBT 不会因为“被拉起”就必然重建 SharedPCH。
- UBT 会因为 SharedPCH action 输入或属性失效而重建 SharedPCH。
- XGE/IncrediBuild 与 UBA 是不同 executor 路径；选择 XGE/IncrediBuild 时，UE 的 UBA CAS 不参与。
- UBA 普通 CAS 是内容寻址复用，不是“让不同内容的 PCH 变成同一个对象”的差量算法。

## 源码证据摘要

| 源码路径 | 证据 |
|---|---|
| `D:\UE\5.8.0r\Engine\Source\Programs\UnrealBuildTool\Actions\Action.cs` | PCH 文件本身不作为普通 cached output，但 compile action 可纳入 PCH creator inputs 做有效性判断 |
| `D:\UE\5.8.0r\Engine\Source\Programs\UnrealBuildTool\System\Utils.cs` | `WriteFileIfChanged` 表明 UBT 辅助文件通常是内容变化才写 |
| `D:\UE\5.8.0r\Engine\Source\Programs\UnrealBuildTool\Executors\ExecutorFactory.cs` | XGE 与 UBA 是不同 executor 路径 |
| `D:\UE\5.8.0r\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaStorageUtils.cpp` | `CasKeyHasher` 按文件内容生成 CAS key |
| `D:\UE\5.8.0r\Engine\Source\Programs\UnrealBuildAccelerator\Common\Public\UbaSessionServer.h` | UBA 核心存在 custom CAS key 设计，可面向 PCH 这类非确定输出 |
| `D:\UE\5.8.0r\Engine\Source\Programs\UnrealBuildTool` | 未检索到 `SetCustomCasKeyFromTrackedInputs` 在 UBT 默认路径中的调用证据 |

逻辑分析推理(无事实依据)：当前 `D:\UE\5.8.0r` 的 UBA 核心具备 custom CAS key 能力，但默认 UBT 路径是否把该能力用于 PCH，未看到已接线证据。因此“不同字节 PCH 也可复用”不能作为 UE 5.8 默认构建事实。

## UBA cold / warm 实测

远端机：`10.226.143.38`

落盘报告：

- `D:\UEProject\Docs\XGE_PCH_Probe\uba_sharedpch_coldwarm_20260706_151339\ColdWarmReport.md`
- `D:\UE\Docs\Build\UBA_SharedPCH_ColdWarm.md`

目标 PCH：`D:\UE\5.8.0r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development\UnrealEd\SharedPCH.UnrealEd.Cpp20.h.pch`

| 指标 | cold | warm |
|---|---:|---:|
| Remote run | `14.5s` | `9.5s` |
| Host SendCas | `241 files, 2.6s` | 无大型 PCH 发送记录 |
| Host SendCas Raw/Comp | `2.6GB / 349.8MB` | 无大型 PCH 发送记录 |
| Large PCH CAS | raw `2,574,778,360 bytes` / comp `337,029,231 bytes` / recv `3.6s` | 无 `RecvCasLarge` |
| Agent RecvTotal | `350.6MB` | `832.1KB` |
| PCH memory map | `783ms` | `563ms` |

结论：

- cold 网络传输接近压缩 CAS 大小，约 `350MB`，不是原始 `2.398 GiB` 全量字节。
- warm 远端已有 CAS 后，不再观察到大型 PCH CAS 重传。
- warm 仍需要把压缩 CAS 映射/填充为 full PCH memory view，所以不是“零成本”。

## IncrediBuild / XGE 对照

### 同 XML 多轮

数据：`D:\UE\5.8.0r\Engine\Programs\UnrealBuildTool\XGE.SharedPCHEdOnly.MultiRun.Corrected.20260706-134937.csv`

同一 `D:\UE\5.8.0r\Engine\Intermediate\Build\UBTExport.005.EdOnly.xge.xml` 跑 5 次：

| Run | BuildId | Remote Agent task | RunTotal |
|---:|---:|---:|---:|
| 1 | `56` | `0` | `2.889 MiB` |
| 2 | `57` | `0` | `4.131 MiB` |
| 3 | `58` | `0` | `1.229 MiB` |
| 4 | `59` | `1` | `756.504 MiB` |
| 5 | `60` | `0` | `1.208 MiB` |

逻辑分析推理(无事实依据)：同 XML 不是每次都大流量；只有 Ed compile task 落到远端 helper 时，才出现约 `755 MiB` 级别的 SharedPCH 分发流量。

### 强制远端 5 个 PCH action

数据：`D:\UE\5.8.0r\Engine\Programs\UnrealBuildTool\XGE.CodexRemoteForcedPCH5.20260706-143154.summary.json`

| 指标 | 值 |
|---|---:|
| BuildId | `63` |
| Remote Agent | `GIH-D-X4947` / `10.226.171.109` |
| 编译任务 | 5 个全部远端 |
| IB BuildTime wall | `45.907s` |
| xgConsole wall | `48.218s` |
| 高发送窗口 | `11.291s` |
| 高发送窗口流量 | `744.430 MiB sent` |
| 全 run 总流量 | `788.211 MiB total` |

判断边界：IncrediBuild DB 没有 per-file transfer 明细；`11.291s` 是网卡高发送窗口推断，不是 IB 明确标注的 PCH 文件传输时长。

## N=99 粗模型

从 `C:\ProgramData\IncrediBuild\BuildData\BuildDB_42.db` 重构得到 remote helper 峰值约 `99`，`ActiveMachines` 峰值 `100`。

以单个 `SharedPCH.UnrealEd.Cpp20.h.pch` 为例：

| 模型 | 单 helper 数据 | N=99 fanout | 1Gbps 理论时间 |
|---|---:|---:|---:|
| UBA CAS 压缩对象 | `0.314 GiB` | `31.074 GiB` | `267s` |
| IB 原始全量假设 | `2.398 GiB` | `237.397 GiB` | `2039s` |
| IB gzip-1 近似 | `0.518 GiB` | `51.282 GiB` | `441s` |
| IB xz-1-T0 近似 | `0.295 GiB` | `29.205 GiB` | `251s` |

逻辑分析推理(无事实依据)：该模型用于解释数量级闭合，不证明 IB 实际使用哪一种压缩算法。

## 阿卡姆剃刀结论

最短治理路线不是先改 UBA 或 UBT，而是：

1. 固定保存每次构建的 `.obj.rsp` 与 action graph。
2. 区分 SharedPCH 是否真的重建、是否被远端 task 消费、是否因为 helper cold 而重传。
3. 针对 `SharedPCH.UnrealEd` 的消费者做模块级筛选，不全局 NoSharedPCH。
4. 对 UBA 使用 cold/warm CAS 数据验证缓存收益，对 IB 使用网卡流量和 BuildDB 验证实际远端调度。

局限性与潜在风险提示：本文件把源码证据、日志证据和逻辑推理合并摘要；精确审计应回到 `D:\UE\5.8.0r\Engine\Source\Programs\UnrealBuildAccelerator`、`D:\UEProject\Docs\XGE_PCH_Probe\uba_sharedpch_coldwarm_20260706_151339\ColdWarmReport.md`、`D:\UE\5.8.0r\Engine\Programs\UnrealBuildTool\XGE.CodexRemoteForcedPCH5.20260706-143154.summary.json` 和对应 `C:\ProgramData\IncrediBuild\BuildData` 数据库。
