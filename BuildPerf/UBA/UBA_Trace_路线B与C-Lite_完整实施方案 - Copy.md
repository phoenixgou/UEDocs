# UBA Trace 路线 B + C-lite 完整实施方案

> 文档状态：三轮审核已完成，最终结论 GO（可实施）  
> 审阅人：thomas  
> 目标分支：`release`  
> 方案结论：路线 B 先形成独立可交付的离线 UBA Trace → Perfetto 转换闭环；随后实施 C-lite 补充业务语义。两条路线不存在优劣，只有实施优先级。暂不引入运行时 Perfetto SDK、ETW Provider 或通用多 Sink 框架。

---

## 1. 执行摘要

本方案采用以下组合：

```text
路线 B：离线转换器
    +
路线 C-lite：补充少量 UBA 自定义 Trace 业务字段
    -
完整路线 C：暂不引入 Perfetto SDK / ETW Provider / ITraceSink 框架
```

实施主线如下：

```text
P0 路线 B（先交付）
现有 UBA 二进制 Trace
    -> UbaTraceAnalysis 共享分析模块
       -> 现有 TraceReader / TraceView（机械迁移）
       -> TrackRegistry + 流式 JSON Exporter
    -> UbaTraceConverter Console Target
    -> Chrome Trace Event JSON
    -> Perfetto：Process / Work / Task / Transfer Slice + Counter

P1 C-lite（路线 B 验收后实施）
UBA Runtime 新业务字段
    -> 新 TraceVersion + 新 Reader 分支
    -> 同一 Export Model
    -> Perfetto：Transfer 语义参数 + Process 到 Transfer Flow
```

路线 B 不等待 C-lite，也不伪造当前 Trace 不具备的因果 Flow。C-lite 复用路线 B 已稳定的导出模型和验证链路。

### 1.1 为什么不是纯路线 B

现有 UBA Trace 足以输出第一版时间线，但不足以可靠回答以下问题：

```text
哪个 UBA 编译进程触发了某次 PCH/CAS Fetch？
同一个 CasKey 并发 Fetch 时，Begin 与 End 如何唯一配对？
数据来自 Agent 本地内存、本地 CAS、Host，还是 StorageProxy？
内容最终进入共享映射、本地 CAS 文件，还是输出文件？
逻辑大小、网络 Payload 大小和最终 Materialize 大小分别是多少？
Fetch 是成功、取消、重试，还是解压/映射失败？
```

### 1.2 为什么不是完整路线 C

现阶段不引入以下运行时能力：

```text
Perfetto SDK
ETW Provider
通用 ITraceSink 抽象
三套并行 Trace 输出链路
所有 Detoured API 的高频埋点
每个 CAS chunk 的阶段事件
```

原因是 UBA 已经具备成熟的自定义二进制 Trace。离线转换能够满足统一展示需求，而运行时多 Sink 会扩大依赖、生命周期、跨平台构建和热路径开销。

---

## 2. 目标、非目标与成功标准

### 2.1 目标

路线 B 的独立成功标准：

1. 转换器接受现有 `TraceReader` 明确判定为可读的 Trace 版本；当前核验快照为 `TraceReadCompatibilityVersion = 6` 且 `TraceVersion = 52`。版本是区间边界，不等于区间内每个版本都已有测试夹具；历史版本缺少的字段允许降级为 `Unknown` 或不输出。
2. 不修改 UBA Runtime，输出已有 Process、Work、Task、Counter 数据。
3. 对 detailed trace，输出可正确配对的 Fetch/Store Duration Slice；对 non-detailed trace，`FileFetchLight`/`FileStoreLight` 只输出聚合计数和字节数，不伪造逐次 Slice。
4. 路线 B 不输出 Process → Fetch Flow，因为当前 Trace 不包含稳定的请求进程关联。
5. 转换器提供边界日志和稳定退出码，能够定位不兼容版本、损坏事件、未配对事件、降级模式和输出统计。
6. 相同输入和参数必须产生字节级一致的 JSON，支持 golden file 测试。

C-lite 的后续成功标准：

> 对于一次远程 `cl.exe` 编译，能够确定与它关联的 CAS/PCH Transfer，并看到来源、目标、三类大小、结果和总耗时。

C-lite 在路线 B 验收后独立提升 TraceVersion，并保持历史版本读取；C-lite 的实施失败不得回退或阻塞路线 B 已交付能力。

### 2.2 非目标

首版不负责：

```text
替代 UbaVisualizer
采集 Windows 内核 Page Fault / Context Switch / TCP Retransmission
复刻 ETW 的内核观测能力
追踪所有 MapViewOfFile / CreateFile 调用
记录 HANDLE、虚拟地址、PFN 等低层地址信息
精确同步多台独立机器的非同源时钟
实时把 UBA 事件直接写入 Perfetto service
在路线 B 中推断当前 Trace 未记录的来源、目标、结果或 Process → Fetch 因果关系
```

逻辑分析推理(无事实依据，工程取舍)：UBA Trace 应保留业务语义边界。内核调度、Page Fault、文件系统与网络协议栈细节应由 ETW 等系统 Trace 补充，而不是在 UBA 中重复实现。

---

## 3. 当前 UBA Trace 能力与源码边界

逻辑分析推理(有事实依据)：当前 UBA 已经具有版本化、结构化、实时与离线兼容的自定义 Trace，而不是普通文本日志。

核心源码位置：

```text
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Public\UbaTrace.h
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaTrace.cpp
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Public\UbaTraceChannel.h
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Visualizer\Private\UbaTraceReader.h
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Visualizer\Private\UbaTraceReader.cpp
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Visualizer\Private\UbaVisualizer.h
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Visualizer\Private\UbaVisualizer.cpp
```

对应仓库源码仅作为参考；实现与审核以 `E:\UEWS\5.8.0` 本地源码为事实源：

```text
https://github.com/phoenixgou/UnrealEngine5.8/blob/release/Engine/Source/Programs/UnrealBuildAccelerator/Common/Public/UbaTrace.h
https://github.com/phoenixgou/UnrealEngine5.8/blob/release/Engine/Source/Programs/UnrealBuildAccelerator/Common/Private/UbaTrace.cpp
https://github.com/phoenixgou/UnrealEngine5.8/blob/release/Engine/Source/Programs/UnrealBuildAccelerator/Common/Public/UbaTraceChannel.h
https://github.com/phoenixgou/UnrealEngine5.8/blob/release/Engine/Source/Programs/UnrealBuildAccelerator/Visualizer/Private/UbaTraceReader.h
https://github.com/phoenixgou/UnrealEngine5.8/blob/release/Engine/Source/Programs/UnrealBuildAccelerator/Visualizer/Private/UbaTraceReader.cpp
```

当前事件已经覆盖：

| 领域 | 现有事件示例 |
|---|---|
| Session | `SessionAdded`、`SessionUpdate`、`SessionDisconnect` |
| Process | `ProcessAdded`、`ProcessExited`、`ProcessReturned`、`ProcessRace` |
| CAS Fetch | `FileFetchBegin`、`FileFetchSize`、`FileFetchEnd` |
| CAS Store | `FileStoreBegin`、`FileStoreEnd` |
| Cache | `CacheBeginFetch`、`CacheEndFetch`、`CacheBeginWrite`、`CacheEndWrite` |
| Scheduler | `SchedulerUpdate`、`SchedulerKillProcess` |
| 通用工作 | `WorkBegin`、`WorkHint`、`WorkEnd`、`TaskBegin`、`TaskEnd` |
| 系统状态 | CPU、内存、网络、ping、磁盘读写与忙碌率 |

当前核验快照中的 Trace 版本：

```text
TraceVersion = 52
TraceReadCompatibilityVersion = 6
```

新增 schema 时必须提升 Trace 版本，并继续支持历史版本读取。

---

## 4. 总体架构

```text
                              +------------------------------+
                              | 路线 B：不改 Runtime         |
                              | UbaTraceAnalysis             |
现有 UBA Trace -------------->+ Reader -> Registry -> JSON   +--> Perfetto
                              +------------------------------+
                                             ^
                                             |
                              +--------------+---------------+
                              | C-lite：后续提升 TraceVersion |
UBA Runtime 新 Transfer 语义 ->+ Reader 兼容 -> 同一模型      |
                              +------------------------------+
```

### 4.1 首版输出格式决策

首版输出 **Chrome Trace Event JSON**，并由 Perfetto UI 加载。

原因：

```text
不向 UBA Runtime 增加 Perfetto 依赖
离线实现简单
支持 Process/Thread Metadata、Slice、Counter 和 Flow
便于人工审核原始输出
便于编写 golden file 测试
```

后续只有在以下条件成立时，才增加原生 Perfetto protobuf `.pftrace`：

```text
JSON 文件体积或解析性能成为实际瓶颈
需要 Perfetto proto 独有的数据模型
已有稳定的字段映射和兼容测试
项目接受新增 Perfetto schema/tooling 依赖
```

逻辑分析推理(无事实依据，工程取舍)：首版同时支持 JSON 与 `.pftrace` 会扩大实现与测试面，不符合简洁优先原则。

### 4.2 Chrome Trace Event JSON 契约

首版固定输出顶层对象 `{"traceEvents":[...]}`。所有 `ts`/`dur` 使用微秒数值，禁止把 `displayTimeUnit` 当作时间单位换算器。

| UBA 模型 | `ph` | 必填字段 | 决策 |
|---|---:|---|---|
| Process/Thread 名称 | `M` | `pid`、`tid`、`name`、`args.name` | 在对应轨道首个事件前输出 |
| Duration Slice | `X` | `pid`、`tid`、`ts`、`dur`、`cat`、`name` | 仅输出已知且非负的完整区间 |
| Counter | `C` | `pid`、`tid`、`ts`、`name`、`args` | 每个 `Session.updates[i]` 与平行值数组同索引导出 |
| Flow Start/End | `s` / `f` | `pid`、`tid`、`ts`、`id`、`scope`、`bp=e` | 仅 C-lite C2 关联完成后启用；`id` 为小写十六进制 `operationId`，`scope="uba.transfer"` |

首先执行不输出事件的 `TrackRegistry` 预扫描，再流式输出 JSON。`TrackKey = {category, sessionIndex, subIndex}`，登记顺序固定为 Session 向量顺序→类别固定顺序→子索引顺序；每个 Session 分配一个从 1 递增的 `pid`，其内轨道按固定顺序分配从 1 递增的 `tid`。禁止使用指针、容器哈希值、当前进程 PID 或当前时间。JSON 数值格式必须 locale-independent，字符串必须正确转义控制字符、反斜杠、引号和 Unicode。

---

## 5. 路线 B：离线转换器设计

### 5.1 解析策略

转换器必须复用现有 `TraceReader`，禁止实现第二套 UBA 二进制解析器。

```text
UBA Trace 文件
    -> TraceReader::ReadFileForExport
       -> TraceReadResult（显式终止原因）
    -> TraceView
    -> TrackRegistry + TraceView 只读迭代
    -> Chrome Trace Event JSON
```

`TraceReader::ReadFile(..., replay=false)` 的布尔返回值会把 `TraceType_Summary` 的正常终止与错误共用为 `false`。仅检查 `{readOk, TraceView.finished}` 仍不足够：`TraceView::Clear()` 会把 `finished` 设为 `true`，而输入打开失败或版本拒绝可在 Reader 把 `finished` 置为 `false` 之前返回。

路线 B 因此必须在共享 Reader 模块增加窄接口，不依赖日志文本推断：

```cpp
enum class TraceReadStatus : u8
{
    CompleteSummary,
    CompleteEof,
    InputOpenFailed,
    UnsupportedVersion,
    Truncated,
    UnknownEvent,
    CorruptData
};

struct TraceReadResult
{
    TraceReadStatus status;
    u32 version = 0;
    u64 failureOffset = 0;
};
```

`ReadFileForExport` 在每个退出分支显式设置状态；解析到 `TraceType_Summary` 返回 `CompleteSummary`，所有完整事件恰好消耗至文件末尾时返回 `CompleteEof`，事件负载在 EOF 处不完整才是 `Truncated`。`version` 在读取头部后设置，版本失败不得伪装成 Summary/EOF 成功。既有 `ReadFile` 布尔接口作为 Visualizer 兼容包装保留；Exporter 只使用类型化结果。`UnknownEvent`/`CorruptData`/`Truncated` 用于诊断，共同映射 CLI 退出码 2；如果现有 Reader 分支无法在不扩大改动的前提下精确区分，允许统一归类为 `CorruptData`，不得根据 Logger 文本猜测。

### 5.2 共享模块与 Console Target

```text
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\UbaTraceConverter.Target.cs
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\TraceAnalysis\UbaTraceAnalysis.Build.cs
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\TraceAnalysis\Public\UbaTraceReader.h
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\TraceAnalysis\Private\UbaTraceReader.cpp
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\TraceAnalysis\Public\UbaTraceExporterPerfetto.h
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\TraceAnalysis\Private\UbaTraceExporterPerfetto.cpp
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\TraceAnalysis\Private\UbaJsonStreamWriter.h
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\TraceAnalysis\Private\UbaJsonStreamWriter.cpp
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\TraceConverter\UbaTraceConverter.Build.cs
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\TraceConverter\Private\UbaTraceConverterMain.cpp
```

`UbaTraceAnalysis` 是 Desktop Program 共享模块，仅依赖 `UbaCommon` 和现有 Reader 机械迁移所必需的平台依赖。`UbaVisualizer`、`UbaTraceConverter` 和 `UbaTest` 都依赖它；转换器不依赖 `UbaVisualizer`，不编译或初始化 GUI。`E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\UbaTraceConverter.Target.cs` 设置 `LaunchModuleName = "UbaTraceConverter"` 并调用 `UbaAgentTarget.CommonUbaSettings(this, Target)`；`bIsBuildingConsoleApplication = true` 由该公共配置继承，不需要第二套 Target 配置。

第一个实现提交只把 `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Visualizer\Private\UbaTraceReader.h` 和 `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Visualizer\Private\UbaTraceReader.cpp` 机械迁移到第 5.2 节列出的 `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\TraceAnalysis` 计划目录，并更新 include/模块依赖与编译验证；禁止同时重命名数据结构、清理 `BitmapCache` 或改变解析行为。第二个提交再增加类型化读取结果、Exporter、最小 `UbaJsonStreamWriter` 和 Console Target。这一分层使 `UbaTest` 可以直接构造 `TraceView` 测试 Exporter，不需要穿透 GUI 私有模块。

运行时代码不得硬编码工作区绝对路径。输入文件和输出文件必须由调用方或命令行参数传入。

### 5.3 命令行与文件事务契约

```text
UbaTraceConverter -input=<trace-file> -output=<json-file> [-force]
```

规则：

1. `-input`、`-output` 必填；解析规范化后两者不得相同。
2. 输出已存在时默认拒绝；只有 `-force` 允许替换。
3. 先在输出同目录写 `<output>.tmp.<pid>`，flush/close 成功后再原子提交；失败时删除本次临时文件，不破坏既有输出。
4. 不提供隐式输出路径、批量目录扫描、pretty-print 或第二种格式，避免扩大首版接口。
5. 退出码固定为：`0` 成功；`1` 参数错误；`2` 输入打开或解析失败；`3` 输出已存在且未指定 `-force`；`4` 输出写入或原子提交失败；`5` 导出模型不变量失败。
6. non-detailed 输入仍返回 `0`，但必须输出 `mode=aggregate-only` 警告和聚合统计；该行为是能力降级，不是转换失败。输入中没有任何 Transfer 证据时返回 `0`，记录 `mode=no-transfer-evidence`，不输出 Transfer Slice/Counter。

### 5.4 Perfetto 轨道模型

| UBA 数据 | Perfetto 表达 | 轨道建议 |
|---|---|---|
| Host Session | Process Metadata / Group | `UBA Host` |
| Agent Session | Process Metadata / Group | 每个 Agent 一个组 |
| UBA 编译 Process | Duration Slice | 按执行 Session/Processor 分轨 |
| CAS Fetch | Duration Slice | 每个 Agent 的 Fetch Track |
| CAS Store | Duration Slice | 每个 Agent 的 Store Track |
| Cache 操作 | Duration Slice | Cache Track |
| Work | Duration Slice | `WorkTrack.records` 导出到 Work Track |
| Task | Duration Slice | 只复用 Process 路径，以 `ProcessType_Task` 分类；不再重复输出为 Work |
| Process → Fetch | Flow | 路线 B 不输出；C-lite 具备稳定关联后输出 |
| CPU Load | Counter | 每个 Session Counter Track |
| 可用内存 | Counter | 每个 Session Counter Track |
| 网络发送/接收 | Counter | 累计值或派生速率 |
| Drive Update | Counter | 每个盘符独立 Track |
| Proxy 事件 | Instant Event / Metadata | Session Track |

Fetch/Store Slice 只来自 `Session.fetchedFiles` / `Session.storedFiles` 中 `stop != ~u64(0)` 且 `stop >= start` 的条目。non-detailed 的 `FileFetchLight` / `FileStoreLight` 不进入这两个向量，只能导出 `fetchedFilesCount`、`storedFilesCount` 和对应字节聚合 Counter。

Reader 在 detailed 模式也会填充 `fetchedFilesCount`/`storedFilesCount`，且口径不对称：fetch 在 End 计数，store 在 Begin 计数。因此 detailed 模式以向量派生 Slice，不使用聚合计数校验 Slice 数；聚合 Transfer Counter 仅在 aggregate-only 模式输出。模式判定规则固定为：任一 Fetch/Store 向量非空则为 `detailed`；向量全空但任一聚合计数非零则为 `aggregate-only`；两者皆无证据则记录 `no-transfer-evidence`，不猜测采集模式。

### 5.5 时间换算与确定性

Trace 文件头实际包含：

```text
systemStartTimeUs（TraceVersion >= 18）
frequency（TraceVersion >= 18）
realStartTime 对应的原始 m_startTime
```

`TraceReader` 会用 `timeMultiplier = GetFrequency() / trace.frequency` 把事件偏移换算到当前进程的 UBA tick 频率域。Exporter 消费 `TraceView` 后不得再次使用文件头 `frequency` 缩放；应把 TraceView 中的事件 tick 统一转换为微秒：

```text
perfetto_us = trace_view_event_tick * 1,000,000 / GetFrequency()
```

实现应使用 UBA 现有时间辅助函数或等价的溢出安全换算，禁止假设固定频率。TraceView 中事件时间已经相对 Trace 起点，JSON 直接以 Trace 起点为 `ts=0`；`TraceView.startTime`、`GetTime()` 和导出时当前系统时间不得参与输出。`traceSystemStartTimeUs` 只允许作为可选元数据，不参与相对时间轴换算。

单个 UBA Trace 文件可以使用同一时间域直接转换。多个独立机器 Trace 的精确合并不属于首版范围。

相同输入必须按 Session、Processor、Process、WorkTrack、FileTransfer 和 Counter 的向量顺序稳定遍历；无序容器内容必须先按稳定业务键排序后输出。浮点或十进制微秒序列化必须与 locale 无关。

### 5.6 JSON 写入与异常事件策略

Exporter 使用 `UbaTraceAnalysis` 私有的最小 `UbaJsonStreamWriter` 流式写出 JSON，不允许构造完整 JSON DOM，也不允许把所有事件复制到中间向量。Exporter 阶段在已构建 `TraceView` 之上的增量内存复杂度必须为 `O(track count + writer buffer)`，不得随事件数线性额外增长。复用 Reader 必然产生的只读文件映射和 `TraceView O(events)` 不属于该增量，但必须单独记录总峰值。

逻辑分析推理(无事实依据，验收阈值)：对 1 GiB 输入夹具，Exporter 阶段相对 `ReadFileForExport` 返回后 RSS 基线的额外峰值常驻内存不超过 `min(256 MiB, 输入大小的 25%)`。测试必须测量导出前 RSS 基线与导出阶段峰值之差；总峰值约为文件映射 + `TraceView O(events)` + Exporter 增量。该数值是工程门槛，不是当前实测结果。

边界日志必须包括：

```text
[TraceExport] Start input=<path> output=<path>
[TraceExport] Input version=<version>
[TraceExport] Input mode=<detailed|aggregate-only|no-transfer-evidence>
[TraceExport] Parsed processes=<n> fetches=<n> stores=<n> counters=<n>
[TraceExport] Degraded fetch_count=<n> fetch_bytes=<n> store_count=<n> store_bytes=<n>
[TraceExport] Unmatched begin=<n> unmatched end=<n>
[TraceExport] Output slices=<n> counters=<n> flows=<n>
[TraceExport] Complete bytes=<n>
[TraceExport] Failed stage=<stage> error=<error>
```

日志中的路径来自调用参数，代码中不得硬编码绝对路径。

未配对规则固定如下：

```text
stop == ~u64(0)                  -> 不输出 Duration；计入 unmatched begin
stop < start                     -> 不输出 Duration；计入 invalid duration；最终退出码 5
End 找不到 active CasKey         -> Reader 当前不会生成条目；适配层只能依据 Logger/统计报告
同 CasKey 并发导致配对歧义      -> 不猜测；输出能确认的条目，其余计入 unmatched，待 C-lite operationId 解决
```

路线 B 禁止把未配对传输裁剪到 Trace 结尾，因为这会制造并不存在的耗时结论。

---

## 6. C-lite：必须补充的 Trace 语义

C-lite 只在路线 B 验收后开始。当前 `FileFetch*`/`FileStore*` Trace 调用点位于 `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaStorageServer.cpp`，主要使用 `connectionInfo.GetId()` 形成连接 `clientId`；它不是 UBA `requesterProcessId`。`E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaStorageClient.cpp` 当前没有对应 Trace 调用。因此服务端现有调用点不能天然知道请求 `cl.exe`、客户端最终 mapping 类型或 materialize 结果。

当前源码快照中，`E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Public\UbaNetwork.h` 定义 `StorageNetworkVersion = 4`；`E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaStorageServer.cpp` 按每连接版本窗口接受 4～5，不是全局严格相等。服务端已有 `>= 5` 的 `keyHint` 读取分支，而客户端以编译时版本决定是否发送。

C-lite 必须按以下顺序推进，不得把服务端与客户端才知道的字段混成“单点直接可加”的事件：

```text
C1-Fetch：StorageNetworkVersion 6 + TraceVersion 53
    -> 服务端生成 operationId，客户端回显 End
    -> 新服务端按连接版本兼容 4/5/6
    -> 断连与所有终止路径只结束一次

C2-Context：可选 TraceRequestContext 穿透
    -> requesterProcessId / requestReason 从 SessionClient/ProcessImpl 传到 FetchBegin
    -> 后台请求保持 Invalid，只对有效关联输出 Flow

C3-Client：客户端 materialization 观测 + ProcessExecutionInfo
    -> target/materializedBytes/result/ActiveTicks 从客户端事实回传
    -> osProcessId 与 ETW 对齐

C4-Store：Fetch 闭环验收后独立实施 Store 协议与 Trace
    -> 复用语义模型，不假设 Store RPC 与 Fetch RPC 线上布局相同
```

`TraceVersion` 控制 Trace 文件 schema，`StorageNetworkVersion` 控制 Storage RPC 线上布局；它们是两条独立版本轴，不得共用常量或隐式绑定。任一轴的二进制布局扩展都必须提升对应版本并增加显式分支。

### 6.1 第一优先级字段

| 字段 | 类型 | 目的 | 决策 |
|---|---|---|---:|
| `operationId` | `u64` | 唯一关联 Begin/End 和 Flow | 必加 |
| `source` | enum | 区分本地内存、本地文件、Host、Proxy | 必加 |
| `target` | enum | 区分共享映射、独立映射、本地 CAS、输出文件 | C3 客户端观测后增加 |
| `logicalBytes` | `u64` | 解压后的业务内容大小 | 必加 |
| `wireBytes` | `u64` | UBA 应用层网络 Payload 大小 | 必加 |
| `materializedBytes` | `u64` | 最终写入 mapping 或文件的大小 | C3 客户端观测后增加 |
| `result` | enum | 成功、取消、重试和失败分类 | 必加 |
| `systemError` | `u32` | 记录平台错误码 | 建议 |
| `compressed` | `bool` | 解释逻辑大小与 wire 大小差异 | 建议 |

### 6.2 第二优先级字段

| 字段 | 类型 | 目的 | 决策 |
|---|---|---|---:|
| `requesterProcessId` | `u32` | 关联编译 Process 与 CAS/PCH Fetch | C2，需要上下文/协议传播 |
| `osProcessId` | `u32` | 与 ETW 的 CPU/Page Fault/File I/O 对齐 | 高优先级，独立提交 |
| `executionType` | enum | 区分 Native、Detoured、Remote、Cache | 建议 |
| `requestReason` | enum | 区分按需读取、Crawler、预热、自动更新 | 建议 |

### 6.3 `operationId` 的必要性

现有 Fetch/Store 主要依靠 `CasKey` 配对。同一 Session 对相同 `CasKey` 并发操作时，`CasKey` 不是唯一操作实例标识。

建议：

```cpp
using TraceOperationId = u64;
```

要求：

```text
Trace 文件范围内唯一
0 保留为 Invalid；使用原子计数从 1 单调增长，不要求跨进程、跨机器全局唯一
Begin 创建 ID，End 必须携带同一个 ID
失败、取消和重试也必须结束原 operation
重试应创建新 operationId，并通过可选 parentOperationId 关联；首版可暂不增加 parentOperationId
```

### 6.4 请求进程关联

增加可选值：

```cpp
static constexpr u32 InvalidProcessId = ~u32(0);
```

以下 Fetch 允许没有直接请求进程：

```text
Dependency Crawler 预取
CAS 预热
Agent 自动更新
缓存恢复
后台存储维护
```

不得用 `0` 表示“无进程”，也不得强制所有 Fetch 都关联进程。

### 6.5 来源与目标枚举

建议定义：

```cpp
enum class TraceTransferSource : u8
{
    Unknown,
    LocalMemory,
    LocalCasFile,
    DirectStorageServer,
    StorageProxy,
    HintFallback
};

enum class TraceMaterializationTarget : u8
{
    Unknown,
    SharedMapping,
    IndependentMapping,
    LocalCasFile,
    OutputFile
};
```

必须保留 `Unknown`，用于：

```text
读取历史 Trace
不完整错误路径
尚未覆盖的新代码路径
```

### 6.6 结果枚举

```cpp
enum class TraceOperationResult : u8
{
    Success,
    Cancelled,
    Retry,
    NotFound,
    Disallowed,
    NetworkError,
    DecompressionError,
    MaterializationError,
    UnknownError
};
```

`systemError` 仅补充具体平台错误；稳定统计必须依据 `TraceOperationResult`，不能依赖 Windows 错误码。

### 6.7 StorageNetworkVersion 6 Fetch 线上契约

版本 4/5 连接保持现有字节布局。仅当 `connectionInfo.GetStorageVersion() >= 6` 时读写新字段：

```text
FetchBegin request（v6 在现有 hint 之后追加）
    requesterProcessId : u32，InvalidProcessId 表示无直接请求进程
    requestReason      : u8

FetchBegin response header（v6 追加）
    operationId        : u64，0 表示 Trace 关闭或未建立追踪

FetchEnd request（v6）
    operationId        : u64
    target             : u8
    logicalBytes       : u64
    wireBytes          : u64
    materializedBytes  : u64
    result             : u8
    systemError        : u32
    compressed         : bool
    receiveActiveTicks : u64
    decompressActiveTicks : u64
    materializeActiveTicks : u64
```

v4/v5 `FetchEnd` 继续使用现有 `CasKey` 路径；v6 服务端以 `operationId` 查找活动传输并结束 TraceVersion 53 新事件，不再以 `CasKey` 作为唯一配对键。为了防止升级期间的格式漂移，v6 End 字段一次定义完整；C2/C3 未实施时使用 `Invalid`/`Unknown`/0，不改变布局。

### 6.8 operationId 生命周期

现有 Storage 协议已有服务端生成的可回收 `u16 fetchId`：它由 `PopId()`/`PushId()` 管理，FetchBegin 响应已携带，用于 `FetchSegment` 传输，并在最后一个 segment 后从 `m_activeFetches` 移除并回收；它还有 `FetchCasIdDone`/`FetchCasIdDisallowed` 哨兵值。该 ID 早于 trace-only `FetchEnd` 和客户端 materialization 完成而失效，既不是 Trace 文件范围唯一，也不得改名复用为 `operationId`。

v6 必须另建 `u64 operationId`。正常 FetchBegin 响应在现有 `fetchId(u16) + fileSize(7-bit) + flags(u8)` 之后、首批 payload 之前追加 `operationId(u64)`，并同步调整 Server/Proxy 的响应头大小计算。`operationId` active 表独立于现有 `m_activeFetches`：前者存活到 v6 FetchEnd 或断连，后者仍在 segment 发送完成后回收。Proxy 的 `SendFetchBeginResponse` 和上游响应路径必须按连接版本生成或转发同一 `operationId`。

1. Trace 开启时，服务端用原子计数器从 1 分配 `operationId`；0 保留。
2. 服务端在返回 FetchBegin 响应前登记 active operation，客户端收到非 0 ID 后建立 completion guard。
3. completion guard 保证成功、取消、重试、解压失败、映射失败和所有早退分支最多发送一次 End。重试创建新 ID。
4. 连接中断时，`OnDisconnected` 将服务端仍 active 的 ID 结束为 `Cancelled`；与显式 End 的竞争由同一 active 表的单次移除保证不重复。
5. 如果响应未能把 ID 交给客户端，服务端仍必须在发送失败或断连路径关闭该 ID，并记录 `unreturnable_operation` 诊断日志。

active operation 表的键为 `operationId`，值中保留 `CasKey`、连接 ID、起始时间和服务端已知字段。`CasKey` 仍作为业务参数，但不负责唯一性。

### 6.9 TraceRequestContext 传播边界

```cpp
struct TraceRequestContext
{
    u32 requesterProcessId = InvalidProcessId;
    TraceFileRequestReason reason = TraceFileRequestReason::Unknown;
};
```

该结构以可选值沿 `ProcessImpl` 发起路径→`SessionClient`→`StorageClient::RetrieveCasFile` 传播；现有 `RetrieveCasFile` 尾部 `clientId` 参数不得改名后冒充 `requesterProcessId`。Crawler、预热、自动更新和后台维护不构造伪进程，保持 Invalid/Unknown。

### 6.10 部署与兼容矩阵

| 客户端 | 服务端 | 结果 |
|---|---|---|
| v4/v5 | 新 v6 服务端 | 允许；按连接版本走旧布局 |
| v6 | 新 v6 服务端 | 允许；走 v6 operationId 布局 |
| v6 | 旧服务端 | 拒绝；旧窗口上限为 5，禁止降级解析 |

发布顺序固定为服务端先于客户端。新服务端将接受窗口扩展为 4～6，并依赖每连接版本门控字段。`E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaStorageProxy.cpp` 当前已核验为 `clientVersion != StorageNetworkVersion` 即拒绝的严格相等策略。C1-Fetch 必须同提交把 Proxy 改为 4～6 窗口并按每连接版本读写/转发字段；Proxy 不允许成为未声明的严格配套部署例外。

---

## 7. 推荐事件模型

以下接口是设计基线，不是要求实施时无条件照抄；实施前应以最小调用面修改为准。

该基线描述 C1～C3 的最终归一化模型，不表示所有字段在 `UbaStorageServer` 的单个 End 调用点都可获得。C1、C2、C3 每阶段只写其事实来源能够确认的字段，其余保持 `Unknown`/0，并在后续 TraceVersion 中扩展；禁止为一次性填满结构而跨层猜值。

```cpp
using TraceOperationId = u64;

struct TraceFileTransferResult
{
    TraceOperationId operationId = 0;
    u32 requesterProcessId = InvalidProcessId;

    TraceTransferSource source = TraceTransferSource::Unknown;
    TraceMaterializationTarget target = TraceMaterializationTarget::Unknown;
    TraceOperationResult result = TraceOperationResult::UnknownError;

    u64 logicalBytes = 0;
    u64 wireBytes = 0;
    u64 materializedBytes = 0;

    u64 receiveActiveTicks = 0;
    u64 decompressActiveTicks = 0;
    u64 materializeActiveTicks = 0;

    u32 systemError = 0;
    bool compressed = false;
};
```

建议事件接口：

```cpp
TraceOperationId FileTransferBegin(
    TraceFileTransferDirection direction,
    u32 clientId,
    u32 requesterProcessId,
    const CasKey& casKey,
    StringView hint,
    TraceFileRequestReason reason);

void FileTransferEnd(const TraceFileTransferResult& result);

void ProcessExecutionInfo(
    u32 ubaProcessId,
    u32 osProcessId,
    u32 executorSessionId,
    ProcessExecutionType executionType,
    bool detoured);
```

### 7.1 兼容策略

不立即删除旧事件：

```text
FileFetchBegin
FileFetchSize
FileFetchEnd
FileStoreBegin
FileStoreEnd
```

推荐采用以下迁移：

```text
TraceVersion 52 及以前：Reader 按旧事件解析
TraceVersion 53（C1）：Reader 解析带 operationId 的新 FileTransfer 事件
后续 C2/C3 若继续扩展二进制字段：每次扩展都提升 TraceVersion 并增加显式版本分支
Exporter 将两种模型归一化到同一 Export Model
Visualizer 在新模型稳定前继续兼容旧事件
```

逻辑分析推理(无事实依据，工程取舍)：在同一版本中同时写旧事件和新事件会造成重复数据和额外开销。优先通过版本分支兼容，而不是长期双写。

---

## 8. 阶段耗时：先累计，不先制造高频事件

客户端 materialization 的真实执行可能包含并发生效的阶段：

```text
网络分块接收
解压
拷贝到 mapping
文件写入
代理重试
```

第一阶段不增加每个 chunk 的 Begin/End。建议在 `FileTransferEnd` 中汇总：

```cpp
u64 receiveActiveTicks;
u64 decompressActiveTicks;
u64 materializeActiveTicks;
```

定义必须稳定：

```text
receiveActiveTicks
    = StorageClient 接收和处理网络 Payload 的累计活动时间

decompressActiveTicks
    = 解压代码实际执行的累计活动时间

materializeActiveTicks
    = 写 mapping、内存复制或写本地文件的累计活动时间
```

这些值可以作为 Slice 参数和汇总指标，但不能伪造成严格连续的子 Slice，因为阶段可能交错或重叠。

只有第一版数据仍无法定位问题时，才在现有 `detailedTrace` 模式下增加：

```cpp
FileTransferStageBegin(operationId, stage);
FileTransferStageEnd(operationId, stage);
```

本节属于 C3，不属于路线 B。累计字段只有在客户端或明确拥有该阶段的代码路径中测量后才能写入；服务端不得代填客户端阶段耗时。

---

## 9. PCH / MapView 埋点决策

### 9.1 首版不追踪所有映射调用

不增加全量 `MapViewOfFile` Trace，原因：

```text
调用频率高
Detours 热路径敏感
普通映射对 PCH 因果分析价值有限
原始 HANDLE 和虚拟地址难以跨进程稳定解释
Trace 体积与隐私风险上升
```

### 9.2 按数据决定是否增加特殊映射事件

后续仅考虑追踪：

```text
UBA memory file
UBA mapped file
FILE_MAP_COPY
MSVC PCH 指定 BaseAddress 特殊路径
Independent mapping
映射失败
```

候选字段：

```cpp
u64 operationId;
u32 requesterProcessId;
u64 mappingSize;
u64 mappingOffset;
SharedMemoryMapType mapType;
bool independentMapping;
bool success;
u32 systemError;
```

默认不记录：

```text
原始 HANDLE
虚拟地址
PFN
完整绝对业务文件路径
每次 Unmap 调用
```

---

## 10. 进程与 ETW 关联

现有 UBA `processId` 是逻辑进程标识，不等同于 Windows PID。

建议新增独立事件：

```cpp
ProcessExecutionInfo(
    u32 ubaProcessId,
    u32 osProcessId,
    u32 executorSessionId,
    ProcessExecutionType executionType,
    bool detoured);
```

不直接扩展 `ProcessAdded` 的原因：

```text
ProcessAdded 可能发生在排队或逻辑对象创建阶段
Windows PID 只有 CreateProcess 成功后才存在
远程、Native、FromCache 的生命周期不同
```

该事件为后续关联以下 ETW 数据建立桥梁：

```text
CPU Scheduling
Context Switch
Page Fault
File I/O
Working Set
Thread 生命周期
```

首版纯 UBA Perfetto 时间线不依赖 `osProcessId`；但 PCH、Page Fault 和系统 Trace 合并需要它，因此应作为独立高优先级提交。

---

## 11. 实施边界与修改位置

### 11.1 路线 B

```text
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\UbaTraceConverter.Target.cs
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\TraceAnalysis\UbaTraceAnalysis.Build.cs
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\TraceAnalysis\Public\UbaTraceReader.h
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\TraceAnalysis\Private\UbaTraceReader.cpp
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\TraceAnalysis\Public\UbaTraceExporterPerfetto.h
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\TraceAnalysis\Private\UbaTraceExporterPerfetto.cpp
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\TraceAnalysis\Private\UbaJsonStreamWriter.h
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\TraceAnalysis\Private\UbaJsonStreamWriter.cpp
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\TraceConverter\UbaTraceConverter.Build.cs
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\TraceConverter\Private\UbaTraceConverterMain.cpp
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Visualizer\UbaVisualizer.Build.cs
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Test\UbaTest.Build.cs
```

`UbaVisualizer` 和 `UbaTest` 只更新模块依赖与 include；现有 Windows/Linux/Mac Visualizer 入口不增加 CLI 分支。计划中的新路径在实施前不存在属于预期状态，实施时代码必须使用工作区相对 include/构建路径，不得硬编码本文档的绝对路径。

### 11.2 C-lite schema

```text
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Public\UbaTrace.h
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaTrace.cpp
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\TraceAnalysis\Public\UbaTraceReader.h
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\TraceAnalysis\Private\UbaTraceReader.cpp
```

### 11.3 C-lite 协议、请求上下文与 Transfer 埋点

```text
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Public\UbaNetwork.h
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Public\UbaStorageClient.h
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Public\UbaStorageServer.h
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Public\UbaStorageProxy.h
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaStorageServer.cpp
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaStorageClient.cpp
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaStorageProxy.cpp
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaSessionClient.cpp
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaProcess.cpp
```

现有 `FileFetch*` 和 `FileStore*` Trace 调用点已核验集中于 `E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Common\Private\UbaStorageServer.cpp`。C2/C3 只沿请求发起和 materialization 事实路径增加可选上下文与统计；不得顺手重构无关 Storage 代码。

---

## 12. 分阶段提交计划

### 提交 B1：机械迁移 Reader 到共享模块

范围：

```text
新增 E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\TraceAnalysis\UbaTraceAnalysis.Build.cs
把 E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Visualizer\Private\UbaTraceReader.h 和 E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Visualizer\Private\UbaTraceReader.cpp 移到第 11.1 节的完整计划路径，只修改 include 和模块依赖
UbaVisualizer 和 UbaTest 改为依赖 UbaTraceAnalysis
不改解析行为，不清理 BitmapCache，不新增 Exporter
```

验收：`UbaVisualizer` 与 `UbaTest` Win64 Development 编译通过，现有 `UbaTest` 全量测试通过。当前源码没有 Reader 专项测试，不得声称已有该覆盖；B1 以机械移动 diff、Visualizer 打开同一 v52 样本的前后结果与全量构建/测试证明行为无变化。

### 提交 B2：路线 B 转换闭环，零 Runtime schema 修改

范围：

```text
新增独立 UbaTraceConverter Console Target 和统一 CLI 入口
新增 TraceReadResult 类型化终止原因
接受当前 Reader 判定可读的 TraceVersion
输出 Session / Process / Work / Task / Counter
detailed 输入输出有效 Fetch / Store Slice
non-detailed 输入只输出 Fetch / Store 聚合 Counter，并记录 aggregate-only 降级
首版输出 Chrome Trace Event JSON
实现 TrackRegistry、确定性遍历、JSON 转义和微秒时间换算
使用 UbaJsonStreamWriter，不构造全量事件中间模型或 DOM
实现临时文件 + 原子提交、覆盖保护、稳定退出码
输出转换统计、未配对事件和无效 Duration 日志
```

验收：

```text
v52 detailed、v52 non-detailed 和至少一个历史版本样本可以被转换
输出可以由 Perfetto UI 打开
Slice 数等于 TraceView 中 `stop != ~u64(0)` 且 `stop >= start` 的有效条目数
aggregate-only 模式的聚合计数等于 TraceView 的 `fetchedFilesCount` / `storedFilesCount`
相同输入和参数连续转换两次，输出 SHA-256 一致
路线 B 输出中不存在 Process → Fetch Flow
1 GiB 夹具的导出阶段相对解析后 RSS 基线的额外峰值满足第 5.6 节工程门槛
```

### 提交 C1：Fetch v6 闭环 + TraceVersion 53

范围：

```text
StorageNetworkVersion 6；新服务端按连接版本接受 4～6
FetchBegin 请求/响应和 FetchEnd v6 线上布局
服务端生成 operationId，客户端 completion guard 回显
服务端可确认的 source
logicalBytes / wireBytes
result / systemError / compressed，客户端字段暂以 Unknown/0 填充
OnDisconnected 以 Cancelled 单次关闭 active operation
TraceVersion 53 与 Reader/Visualizer/Exporter 兼容
核验并更新 StorageProxy 版本策略
```

验收：

```text
同 CasKey 并发传输可以唯一配对
所有终止路径都生成明确 result
新服务端 + v4/v5 客户端通过；新服务端 + v6 客户端通过
v6 客户端连接旧服务端时明确拒绝，不误解析
历史 Trace 仍可读取
```

### 提交 C2：请求上下文与进程因果关联

范围：

```text
requesterProcessId
requestReason
TraceRequestContext 沿 ProcessImpl / SessionClient / StorageClient 可选传播
Process → Fetch Perfetto Flow
后台请求保持 InvalidProcessId/Unknown
```

验收：

```text
能够从 cl.exe Slice 跳转到对应 Fetch Slice
无直接请求进程的 Fetch 使用 InvalidProcessId
```

### 提交 C3：客户端 materialization 与 OS PID

范围：

```text
target / materializedBytes / materialize result
osProcessId / executionType / ProcessExecutionInfo
客户端明确拥有的 receive/decompress/materialize ActiveTicks
```

验收：

```text
目标和 materializedBytes 来自客户端事实，不由服务端猜测
能够用 osProcessId 与 ETW 进程对齐
旧 Trace 和 C1/C2 Trace 仍可读取
```

### 提交 C4：Store 闭环

在 Fetch v6 混合版本验收后，核验 Store RPC 的现有请求/响应/断连路径，为 Store 定义独立的 operationId 与完成契约。只复用枚举、Exporter 和兼容原则，不复制 Fetch 线上布局的假设。

验收：同 CasKey 并发 Store 唯一配对，所有失败/断连路径关闭且只关闭一次，v4/v5/v6 混合版本矩阵通过。

### 后续候选：按实测数据决定

候选：

```text
PCH 特殊 Map 事件
精确阶段 Begin/End
原生 .pftrace 输出
多机 ClockSnapshot
```

没有数据证明需要时，不实施上述候选功能。

---

## 13. 验证方案

### 13.1 结构验证

```text
现有 TraceReader 接受的 TraceVersion 6～52 不因 Exporter 改动而回归
至少一个历史版本样本、v52 detailed、v52 non-detailed 可以读取
新 TraceVersion 可以读取
未知新版 Trace 能明确拒绝
损坏文件能在具体偏移或阶段报错
正常 Summary/EOF 不得因 `ReadFile` 返回 false 被误判为失败
```

### 13.2 计数一致性

```text
转换前后 Process 数量一致
detailed 模式的有效 Fetch/Store Slice 数量与向量中有效区间数一致
aggregate-only 模式的 Transfer Counter 与聚合字段一致
Counter 样本数量一致
所有可配对 Begin/End 正确配对
未配对事件有明确日志，不静默丢弃
non-detailed 聚合事件不得伪造成 Duration Slice
```

### 13.3 时间一致性

```text
Process 起止时间误差不超过一次 tick 换算误差
Fetch/Store 起止时间误差不超过一次 tick 换算误差
Counter 时间单调不回退
Duration 不为负数
相同输入不得引用导出时当前时钟，连续输出必须字节级一致
```

### 13.4 Perfetto 验证

```text
Perfetto UI 可以加载输出
Process、Fetch、Store 和 Counter 轨道可见
路线 B 的 Flow 数为 0；C-lite C2 可以从 Process 关联到 Fetch
大文件输出不需要构造完整内存 DOM
特殊字符、反斜杠和 Unicode 能正确转义
同一 operationId 的 Flow `id`/`scope` 稳定且唯一
```

其中 Process → Fetch Flow 只适用于 C-lite C2；路线 B 验收必须断言 Flow 数为 0。

### 13.5 Runtime 开销验证

C-lite 默认模式必须验证：

```text
不按 CAS chunk 产生事件
不追踪全部 Detoured API
operationId 分配无明显锁竞争
Trace 关闭时新增代码路径接近零开销
同一编译样本开启新字段前后 wall time 无统计显著回退
```

### 13.6 测试组织与夹具

路线 B 的 Exporter 单元测试通过 `UbaTest -> UbaTraceAnalysis` 直接构造最小 `TraceView`，覆盖：Process、Task、Work、detailed Transfer、aggregate-only Counter、未配对哨兵、无效 Duration、JSON 转义、`TrackRegistry`、稳定 ID 和确定性顺序。Task 必须只通过 `ProcessType_Task` 输出一次。这样不复制第二套二进制解析器。

计划新增的集成夹具目录为：

```text
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Test\Private\TraceFixtures
```

最小夹具集合：

```text
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Test\Private\TraceFixtures\v52-detailed.trace + E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Test\Private\TraceFixtures\v52-detailed.golden.json
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Test\Private\TraceFixtures\v52-light.trace + E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Test\Private\TraceFixtures\v52-light.golden.json
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Test\Private\TraceFixtures\historical-supported.trace + E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Test\Private\TraceFixtures\historical-supported.golden.json（选用一个仍由 TraceReader 支持的历史版本）
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Test\Private\TraceFixtures\truncated.trace
E:\UEWS\5.8.0\Engine\Source\Programs\UnrealBuildAccelerator\Test\Private\TraceFixtures\unsupported-version.trace
```

夹具必须去除业务绝对路径、机器名和敏感命令行；golden 只通过显式 `-UpdateGolden` 测试开关更新，CI 默认只比较。字节级 golden 比对限定为同一时钟频率环境；跨机 CI 可因 `double` 换算与 `u64` 截断出现亚微秒差异，必须使用 tick→微秒容差断言或固定频率夹具，不做跨机字节比对。路线 B 测试挂入现有 `UbaTest` 目标；Perfetto UI 人工加载是补充验收，不能替代 JSON 结构、计数、确定性和内存门槛自动测试。C-lite 额外覆盖 v4/v5 客户端→v6 服务端、v6→v6、v6→旧服务端拒绝、显式 End/断连竞争和同 CasKey 并发。

---

## 14. 风险与控制措施

| 风险 | 后果 | 控制措施 |
|---|---|---|
| `CasKey` 并发配对歧义 | 错误 Duration/Flow | 增加 `operationId` |
| 双写旧事件和新事件 | 重复数据、Trace 膨胀 | 按 TraceVersion 切换，避免长期双写 |
| 请求进程上下文跨层传播过深 | 大范围签名修改 | 使用可选 Trace Context，分独立提交 |
| JSON 体积过大 | Perfetto 加载慢 | 流式写出；有真实瓶颈后再支持 `.pftrace` |
| 记录过多映射细节 | 热路径开销和地址泄露 | 仅追踪 PCH 特殊路径，默认不记录地址 |
| ActiveTicks 被误画成顺序阶段 | 错误性能结论 | 仅作为参数/统计，不伪造成连续子 Slice |
| 多机时钟偏移 | Host/Agent 时间线错位 | 首版限制单 Trace；后续单独设计 ClockSnapshot |
| 新事件破坏旧 Visualizer | 无法查看新 Trace | Reader/Visualizer 与 schema 同提交验证 |
| Reader 迁移与行为修改混杂 | 回归难以定位 | B1 只机械迁移并独立编译验证 |
| Trace/Storage 版本轴混淆 | 错读二进制布局 | 分离常量、分别门控、混合版本矩阵测试 |
| End 与断连重复关闭 | 重复 Trace 或 active 表泄漏 | operationId 单次移除 + 客户端 completion guard |

---

## 15. 关键设计决策表

| 事项 | 决策 |
|---|---:|
| 实施离线 UBA → Perfetto 转换 | 是 |
| 首版等待 C-lite 完成后再交付 | 否 |
| 首版输出 Chrome Trace Event JSON | 是 |
| 首版直接输出原生 `.pftrace` | 否 |
| 复用现有 `TraceReader` | 是 |
| 实现第二套 UBA 二进制解析器 | 否 |
| 路线 B 使用独立 Console Target | 是，`LaunchModuleName = "UbaTraceConverter"` |
| 路线 B 迁移 TraceReader 到共享模块 | 是，B1 先机械迁移并单独验证 |
| 转换器依赖 UbaVisualizer | 否 |
| non-detailed 输入输出逐次 Fetch/Store Slice | 否，只输出聚合 Counter |
| 路线 B 输出 Process → Fetch Flow | 否 |
| 增加 `operationId` | 是 |
| 增加来源、目标、大小和结果 | 是，按 C1～C3 的事实来源分步增加 |
| 增加 `requesterProcessId` | 是，C2 独立提交 |
| 增加 `osProcessId` | 是，C3 独立提交 |
| 增加累计阶段 ActiveTicks | 是，C3 由客户端事实填充 |
| StorageNetworkVersion 6 | 是，C1-Fetch；服务端先部署 |
| TraceVersion 与 StorageNetworkVersion 绑定 | 否，两条独立版本轴 |
| Fetch 与 Store 同提交实施 | 否，Fetch 先验收，Store 独立 C4 |
| 追踪每个 CAS chunk | 否 |
| 追踪所有 Detoured API | 否 |
| 立即追踪全部 MapView | 否 |
| 按数据决定 PCH 特殊 Map 埋点 | 是 |
| Runtime 引入 Perfetto SDK | 暂不实施 |
| 增加 ETW Provider | 暂不实施 |
| 提前抽象通用 `ITraceSink` | 否 |
| 多机独立 Trace 精确合并 | 后续单独设计 |

---

## 16. 阿卡姆剃刀审视

最小闭环是：

```text
现有 UBA Trace
    → 复用 TraceReader
    → 流式导出 Perfetto 可加载 JSON
    → 补充 operationId、来源、目标、大小和结果
    → 再补请求进程与 OS PID
```

不应把最小闭环扩大为：

```text
全面重构 Trace 架构
引入实时 Perfetto SDK
引入 ETW Provider
追踪所有系统调用
一次性解决多机时钟同步
```

---

## 17. 已锁定的实施决策

第一轮与第二轮已经锁定：

1. 路线 B 先独立交付；它负责离线可视化现有事实，不承诺 Process → Fetch 因果关系。
2. 首版输出 Chrome Trace Event JSON，不引入 `.pftrace` 依赖。
3. 路线 B 使用独立 `UbaTraceConverter` Console Target；Reader 先机械迁移到 `UbaTraceAnalysis`，Visualizer、Converter 和 Test 共享该模块。
4. detailed 输入输出逐次 Fetch/Store Slice；non-detailed 输入只输出聚合 Counter，并明确记录降级。
5. C-lite 按 C1 Fetch v6 闭环、C2 请求进程传播、C3 客户端 materialization 与 OS PID、C4 Store 闭环分步提交。
6. 不追踪所有 MapView；只有路线 B/C-lite 数据证明需要时才增加 PCH 特殊映射事件。
7. 多机独立 Trace 精确合并不进入首版。
8. TraceVersion 与 StorageNetworkVersion 分离；Storage v6 采用新服务端先部署的 4～6 窗口策略。
9. 路线 B 读取成功依据类型化 `TraceReadResult`，禁止以 `ReadFile` 布尔值或 `finished` 单独推断。
10. 现有可回收 `u16 fetchId` 只属于 segment 传输层；C1 新增的 `u64 operationId` 独立存活至 FetchEnd/断连，二者不复用。
11. Exporter 内存门槛以解析后 RSS 为基线；只限制导出阶段的增量，同时单独记录 Reader/TraceView 总峰值。

---

## 18. 源码核验入口

```text
https://github.com/phoenixgou/UnrealEngine5.8/blob/release/Engine/Source/Programs/UnrealBuildAccelerator/Common/Public/UbaTrace.h
https://github.com/phoenixgou/UnrealEngine5.8/blob/release/Engine/Source/Programs/UnrealBuildAccelerator/Common/Private/UbaTrace.cpp
https://github.com/phoenixgou/UnrealEngine5.8/blob/release/Engine/Source/Programs/UnrealBuildAccelerator/Common/Public/UbaTraceChannel.h
https://github.com/phoenixgou/UnrealEngine5.8/blob/release/Engine/Source/Programs/UnrealBuildAccelerator/Visualizer/Private/UbaTraceReader.h
https://github.com/phoenixgou/UnrealEngine5.8/blob/release/Engine/Source/Programs/UnrealBuildAccelerator/Visualizer/Private/UbaTraceReader.cpp
https://github.com/phoenixgou/UnrealEngine5.8/blob/release/Engine/Source/Programs/UnrealBuildAccelerator/Common/Private/UbaStorageClient.cpp
https://github.com/phoenixgou/UnrealEngine5.8/blob/release/Engine/Source/Programs/UnrealBuildAccelerator/Common/Private/UbaStorageServer.cpp
```
