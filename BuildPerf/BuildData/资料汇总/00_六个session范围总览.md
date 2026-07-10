# 六个 session 范围总览

文件路径：`D:\UE\Docs\Build\资料汇总\00_六个session范围总览.md`

## 范围总表

| Session ID | 范围 | 已沉淀材料 |
|---|---|---|
| `019f2b0f-faab-7b81-b4da-c15b18739168` | 创建并修复 `D:\UE\5.7.4r\BuildEditor.bat`；分析初次失败、增量成功、clean、PCH/SharedPCH、Build32/33/34 和 helper 长尾 | `D:\UE\Docs\Build\5.7.4r`、`D:\UE\Docs\Build\5.7.4r\定性分析SharedPCH对UE编译速度的影响.md` |
| `019f2c30-202a-7582-82f1-18a286c013e8` | 检查 `D:\UE\5.4.4r` BuildConfiguration；修复 MSVC 14.44 暴露的 5.4 编译问题；安装 MSVC 14.38；形成 Build39/40/41 对照 | `D:\UE\5.4.4r\Engine\Saved\Logs`、`D:\UE\Docs\Build\SharedPCH数学证明\SharedPCH冷态分发税数学证明.md` |
| `019f2d8f-5402-78c1-98f0-dbc8c4929ac3` | 在 `D:\UE\5.8.0r` 对比 NoSharedPCH 与 SharedPCH enabled；分析 Build42/43、remote task、`.rsp`、SharedPCH 冷态分发税 | `D:\UE\5.8.0r\Engine\Saved\Logs`、`D:\UE\Docs\Build\SharedPCH数学证明\SharedPCH冷态分发税数学证明.md` |
| `019f2fbc-5fa5-7e50-9c32-c9e50ef63a5e` | 评估局域网 1Gbps/10Gbps、多机汇聚、交换机背板、目标端存储/CPU/协议栈对 IncrediBuild 的影响 | `D:\UE\Docs\Build\资料汇总\04_局域网带宽瓶颈评估.md` |
| `019f2fac-c36a-7fa1-b40e-03cd789577b8` | 分析 Build42/43 首轮 cold remote task；把 task 追到 `.obj.rsp`、`/Yu`、`/Fp`、SharedPCH/ModulePCH；补充 PCH 3/4/5 体积模型和 UBA 抽样验证 | `D:\UEProject\Docs\UE5.8_SharedPCH_ModulePCH_3_4_5_Analysis.md`、`D:\UEProject\Docs\XGE_PCH_Probe\uba_pch_sampling_full_20260706_01`、`D:\UE\Docs\Build\UBA_SharedPCH_ColdWarm.md` |
| `019f34d3-e918-7f23-b84e-72e6524140e6` | 澄清 UBT 是否每次重建 SharedPCH、XGE 与 UBA CAS 是否互通、UBA 普通 CAS/custom CAS key 能力，以及 IB/UBA 同 XML 的实际传输差异 | `D:\UEProject\Docs\XGE_PCH_Probe\uba_sharedpch_coldwarm_20260706_151339\ColdWarmReport.md`、`D:\UE\5.8.0r\Engine\Programs\UnrealBuildTool\XGE.CodexRemoteForcedPCH5.20260706-143154.summary.json` |

## 统一主线

```text
用户问题
  |
  +--> 为什么 SharedPCH 让 UE 构建变慢？
  |       |
  |       +--> D:\UE\5.7.4r 的 Build32/34 冷态 SharedPCH 样本
  |       +--> D:\UE\5.4.4r 的 Build39/40 对照样本
  |       +--> D:\UE\5.8.0r 的 Build42/43 对照样本
  |       +--> D:\UE\5.8.0r 的 Build42/43 top remote task PCH 映射
  |
  +--> SharedPCH 为什么巨大？
  |       |
  |       +--> D:\UE\PCHAnalysis\UnrealEdCpp20 的 /Z7、/Zi、no-debug 对照
  |       +--> 第 3 部分 AST/语义结构约 1444.23 MiB
  |       +--> 第 4 部分模板状态约 1011.33 MiB
  |       +--> 第 5 部分对 .pch 本体约 0 MiB，主要进 .obj/.pdb
  |
  +--> 为什么 helper 多了反而更慢？
  |       |
  |       +--> SharedPCH 大文件冷态上传
  |       +--> 多 helper 首任务拉取同类大 PCH
  |       +--> 网络/缓存/磁盘/调度共同放大早段耗时
  |
  +--> UBA CAS 能否缓解？
  |       |
  |       +--> 同字节文件内容 CAS key 相同，可以减少 warm 传输
  |       +--> PCH 被重新生成且字节不同，普通 CAS key 不同
  |       +--> custom CAS key 是 UBA 核心能力，但 D:\UE\5.8.0r 的 UBT 默认路径未看到已接线证据
  |
  +--> 升级 10Gbps 是否能解决？
          |
          +--> 如果瓶颈是单机链路，10Gbps 可能有效
          +--> 如果瓶颈是汇聚、服务器网卡、存储、杀毒或 helper 状态，10Gbps 不能单独闭环
```

## 总结结论

1. `D:\UE\5.7.4r` 的 session 首先解决了脚本可用性问题，随后通过 clean、SharedPCH 删除/再生成和 Build32/33/34 证明 SharedPCH 冷态成本很高。
2. `D:\UE\5.4.4r` 的 session 首先解决了工具链和 UE 5.4 源码兼容问题，随后用 Build39/40/41 证明 SharedPCH enabled 的冷态分发会显著拉长 remote `cl.exe` 耗时。
3. `D:\UE\5.8.0r` 的 NoSharedPCH session 将对比扩展到 Build42/43，并把 `.rsp`、`/Yu`、helper 首任务和同名 remote task 证据纳入数学证明。
4. `019f2fbc-5fa5-7e50-9c32-c9e50ef63a5e` 的网络 session 给这些构建数据提供解释框架：局域网不是老式共享总线，但多机同时拉大文件仍会在端到端最窄环节形成瓶颈。
5. `019f2fac-c36a-7fa1-b40e-03cd789577b8` 把 Build42/43 的慢 remote task 归因到 SharedPCH/ModulePCH，并通过 `D:\UEProject\Docs\XGE_PCH_Probe\uba_pch_sampling_full_20260706_01\summary.csv` 证明 NoSharedPCH 不是全局更快，只对 `SharedPCH.UnrealEd` 的部分消费者明显有效。
6. `019f34d3-e918-7f23-b84e-72e6524140e6` 把机制边界讲清楚：UBT 不会因为被拉起就必然重建 SharedPCH；UBA 普通 CAS 能减少同 key 文件的 warm 传输，但不能替代 SharedPCH 失效治理。

逻辑分析推理(无事实依据)：6 个 session 的共同目标不是单纯追求“不生成 SharedPCH”，而是找到 UE 大型源码在 IncrediBuild/XGE/UBA 下冷态编译变慢的主因、可复现证据和最小治理边界。
