# 定性分析 SharedPCH 对 UE 5.7.4r 编译速度的影响

> 文档定位：对 `D:\UE\5.7.4r` 的 Incredibuild(IB) UnrealEditor 构建做定性分析，解释 BuildId 32 与 BuildId 34 为何明显慢于 BuildId 33，并判定 SharedPCH/PCH 在其中扮演的角色。

---

## 一、阅读约定：四类内容分级

本文强制区分四类内容，避免把推断当事实。每个章节或条目均以对应标签标注可信级别。

| 标签 | 含义 | 可信级别 |
| --- | --- | --- |
| 【事实证据】 | 来自 IB BuildDB 与磁盘 PCH 文件的可复现测量值 | 高，可回溯到具体文件 |
| 【数学模型】 | 基于事实数据构建的可代入、可验算的量级公式 | 中高，公式自洽但依赖假设 |
| 【逻辑分析推理(无事实依据)】 | 用模型把事实连成因果的推断，无直接测量佐证 | 中，合理但未证 |
| 【不可证明边界】 | 现有数据结构上无法证明的命题 | 明确标记为不可证明 |

---

## 二、背景与分析对象 【事实证据】

- 分析对象：`D:\UE\5.7.4r` 的 Incredibuild UnrealEditor 构建。
- 对比数据来自三个构建的 IB BuildDB：
  - `C:\ProgramData\Incredibuild\BuildData\BuildDB_32.db`
  - `C:\ProgramData\Incredibuild\BuildData\BuildDB_33.db`
  - `C:\ProgramData\Incredibuild\BuildData\BuildDB_34.db`
- 当前 PCH 文件统计来自目录：`D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development`

对比意图：BuildId 32 与 BuildId 34 是慢构建，BuildId 33 是快构建，三者形成"慢/快/慢"对照组。

---

## 三、关键事实数据 【事实证据】

### 3.1 全程聚合

| 指标 | BuildId 32 | BuildId 33 | BuildId 34 |
| --- | --- | --- | --- |
| 总发送 | 1065.56 GiB | 36.61 GiB | 972.75 GiB |
| 总接收 | 55.03 GiB | 142.34 GiB | 56.72 GiB |
| 平均 SendRate | 823.8 MiB/s | 94.7 MiB/s | 790.3 MiB/s |
| 平均 RecvRate | 40.6 MiB/s | 341.8 MiB/s | 43.3 MiB/s |
| 平均 ActiveMachines | 64.7 | 45.5 | 64.3 |
| 平均 ActiveSlots | 312.0 | 247.6 | 327.3 |
| 平均 WaitingTasks | 2398.4 | 1874.5 | 2432.4 |
| 平均 HelperCores | 357.4 | 319.1 | 371.1 |

要点：32/34 发送量比接收量高约一个量级，且 SendRate 长期接近顶满；33 相反，接收远大于发送。

### 3.2 前 30 采样窗口 early30

| 指标 | BuildId 32 | BuildId 33 | BuildId 34 |
| --- | --- | --- | --- |
| Duration | 406.0s | 129.1s | 384.0s |
| Send | 365.08 GiB | 20.61 GiB | 345.76 GiB |
| Recv | 4.07 GiB | 41.27 GiB | 4.36 GiB |
| AvgSend | 935.0 MiB/s | 173.5 MiB/s | 931.6 MiB/s |
| AvgRecv | 10.4 MiB/s | 338.0 MiB/s | 11.8 MiB/s |
| AvgActiveMachines | 71.4 | 55.1 | 71.7 |
| AvgWaitingTasks | 2902.4 | 2936.7 | 2856.5 |

要点：32/34 在早期窗口 AvgSend 已顶到约 935 MiB/s，AvgRecv 极低（约 10~12 MiB/s），呈典型"上行分发主导"形态；33 早期即以接收为主。

### 3.3 远端 cl.exe early / late 时长

| 构建 | early avg | early max | late avg |
| --- | --- | --- | --- |
| BuildId 32 | 374.0s | 1010.9s | 44.3s |
| BuildId 33 | 42.0s | 287.7s | 24.3s |
| BuildId 34 | 383.3s | 988.3s | 33.8s |

要点：32/34 早期远端 cl.exe wall-clock 是晚期的 8~10 倍；33 早晚差距小得多。

### 3.4 严格公共 remote cl.exe（distinct BarCaption = 1691，status = 1）

| 构建 | mean | > 200s 数量 |
| --- | --- | --- |
| BuildId 32 | 94.87s | 260 |
| BuildId 33 | 25.01s | 0 |
| BuildId 34 | 96.98s | 240 |

要点：在同一批 1691 个可比对的成功远端编译任务上，32/34 的均值约为 33 的近 4 倍，且各有 240~260 个任务超过 200s；33 无一超过 200s。

### 3.5 磁盘 PCH 文件统计

来源目录：`D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development`

- SharedPCH：7 个，合计 8.73 GiB，最大三个为 2.25 / 2.25 / 2.11 GiB。
- ModulePCH：4 个，合计 4.76 GiB，最大两个为 1.96 / 1.89 GiB。
- 合计：SharedPCH + ModulePCH = 8.73 + 4.76 = 13.49 GiB。

### 3.6 DB 中 SharedPCH action 的本地 / 远端归属

- 在 BuildId 32 / 34，DB 中名为 SharedPCH 的 named action 是**本地 action**，不是远端 action。
- BuildId 32 / 34 的**远端 SharedPCH action 数为 0**。

含义：SharedPCH 本身在发起机本地生成，不是被派发到 Helper 上远端编译。这一条直接排除"远端在编 SharedPCH 导致慢"的假设。

---

## 四、数学模型：冷态预热分发税 【数学模型】

### 4.1 公式与变量

```text
T_warmup ~= H * S / B
```

- `S`：每台冷 Helper 首次可用前需要收到的有效 PCH/依赖载荷（GiB）。
- `H`：参与冷启动的 Helper 数量。
- `B`：发起机有效上行带宽（GiB/s）。

### 4.2 代入值（来自事实数据）

- `B ~= 935 MiB/s ~= 0.913 GiB/s`（取自 32/34 early30 的 AvgSend）。
- `H ~= 64`（全程平均 ActiveMachines），或 early 活跃机器约 72（early30 AvgActiveMachines）。
- 下述三档估算统一代入较保守的 `H = 64`、`B = 0.913 GiB/s`，以便与实测量级对齐。

### 4.3 三档量级估算

| 载荷情景 | S | 估算 T_warmup | 验算 |
| --- | --- | --- | --- |
| 单个最大 SharedPCH | 2.25 GiB | 约 158s | 64 × 2.25 / 0.913 ≈ 157.7 |
| 全部 SharedPCH | 8.73 GiB | 约 612s | 64 × 8.73 / 0.913 ≈ 611.9 |
| SharedPCH + ModulePCH | 13.49 GiB | 约 946s | 64 × 13.49 / 0.913 ≈ 945.6 |

### 4.4 量级闭合判定

- 模型估算区间约 158 ~ 946s。
- 实测 early avg 为 374 / 383s，实测 max 为 988 / 1011s。
- 结论：模型估算与实测 early avg 及 max **在同一量级闭合**，模型不需要引入额外机制即可解释观测到的耗时量级。

---

## 五、ASCII 图

### 5.1 模型图：冷态分发税

```text
模型图：冷态分发税

        PCH有效载荷 S
      2.25 ~ 13.49 GiB
              |
              v
Helper数量 H ------> 总分发压力 = H * S
32/34 early约72台          |
                           v
发起机上行 B --------> 冷爬坡时间 T ~= H*S/B
约0.91 GiB/s               |
                           v
                 远端 cl.exe wall-clock 变长
                 32: early avg 374s, max 1011s
                 34: early avg 383s, max 988s
```

### 5.2 量级闭合图

```text
量级闭合图，单位：秒

0        200        400        600        800       1000
|---------|----------|----------|----------|----------|

S=2.25GiB 单个大SharedPCH估算
████████                                                  158s

BuildId 32 实测 early avg
███████████████████                                      374s

BuildId 34 实测 early avg
███████████████████                                      383s

S=8.73GiB 全部SharedPCH估算
███████████████████████████████                          612s

S=13.49GiB SharedPCH+ModulePCH估算
███████████████████████████████████████████████          946s

BuildId 34 实测 max
█████████████████████████████████████████████████        988s

BuildId 32 实测 max
██████████████████████████████████████████████████       1011s
```

### 5.3 因果链图

```text
BuildId 32/34:
本地 SharedPCH/PCH 生成
  -> 大依赖集需要被 IB 分发
  -> 发起机 SendRate 长时间顶满
  -> Helper 早期冷爬坡
  -> 远端 cl.exe wall-clock 耗时暴涨
  -> recovery / disabling helper / status=3 增多

BuildId 33:
无本地 SharedPCH 生成
  -> SendRate 不顶满
  -> RecvRate 更高
  -> 远端 cl.exe 早期也较快
```

---

## 六、逻辑分析推理(无事实依据) 【逻辑分析推理(无事实依据)】

以下为用模型把事实连成因果的推断，逻辑自洽但缺乏 per-file 级别直接测量佐证：

1. 32/34 的慢**大概率不是因为 Helper 少**。它们的 ActiveMachines（64.7 / 64.3）甚至高于快构建 33（45.5）。Helper 更多时，发起机反而需要向更多冷 Helper 分发同一批大依赖，冷态分发总量 `H * S` 随 H 上升，预热税更重。
2. 32/34 的形态是**上行分发瓶颈 + 冷爬坡**：SendRate 顶满、RecvRate 极低、early 远端 cl.exe 是 late 的 8~10 倍，三者共同指向"早期把大量依赖推给 Helper，Helper 就绪前任务 wall-clock 被拉长"。
3. SharedPCH/PCH 是这批冷态载荷**最强的量级来源候选**：磁盘上 SharedPCH 合计 8.73 GiB、加 ModulePCH 合计 13.49 GiB，代入模型后与实测量级闭合，无需第二种机制。
4. 33 快，是因为它没有本地 SharedPCH 生成这一冷态分发负担，早期即进入以接收为主的正常协作节奏。

上述第 3 点是"量级候选"，不是"字节级证实"，故归入无事实依据推理，详见第十节不可证明边界。

---

## 七、判断矩阵

| 命题 | 支持强度 | 依据 |
| --- | --- | --- |
| 32/34 更像上行分发瓶颈 | 强支持 | SendRate 顶满（823.8 / 790.3 MiB/s），发送量比接收量高约一个量级 |
| 32/34 存在冷爬坡 | 强支持 | early30 AvgSend 约 935 MiB/s 顶满；早期远端 cl.exe 374 / 383s 远高于 late |
| SharedPCH/PCH 是主要嫌疑对象 | 中强支持 | 磁盘 PCH 合计 13.49 GiB，代入模型与实测量级闭合，但缺 per-file 佐证 |
| 远端在编 SharedPCH 导致慢 | 不支持 | DB 中 SharedPCH 为本地 action，32/34 远端 SharedPCH action = 0 |
| 更多 Helper 稀释 cache 命中 | 不能证明 | 现有 DB 无 cache 命中率明细，无法证实或证伪 |
| 问题风暴（持续正反馈失控） | 部分支持 | 应降级为"冷分发税 + 恢复长尾"，不可过度断言持续正反馈失控 |

---

## 八、最终结论

1. BuildId 32 / 34 慢，**大概率不是因为 Helper 少**，而是因为 Helper 多时发起机需要向**更多冷 Helper 分发大依赖**，冷态分发税 `H * S / B` 随 H 增大而升高。
2. **SharedPCH/PCH 是最强嫌疑**，但现有 DB **不能直接证明**被传输的字节就是 SharedPCH 文件本身。
3. 更稳妥的分层模型是：
   - 主因 = **冷态 PCH/依赖分发税**；
   - 副因 = **Helper recovery / disabling / status=3 长尾**；
   - 弱证据 = **cache 命中稀释**（仅作为可能性保留，不作定论）。

---

## 九、阿卡姆剃刀检查

- 是否需要把所有慢都归因到 SharedPCH？**不需要**。SharedPCH/PCH 只是最强量级候选，恢复长尾等副因同样贡献耗时。
- 是否需要证明 cache 稀释？**当前不能证明**，故仅作为弱证据保留，不纳入主因。
- 是否需要用正反馈风暴解释全部现象？**不需要**。"冷分发税 + 恢复长尾"已能解释主要数据，引入持续正反馈失控属于过度假设。

---

## 十、不可证明边界 【不可证明边界】

以下命题受限于现有 IB DB 的数据结构，当前**无法被证明或证伪**：

- 无法证明发送字节流中"哪一部分、多少字节"精确对应 SharedPCH/PCH 文件；DB 只有聚合发送量，没有 per-file transfer 明细。
- 无法证明"更多 Helper 稀释 cache 命中率"，因为 DB 不含 cache 命中率明细。
- 因此第六节与第八节第 2 点的 SharedPCH 归因，其性质是**高可信推论**，尚未达到**直接事实**级别。

---

## 局限性与潜在风险提示

当前 IB DB 没有 per-file transfer 明细，所以不能把发送量逐项归因到 SharedPCH；需要 IB 详细文件传输日志，或同步记录发起机网卡上行与磁盘读取 `D:\UE\5.7.4r\Engine\Intermediate\Build\Win64\x64\UnrealEditor\Development` 的计数器，才能把该结论从高可信推论升级为直接事实。
