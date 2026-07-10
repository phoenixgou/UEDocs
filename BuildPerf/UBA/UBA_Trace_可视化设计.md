# UBA Trace 可视化设计

## 1. 文档定位

本文档定义 Unreal Build Accelerator（UBA）Trace 从运行时采集、二进制记录、离线解析到 Perfetto 展示的稳定设计契约。

- 当前实现基线：`E:\UEWS\5.8.0`。
- 当前 Trace 二进制版本：`TraceVersion = 52`。
- 当前 Trace 最低兼容版本：`TraceReadCompatibilityVersion = 6`。
- 当前存储网络协议版本：`StorageNetworkVersion = 4`。
- 路线 B 的目标：不改变 UBA Trace 生产协议，将已有 Trace 离线转换为 Perfetto 可导入的 Chrome Trace JSON。
- 路线 C-Lite 的目标：在路线 B 稳定后补充文件传输的端到端语义和 Perfetto flow，不替代路线 B。

路线 B 与路线 C-Lite 不存在质量优劣，只有交付优先级：先完成路线 B，再实施路线 C-Lite。

实施任务、协议变更步骤与验收命令由 `E:\UEWS\OpenSpec\changes\add-uba-trace-perfetto-export` 管理，本文档不承担施工清单职责。

规范性关系：本文档是架构、采集点与可视化语义的权威来源；`E:\UEWS\OpenSpec\changes\add-uba-trace-perfetto-export\specs\uba-trace-perfetto-export\spec.md` 与 `E:\UEWS\OpenSpec\changes\add-uba-trace-perfetto-export\specs\uba-trace-transfer-semantics\spec.md` 是实施 SHALL、版本目标和验收行为的权威来源。本文镜像的目标版本与退出码必须在 OpenSpec 评审门禁中核对，不得独立演进。

## 2. 范围与非目标

### 2.1 范围

1. 说明 UBA 运行时、Trace 子系统、Visualizer 与离线转换器之间的边界。
2. 列出当前 Trace 已有采集点及路线 C-Lite 计划新增的传输语义采集点。
3. 定义 UBA 数据到 Perfetto Track、Slice、Counter、Flow 和 Metadata 的转换关系。
4. 定义缺失数据、未闭合事件、版本与时间单位的显示语义。

### 2.2 非目标

- 不引入 Perfetto SDK，也不直接生成 `.pftrace`。
- 不增加 ETW Provider。
- 不在 UBA Runtime 中引入通用 `ITraceSink` 抽象。
- 不把所有 Trace 读取改为新的内存映射方案。
- 不记录逐数据块事件。
- 不在首期合并多台机器的独立 Trace 时间线。
- 不改变 UbaVisualizer 的 UI 与绘制逻辑。

## 3. UBA 总体架构与 Trace 所在位置

图例：`[现有]` 表示 `E:\UEWS\5.8.0` 已存在；`[B]` 表示路线 B 新增；`[C]` 表示路线 C-Lite 新增。

```text
构建发起端 / UbaHost
|
+-- [现有] SessionServer --------------------------+
|   |                                               |
|   +-- 进程、任务、调度、会话统计                  |
|   +-- [现有] StorageServer -- CAS Fetch / Store --+----+
|   +-- [现有] Trace(Host) <------------------------+    |
|                                                        |
|                 UBA 网络协议                           |
|                                                        |
+--------------------------+-----------------------------+
                           |
                 +---------+----------+
                 |                    |
          [现有] StorageProxy   [现有] UbaAgent
          可选转发与缓存         |
                                +-- SessionClient
                                +-- StorageClient
                                +-- Trace(Agent)

[现有] Trace / TraceChannel
|
+-- 版本化 UBA 二进制 Trace 文件、命名共享内存或网络流
|
+-- [现有] TraceReader -> TraceView -> UbaVisualizer
|
+-- [B] TraceReader -> TraceView -> TrackRegistry
                                   -> UbaJsonStreamWriter
                                   -> Chrome Trace JSON
                                   -> Perfetto UI

[C] StorageServer / StorageProxy / StorageClient
|
+-- operationId + 请求上下文 + 客户端完成事实
+-- TraceVersion 53 传输事件
+-- [B] 相同离线管线输出 Perfetto Slice + Flow
```

### 3.1 组件边界

| 组件 | 当前职责 | 可视化关系 |
|---|---|---|
| `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Public\UbaSessionServer.h` | Host 侧会话、进程与调度编排 | 生产 Session、Process、Task、Work 和周期统计事件 |
| `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Public\UbaSessionClient.h` | Agent 侧会话和远程进程执行 | 生产 Agent 会话更新、进程统计及执行结果 |
| `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Public\UbaStorageServer.h` | CAS 存储、Fetch 与 Store 服务 | 当前生产文件传输事件；路线 C-Lite 管理传输 operation 生命周期 |
| `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Public\UbaStorageProxy.h` | 存储协议代理和数据转发 | 路线 C-Lite 透传 operation 与请求上下文，不成为新的 Trace 所有者 |
| `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Public\UbaStorageClient.h` | 请求 CAS、接收和物化文件 | 路线 C-Lite 回传接收、解压、物化与结果事实 |
| `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Public\UbaTrace.h` | 定义 Trace 版本、事件及写入 API | Trace 生产端的稳定边界 |
| `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Public\UbaTraceChannel.h` | Trace 数据通道 | 承载文件、共享内存或网络读取 |
| `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Visualizer\Private\UbaTraceReader.h` | 当前解析二进制 Trace 并构造 `TraceView` | 路线 B 将 Reader 与纯数据模型机械迁移到共享分析模块 |
| `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Visualizer\Private\UbaVisualizer.cpp` | 当前 Windows 可视化 UI | 继续消费共享 `TraceView`；离线转换器不依赖该 UI |

## 4. Trace 数据流与所有权

```text
运行时事实
  |
  | SessionServer / SessionClient / StorageServer / Scheduler
  v
Trace API
  |
  | 带 TraceVersion 的事件序列
  v
Trace / TraceChannel
  |
  +--> 二进制 Trace 文件
  +--> 命名共享内存
  +--> 网络读取
          |
          v
      TraceReader
          |
          | 负责版本检查、时间换算、配对 Begin/End、构建聚合数据
          v
      TraceView
          |
          +--> UbaVisualizer 绘制
          |
          +--> TrackRegistry 确定稳定 pid/tid
                    |
                    v
              JSON 流式写入
                    |
                    v
                Perfetto UI
```

边界约束：

1. Trace 生产端只表达 UBA 事实，不感知 Perfetto。
2. `TraceReader` 只负责 UBA 二进制语义和 `TraceView` 构建，不写 JSON。
3. `TrackRegistry` 只负责稳定 Track 编号，不读取系统进程 ID、地址或哈希迭代顺序。
4. Exporter 只消费完整 `TraceView`，不修改 Visualizer 状态。
5. Perfetto 输出错误不得反向改变 UBA Trace 输入文件。

## 5. 当前采集点与目标呈现

| 采集域 | 当前生产位置 | 当前事实或粒度 | Perfetto 目标呈现 | C-Lite 增量 |
|---|---|---|---|---|
| Trace 文件头与 Summary | `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaTrace.cpp` | Trace 版本、频率、起始时间、Summary/EOF | Metadata；解析结果不作为时间片 | 无 |
| Session 生命周期 | `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaSessionServer.cpp` | Session 名称、机器、连接、断开与信息 | 每个 Session 一个 Perfetto process；断开信息作为参数或状态 | 无 |
| Session 周期统计 | `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaSessionServer.cpp`、`E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaSessionClient.cpp` | CPU、可用内存、网络收发、Ping、连接数 | Counter tracks | 无 |
| Process | `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaTrace.cpp` | 开始、结束、描述、远端/复用/竞速/空闲、退出与统计 | Processor track 上的 `X` Slice | 可附加 requester process 关联，但不改变 Process Slice |
| Task | `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaTrace.cpp` | `TaskBegin`/`TaskEnd`，在 `TraceView` 中类型为 `ProcessType_Task` | Processor track 上的 `X` Slice，分类为 Task | 无；不得重复输出为 Work |
| Work | `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaTrace.cpp` | `WorkBegin`/`WorkEnd`、描述、颜色和日志 | Work track 上的 `X` Slice | 无 |
| Fetch（详细模式） | `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaStorageServer.cpp` | key、size、hint、start、stop；`TraceReader` 在 `FileFetchEnd` 闭合时增加 count/bytes | Fetch track 上的 `X` Slice | 增加 operation、来源、目标、结果和客户端阶段事实 |
| Fetch（轻量模式） | `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaStorageServer.cpp` | `FileFetchLight` 在 FetchBegin 路径单次发出；`TraceReader` 收到后立即增加 count/bytes | Counter；不得虚构详细 Slice | 无详细证据时保持“无传输明细证据” |
| Store（详细模式） | `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaStorageServer.cpp` | key、size、hint、start、stop；`TraceReader` 在 `FileStoreBegin` 时增加 count/bytes | Store track 上的 `X` Slice | Fetch 稳定后单独设计 Store operation，不直接复制 Fetch 报文 |
| Store（轻量模式） | `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaStorageServer.cpp` | `FileStoreLight` 由 StoreBegin 路径单次发出；`TraceReader` 收到后立即增加 count/bytes | Counter；不得虚构详细 Slice | 无 |
| Cache Fetch/Write | `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaTrace.cpp` | Cache 开始、结束、结果与统计 | Cache track 上的 Slice，结果写入 args | 无 |
| Scheduler/Active Process | `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaScheduler.cpp` | 活跃进程计数和调度相关 Work | Counter 与 Work Slice | 无 |
| Drive | `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaTrace.cpp` | busy、读写次数、读写字节 | 每个盘符的 Counter tracks | 无 |
| Proxy | `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaStorageProxy.cpp` | 当前以代理连接、缓存和传输行为为主；一次上游 Fetch 可服务多个下游请求 | Metadata 或独立上下游传输 Slice 参数 | 上游 cache-fill 与每个下游 serve 使用各自 operationId；不得把一个上游 ID 透传给多个下游 |

## 6. TraceView 数据语义

路线 B 共享分析模块保留当前 `TraceView` 语义；第一步只做机械迁移，不顺带重构 `BitmapCache` 或 UI 数据。

`BitmapCache` 当前仅携带 `HBITMAP` 句柄、偏移与 dirty 状态；非 Windows 已将 `HBITMAP` 定义为 `void*`。B1 保留该句柄是机械迁移的显式权衡，不等于 `UbaTraceAnalysis` 或 `UbaTraceConverter` 依赖 Visualizer UI、GDI 绘制或窗口生命周期。

主要数据来源为 `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Visualizer\Private\UbaTraceReader.h`：

| `TraceView` 数据 | 语义 | 转换约束 |
|---|---|---|
| `sessions` | 主机及 Agent 会话 | `sessionIndex` 决定稳定 pid 顺序 |
| `Session.processors[].processes[]` | Process 与 Task | `ProcessType_Task` 只映射为 Task Slice |
| `workTracks[].records[]` | Work | 每个 work track 独立 tid |
| `Session.fetchedFiles`、`Session.storedFiles` | 仅详细模式下存在的传输记录 | 只对有效闭合记录输出 Slice |
| `Session.fetchedFilesCount/Bytes`、`Session.storedFilesCount/Bytes` | 聚合传输统计 | 输出 Counter；不得用来推断详细记录数量 |
| `networkSend`、`networkRecv`、`ping`、`memAvail`、`cpuLoad`、`connectionCount` | 周期采样 | 使用采样时间轴输出 Counter |
| `drives` | 盘符维度的忙碌度和 I/O | 每个盘符、每种指标使用独立 Counter |
| `cacheWrites`、`cacheSummaries` | Cache 写入和摘要 | 可验证区间输出 Slice；摘要作为 args 或 Metadata |
| `activeProcessCounts` | 全局活动进程数 | Counter |
| `realStartTime`、`traceSystemStartTimeUs`、`startTime`、`frequency`、`timeMultiplier` | 时间基准 | Exporter 不重新应用输入 Trace 的 frequency |
| `version`、`finished` | 输入版本与读状态的一部分 | `finished` 不能单独判断解析成功 |

### 6.1 三态传输证据

传输可视化必须区分：

1. **有详细证据**：存在有效 Begin/End，可输出 Slice。
2. **只有聚合证据**：只有 count/bytes，可输出 Counter，不能合成 Slice。
3. **无传输证据**：既无详细记录也无聚合增长，显示为空，不解释为“传输耗时为零”。

### 6.2 读取结果

离线读取使用显式结果，不使用单个 `bool` 或 `TraceView.finished` 混淆状态：

| 结果 | 语义 | CLI 分类 |
|---|---|---|
| `CompleteSummary` | 读取到正常 Summary 终止 | 成功 |
| `CompleteEof` | 读取到允许的完整 EOF 终止 | 成功 |
| `InputOpenFailed` | 输入无法打开 | 输入/解析失败 |
| `UnsupportedVersion` | Trace 版本不受支持 | 输入/解析失败 |
| `Truncated` | 事件或字段被截断 | 输入/解析失败 |
| `UnknownEvent` | 出现无法解释的事件 | 输入/解析失败 |
| `CorruptData` | 数据不满足格式约束 | 输入/解析失败 |

诊断实现可以把无法安全细分的解析异常归入 `CorruptData`，但不得误报成功。

## 7. 时间模型

1. `TraceReader` 读取原始事件时，使用 `GetFrequency() / trace.frequency` 把输入 tick 转换为当前 `TraceView` tick。
2. Exporter 将 `TraceView` tick 使用当前 `GetFrequency()` 转为微秒。
3. Exporter 不得再次套用输入 Trace 的 `frequency`，否则会发生二次缩放。
4. Chrome Trace JSON 的 `ts` 和 `dur` 统一使用微秒；顶层声明 `displayTimeUnit` 为 `ms` 只影响 UI 显示，不改变数值单位。
5. 同一机器、同一输入要求字节级确定性；跨机器 golden test 使用固定频率 fixture 或允许明确的时间容差。

## 8. Perfetto Track 结构

`TrackRegistry` 在输出事件前进行确定性预扫描。键为 `{category, sessionIndex, subIndex}`；pid 从 1 按 Session 顺序分配，tid 按固定类别和 subIndex 顺序分配。

```text
Perfetto process: UBA Session 0                         pid=1
|
+-- Processor 0                                        tid=1
|   +-- Process / Task slices
+-- Processor 1                                        tid=2
|   +-- Process / Task slices
+-- Work 0                                             tid=3
|   +-- Work slices
+-- Fetch                                              tid=4
|   +-- Fetch slices
+-- Store                                              tid=5
|   +-- Store slices
+-- Cache                                              tid=6
|   +-- Cache slices
+-- Counters                                           独立 tid
    +-- CPU / Memory / Network / Ping / Connection
    +-- Fetch count+bytes / Store count+bytes
    +-- Drive metrics

Perfetto process: UBA Session 1                         pid=2
+-- 按相同固定类别顺序分配 tid
```

禁止使用下列不稳定来源生成 pid/tid：

- 当前操作系统进程 ID；
- 指针地址；
- 哈希值或无序容器迭代顺序；
- 当前墙钟时间。

## 9. UBA 到 Perfetto 转换列表

Chrome Trace JSON 顶层对象固定为 `{"traceEvents":[...]}`，按需附加 `displayTimeUnit`。路线 B 使用 `M`、`X`、`C`；路线 C-Lite 第二阶段增加 `s`、`f` flow。

| UBA 事实 | `TraceView` 来源 | Chrome phase | Perfetto Track | 名称与关键 args | 可用阶段 |
|---|---|---|---|---|---|
| Session 名称 | `Session.name`、`Session.machineId` | `M` | process metadata | `process_name`；machine、clientUid | B |
| Track 名称 | Processor/Work/Transfer/Cache/Counter 类别 | `M` | thread metadata | `thread_name` | B |
| 普通 Process | `Processor.processes` 且 `type=Normal` | `X` | Processor | description；id、exitCode、remote、reuse、racing、idle、returnedReason | B |
| Cache Fetch Test/Download Process | `Processor.processes` 对应类型 | `X` | Processor | description；process_type | B |
| Task | `Processor.processes` 且 `type=Task` | `X` | Processor | description；success/exitCode | B |
| Work | `workTracks.records` | `X` | Work | description；color、日志摘要 | B |
| Fetch 明细 | `Session.fetchedFiles` | `X` | Fetch | hint；cas_key、size | B |
| Store 明细 | `Session.storedFiles` | `X` | Store | hint；cas_key、size | B |
| Fetch 聚合 | `fetchedFilesCount/Bytes` | `C` | Fetch counters | count、bytes | B |
| Store 聚合 | `storedFilesCount/Bytes` | `C` | Store counters | count、bytes | B |
| CPU | `cpuLoad` | `C` | CPU | percent | B |
| 可用内存 | `memAvail` | `C` | Memory | bytes | B |
| 网络发送/接收 | `networkSend`、`networkRecv` | `C` | Network | bytes 或 bytes/s，名称中明确口径 | B |
| Ping | `ping` | `C` | Ping | latency，名称中明确单位 | B |
| 连接数 | `connectionCount` | `C` | Connection | count | B |
| 活跃进程数 | `activeProcessCounts` | `C` | Scheduler | count | B |
| Drive busy/read/write | `Session.drives` | `C` | Drive | drive、busy%、count、bytes | B |
| Cache Write | `cacheWrites` | `X` | Cache | processId、success、stats | B |
| 端到端 Fetch operation | C-Lite 新事件 | `X` | Fetch | operationId、source、target、reason、result、logical/wire/materialized bytes | C-Lite |
| 请求端到完成端关联 | C-Lite 新事件 | `s` / `f` | Process ↔ Fetch | id=`operationId` 小写十六进制；scope=`uba.transfer` | C-Lite C2 |

### 9.1 Slice 输出规则

- `stop == ~u64(0)` 表示没有 Stop，不输出该 Slice。
- `stop < start` 表示模型不变量破坏，转换失败，不生成负时长 Slice。
- 不把未闭合 Slice 自动延长到 Trace 结束时间。
- `Task` 已由 `ProcessType_Task` 表达，不得同时输出为 Work。
- JSON 字符串必须完整转义控制字符、反斜杠和引号。

### 9.2 Counter 输出规则

- Counter 的名称必须包含稳定口径和单位，例如 `Network Send Bytes`，避免 UI 中把累计值误读为速率。
- 并行数组必须先验证采样索引范围；缺失某项时跳过该项，不读取越界数据。
- 详细路径的 Fetch 与 Store count 更新时机不对称：`E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Visualizer\Private\UbaTraceReader.cpp` 在 `FileFetchEnd` 增加 Fetch count，在 `FileStoreBegin` 增加 Store count。轻量路径在各自单次聚合事件到达时增加。转换器不得用聚合 count 与详细 Slice 数量互相校验。

## 10. C-Lite 端到端传输语义

当前 `fetchId` 是 StorageServer 生成并回收的 `u16` 分段传输标识：它在最后一个 segment 发送后即可复用，因此不能承担 Trace 全局关联。

C-Lite 新增独立 `u64 operationId`：

```text
请求 Process
    |
    | FetchBegin(requesterProcessId, requestReason)
    v
StorageServer 创建 operationId -----------------------------+
    |                                                       |
    | 返回 fetchId + 文件信息 + operationId                 |
    v                                                       |
StorageProxy 透传（若存在）                                 |
    |                                                       |
    v                                                       |
StorageClient 接收 -> 解压 -> 物化 -> FetchEnd(operationId) |
    |                                                       |
    +-- target/logical/wire/materialized bytes               |
    +-- result/systemError                                   |
    +-- receive/decompress/materialize ticks                 |
                                                            |
断连时 StorageServer 对仍活动的 operation 执行 Cancelled ---+
```

约束：

1. `operationId` 的活动表与当前 `m_activeFetches` 分离。
2. 显式 FetchEnd 与断连清理必须保证同一 operation 只关闭一次。
3. 请求上下文从 Process 层可选传递到 SessionClient 和 StorageClient；后台请求使用 Unknown/Invalid，不伪造 requester process。
4. `RetrieveCasFile` 现有尾部 `clientId` 是连接/客户端标识，不是请求者 OS process ID。
5. source、target、requestReason、result 枚举必须包含 Unknown，确保前向兼容。
6. Fetch 稳定后再为 Store 设计 operation；不得假设 Fetch 和 Store 的 wire layout 相同。

StorageProxy 同时拥有上游 Client 腿与下游 Server 腿。上游 cache-fill operationId 由上游 StorageServer 分配；StorageProxy 为每个 v6 下游 Fetch 单独分配下游 operationId，包括缓存命中与合并等待者。两类 ID 不复用、不透传，并分别由对应连接的 FetchEnd 或断连 single-close。C-Lite 通过 source/target 标识 Proxy 参与，但不新增跨腿一对多 parent flow；若未来需要合并多机或多腿 Trace，再单独设计父子 operation 关联。

## 11. 版本轴与部署顺序

Trace 文件格式与存储网络协议是两个独立版本轴：

| 版本轴 | 当前值 | C-Lite 目标 | 用途 |
|---|---:|---:|---|
| `TraceVersion` | 52 | 53 | 记录新增的端到端传输事件 |
| `StorageNetworkVersion` | 4 | 6 | 协商 operationId、请求上下文和完成事实 |

当前 `StorageNetworkVersion` 的代码权威位置是 `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Public\UbaNetwork.h`；版本 6 属于 `E:\UEWS\OpenSpec\changes\add-uba-trace-perfetto-export\specs\uba-trace-transfer-semantics\spec.md` 的目标要求，尚非当前源码事实。

版本 5 是当前源码已有但在 `StorageNetworkVersion = 4` 构建中休眠的协议脚手架：StorageServer 已接受 4-5，且 Server、StorageProxy、StorageClient 已存在 `>= 5` 的 keyHint 布局分支。推进到版本 6 会激活编译期 `>= 5` 上游路径，因此 C1 必须逐点审计，而不是只修改版本常量。

部署顺序：

```text
路线 B 独立完成
    |
    v
支持网络协议 4-6 的 Server / StorageProxy 先部署
    |
    v
使用网络协议 6 的 StorageClient 后部署
    |
    v
TraceVersion 53 数据由同一路线 B 离线管线展示
```

新 Server 与 StorageProxy 接受每连接版本 4、5、6；旧客户端继续使用旧布局。新版本 6 客户端连接旧 Server 会被拒绝，这是部署顺序必须 Server/Proxy 优先的原因。

## 12. 离线输出契约

- CLI：`UbaTraceConverter -input=<trace-file> -output=<json-file> [-force]`。
- 默认不覆盖已有输出；`-force` 才允许替换。
- 先写同目录临时文件，完成 flush/close 后原子提交；失败时保留原输出并清理临时文件。
- 退出码：`0` 成功、`1` 参数错误、`2` 输入或解析错误、`3` 输出已存在、`4` 输出写入或提交错误、`5` 数据模型不变量错误。
- 日志必须覆盖阶段边界：参数解析、输入读取、Track 预扫描、事件输出、原子提交、最终计数与失败分类。
- JSON 使用流式写入，不构建完整 DOM 或全事件向量。
- 对 1 GiB 输入，Exporter 在 Reader 已构建 `TraceView` 后增加的 RSS 不超过 `min(256 MiB, 输入大小的 25%)`；Reader 映射和 `TraceView` 内存单独统计。

退出码与资源阈值的可测试 SHALL 由 `E:\UEWS\OpenSpec\changes\add-uba-trace-perfetto-export\specs\uba-trace-perfetto-export\spec.md` 定义；本文保留其可视化管线语境下的镜像摘要。

## 13. 可实施验收视图

一个可接受的 Perfetto 导入结果必须同时满足：

1. 每个 UBA Session 对应一个稳定命名的 process。
2. Process、Task、Work、Fetch、Store 和 Cache 不互相重复。
3. 详细传输显示 Slice；轻量传输只显示 Counter；无证据时不伪造事件。
4. CPU、内存、网络、Ping、连接、进程数与磁盘指标可按时间查看。
5. 相同输入在同一机器重复转换产生字节一致 JSON。
6. 截断、未知事件、不支持版本和非法区间产生稳定非零退出码。
7. C-Lite 完成后，用户可以从请求 Process 沿 `uba.transfer` flow 定位到 Fetch 完成，并查看接收、解压、物化与失败结果。

## 14. 事实依据与待验证项

### 14.1 已由当前源码确认

- `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Public\UbaTrace.h` 定义当前 Trace 版本。
- `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaSessionServer.cpp` 把 SessionServer 的 Trace 连接到 StorageServer 与 NetworkServer。
- `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaStorageServer.cpp` 记录当前 Fetch/Store Trace，并维护现有 `fetchId` 生命周期。
- `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Visualizer\Private\UbaTraceReader.cpp` 负责当前版本检查、时间换算和 `TraceView` 构建。
- `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Visualizer\Private\UbaTraceReader.h` 证明当前详细传输、聚合传输和周期指标的数据形态。

### 14.2 逻辑分析推理（无事实依据）

- Exporter 增量内存阈值是工程验收目标，不是现状测量结果。
- C-Lite 的字段集合、协议版本 6 与 TraceVersion 53 是待实现设计，不是当前运行事实。
- `.pftrace` 仅在 Chrome Trace JSON 被实测证明存在性能或功能瓶颈后再评估；当前不预设必须迁移。

## 15. 阿卡姆剃刀与苏格拉底式自审

1. 若路线 B 能完整回答“何时、在哪个 Session、哪个 Track、耗时多久”，则不为假设中的未来查询引入新的 Runtime Trace 框架。
2. 若当前 Trace 只有聚合数据，则是否有任何事实允许恢复单次传输？答案是否定的，因此只输出 Counter。
3. 若现有 `fetchId` 会回收，它是否能安全跨越整个客户端物化阶段？答案是否定的，因此 C-Lite 使用独立 `operationId`。
4. 若 JSON 流式输出已经满足规模目标，是否需要引入 Perfetto SDK？答案是否定的。
5. 若 Store 尚未验证与 Fetch 具有相同生命周期，是否应共用协议布局？答案是否定的，因此 Store 后置并单独设计。
