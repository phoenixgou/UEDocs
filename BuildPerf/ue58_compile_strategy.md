# `-NoPCH -DisableUnity`：代码健康策略，而非日常提速策略

本文档面向工程团队内部。结论先行：`-NoPCH -DisableUnity` 不是用来"让编译更快"的，恰恰相反，它会让编译更慢。它的价值是**主动关闭编译期的"隐式可见性"，把平时被 PCH 和 Unity Build 掩盖的 include 缺陷暴露出来**，因此属于低频运行的代码健康检查，而不是日常构建配置。

---

## 0. 一句话职责划分

```mermaid
quadrantChart
    title 构建标志的职责定位
    x-axis 面向机器-构建吞吐 --> 面向源码-依赖正确性
    y-axis 低频专项运行 --> 高频常态运行
    quadrant-1 高频依赖校验-少见
    quadrant-2 常态提速
    quadrant-3 低频构建实验
    quadrant-4 低频代码体检
    "默认: PCH+SharedPCH+Unity": [0.20, 0.85]
    "-NoSharedPCH": [0.32, 0.60]
    "-NoPCH -DisableUnity": [0.82, 0.20]
```

- 右下角是 `-NoPCH -DisableUnity`：**面向源码正确性、低频运行**。
- 左侧是常态构建策略（含 `-NoSharedPCH`）：**面向构建吞吐/稳定性**。
- 二者目标不同，不能互相替代（详见第 8 节）。

---

## 1. 术语与事实依据（UE 5.8.0r / UnrealBuildTool）

| 标志 | 控制开关（默认值） | 源码注释含义 | 源码位置 |
| --- | --- | --- | --- |
| `-NoPCH` | `bUsePCHFiles`（默认 `true`） | "Whether PCH files should be used"，关闭**全部** PCH（含私有/显式 PCH） | `TargetRules.cs:2151` |
| `-NoSharedPCH` | `bUseSharedPCHs`（默认 `true`） | 仅关闭跨模块共享 PCH，其注释明确指出该特性 "significantly speeds up compile times" | `TargetRules.cs:2398` |
| `-DisableUnity` | `bUseUnityBuild`（默认 `true`） | "unify C++ code into larger files for faster compilation"，关闭把多个 `.cpp` 合并成大文件的 Unity 编译 | `TargetDescriptor.cs:138` |

事实要点：三者默认都为"开启提速"。`-NoPCH` 覆盖面比 `-NoSharedPCH` 更大——`-NoSharedPCH` 只关共享 PCH，`-NoPCH` 连模块自己的 PCH 也一并关闭。

> 说明：文中源码行号对应本机 Unreal Engine 5.8.0r 源码目录中的 `Engine/Source/Programs/UnrealBuildTool`。

---

## 2. PCH / SharedPCH / Unity Build 如何让源码"偶然通过"

三种机制的共同副作用：**让一个 `.cpp` 在没有写出对应 `#include` 的情况下，依然能看到它用到的符号。**

```mermaid
flowchart TD
    A["单个 .cpp 只写了少量 include"] --> B{"编译时符号从哪来"}
    B --> C["PCH: 预编译头把一大批公共头<br/>预先塞进每个翻译单元"]
    B --> D["SharedPCH: 跨模块共享的预编译头<br/>带来更多引擎公共符号"]
    B --> E["Unity Build: 多个 .cpp 被拼进同一大文件<br/>邻居 .cpp 的 include 顺带可见"]
    C --> F["符号'恰好'可见"]
    D --> F
    E --> F
    F --> G["编译通过<br/>但该 .cpp 并未显式声明依赖"]
    G --> H["'偶然通过' = 依赖靠环境, 不靠自身 include"]
```

关键区别：

- **PCH / SharedPCH** 让"公共大头文件"对所有翻译单元可见——你没写 `#include`，符号也在。
- **Unity Build** 让"同一批次里邻居 `.cpp`"的 include 顺带可见——依赖谁被合进同一大文件决定了你能看见什么，这个分组是构建工具决定的，不稳定。

---

## 3. "没有独立 include" 的成因

独立 include（self-contained include，每个文件都显式包含自己用到的一切）之所以缺失，是因为提速机制持续"补票"，让缺失的 include 从来不报错。

```mermaid
flowchart LR
    W["新写/新改一个 .cpp"] --> X["忘记 #include 某个头"]
    X --> Y{"是否报错"}
    Y -- "PCH/Unity 补上了可见性" --> Z["不报错 -> 开发者以为写对了"]
    Z --> R["缺失的 include 被长期保留"]
    R --> R2["下一个文件继续照抄这种写法"]
    R2 --> W
    Y -- "无任何提速机制" --> ERR["立即报错 -> 当场补 include"]
```

成因链是一个正反馈循环：**提速机制屏蔽了报错 → 缺失被保留 → 习惯被复制 → 缺失规模扩大**。缺陷不是一次写坏的，而是被长期"喂养"出来的。

---

## 4. 会导致哪些让工程团队反感的问题

| 问题 | 机制根因 | 团队感受 |
| --- | --- | --- |
| 编译时间过长 | 公共大头进了每个 TU；改一个公共头触发海量重编 | "改一行等半天" |
| 无关模块突然失败 | A 模块删掉一个传递 include，靠它"蹭"到符号的 B 模块崩了 | "我没碰它，它为什么挂" |
| 错误定位困难 | Unity 把多个 `.cpp` 拼成大文件，报错行号指向合成文件；符号来源不明 | "报错文件我根本没改过" |
| 本地/CI/XGE 行为不一致 | Unity 分组、PCH 命中、并行分发在不同环境下不同，暴露的缺失也不同 | "本地过了，CI 挂了" |
| 引擎升级/模块拆分风险 | 升级后引擎公共头内容变化，或模块被拆分打断了原有传递可见性 | "升个引擎，一片红" |

```mermaid
sequenceDiagram
    autonumber
    participant Dev as 开发者
    participant ModA as 模块A(被改)
    participant Header as 公共头/传递include
    participant ModB as 模块B(远处, 未改)
    participant Build as 构建(PCH+Unity)

    Note over ModB: B 用到 TypeX, 但从未 #include TypeX 的头
    Note over ModB,Build: 一直靠 A 的传递 include + Unity 分组"蹭"到 TypeX

    Dev->>ModA: 删掉一个"看起来没用"的 include
    ModA->>Header: 传递可见性被切断
    Build->>Build: 重新分组 Unity / 重建 PCH
    Build-->>ModB: TypeX 不再可见
    ModB--xBuild: 编译失败(报错点在 B)
    Build-->>Dev: 红在 B, 但根因在 A 的改动
    Note over Dev: 定位成本高: 改动处与失败处相距很远
```

这张时序图就是团队最反感的场景：**改动点与失败点解耦**，责任难归属，排查耗时。

---

## 5. `-NoPCH -DisableUnity` 如何暴露这些问题

原理：把第 2 节里的三条"隐式可见性"来源一次性关掉，每个 `.cpp` 只能看见它自己 `#include` 的内容。凡是"偶然通过"的文件，会在这一刻立即报错——错误点精确落在缺失 include 的那个文件。

```mermaid
flowchart TD
    S["运行 -NoPCH -DisableUnity"] --> S1["关闭全部 PCH 可见性"]
    S --> S2["关闭 Unity 合并, 每个 .cpp 独立编译"]
    S1 --> P["翻译单元只剩自身 include"]
    S2 --> P
    P --> Q{"该文件是否自洽"}
    Q -- "缺 include" --> FAIL["立即报错<br/>错误点=缺失文件本身"]
    Q -- "自洽" --> PASS["通过"]
    FAIL --> FIX["在该文件补上显式 include"]
```

代价与定位对比：

- **代价（预期内）**：编译更慢、失败更多。这是把长期负债一次性显性化，属于诊断成本，不是回归。
- **定位（关键收益）**：错误点 = 缺失 include 的文件本身，不再出现第 4 节那种"改 A 挂 B"的远距离故障。

---

## 6. 修复后的效果：影响范围收敛

补齐显式 include 后，依赖关系从"隐式、发散"变成"显式、收敛"。

```mermaid
graph TD
    subgraph BEFORE["修复前: 隐式依赖, 影响发散"]
        H1["公共头改动"] --> M1["模块1"]
        H1 --> M2["模块2"]
        H1 --> M3["模块3"]
        M1 --> M4["模块4(蹭传递include)"]
        M2 --> M4
        M3 --> M4
    end

    subgraph AFTER["修复后: 显式依赖, 影响收敛"]
        C1["改动某头"] --> D1["仅直接 #include 它的文件"]
        D1 --> D2["精确的增量重编集合"]
    end
```

修复带来的四个可度量效果：

| 维度 | 修复前 | 修复后 |
| --- | --- | --- |
| 依赖 | 隐式，靠环境补齐 | **显式**，每个文件自洽 |
| 模块边界 | 模糊，跨模块蹭符号 | **清晰**，越界即报错 |
| 增量编译 | 影响范围高估/漂移 | **影响范围可预测且更小** |
| 迁移/升级 | 引擎升级、模块拆分易一片红 | **更稳**，改动影响可提前评估 |

---

## 7. 策略定位（mindmap）

```mermaid
mindmap
  root((构建标志分工))
    常态-面向机器
      PCH 与 SharedPCH
        复用公共头 减少重复解析
      Unity Build
        合并 TU 减少编译次数
      NoSharedPCH 标志
        去掉共享PCH换取稳定性与隔离
        仍是构建性能与稳定性范畴
    低频-面向源码
      NoPCH 加 DisableUnity 标志
        关闭隐式可见性
        暴露缺失 include
        验证 self-contained header
        属于代码健康体检
```

---

## 8. 与 `-NoSharedPCH` 的职责区别

这是本文档最容易被混淆的一点，单独澄清。

```mermaid
flowchart LR
    subgraph PERF["构建性能/稳定性策略"]
        NSP["-NoSharedPCH"]
        NSP --> NSP1["只关'跨模块共享PCH'"]
        NSP1 --> NSP2["仍保留模块私有PCH"]
        NSP1 --> NSP3["仍保留 Unity Build"]
        NSP2 --> NSP4["动机: 规避共享PCH的<br/>污染/耦合/构建不稳定"]
        NSP3 --> NSP4
        NSP4 --> NSP5["可长期常态开启"]
    end

    subgraph HEALTH["代码健康检查策略"]
        NPU["-NoPCH -DisableUnity"]
        NPU --> NPU1["关'全部PCH' + 关 Unity"]
        NPU1 --> NPU2["每个 .cpp 完全独立"]
        NPU2 --> NPU3["动机: 暴露缺失include<br/>验证依赖显式"]
        NPU3 --> NPU4["低频/专项运行, 编译更慢"]
    end
```

| 维度 | `-NoSharedPCH` | `-NoPCH -DisableUnity` |
| --- | --- | --- |
| 归类 | 构建性能 / 稳定性策略 | 代码健康检查策略 |
| 关闭范围 | 仅共享 PCH（私有 PCH、Unity 仍在） | 全部 PCH + Unity |
| 主要动机 | 规避共享 PCH 的耦合与构建不稳定 | 暴露隐式依赖、验证 include 自洽 |
| 对编译速度 | 影响有限，可接受为常态 | 明显变慢，不适合常态 |
| 运行频率 | 可长期常态开启 | 低频、专项体检 |
| 成功信号 | 构建更稳定、可复现 | 报错集中在缺失 include 的文件 |

一句话：**`-NoSharedPCH` 解决"构建怎样更稳"，`-NoPCH -DisableUnity` 解决"源码是否真的自洽"。前者可以天天用，后者是定期体检。**

### 为什么 `-NoSharedPCH` 是较小代价的 XGE 策略

`-NoSharedPCH` 的关键价值不是"更严格"，而是**用最小源码扰动，保留大部分 PCH 收益，同时拿掉 XGE 下最重的跨机器共享包**。

```mermaid
flowchart LR
    A["默认: PCH + SharedPCH + XGE"] --> B["SharedPCH 很大<br/>跨模块复用范围广"]
    B --> C["XGE 需要分发/同步更大的 PCH 依赖"]
    C --> D["1Gbps 网络下可能变成传输瓶颈"]

    A --> E["使用 -NoSharedPCH"]
    E --> F["关闭 SharedPCH"]
    E --> G["保留模块内/普通 PCH"]
    G --> H["模块内头文件解析仍可复用"]
    F --> I["减少跨机器传输压力"]
    H --> J["源码适配成本远低于 -NoPCH"]
    I --> K["更容易吃到 XGE 并行收益"]
    J --> K
```

为什么它的源码适配成本通常较小：

- 模块内普通 PCH 仍在，很多模块内部的常用头仍能被局部复用。
- 健康的跨模块依赖本来就应该通过显式 `#include` 和 `.Build.cs` 依赖声明表达，而不是靠 SharedPCH "顺带看见"。
- 因此，关闭 SharedPCH 主要移除的是**跨模块的大范围隐式可见性**；它暴露的问题通常少于 `-NoPCH -DisableUnity`。

为什么它对 XGE 可能收益很大：

- SharedPCH 往往聚合 Engine / Editor 级公共头，体积可能显著大于普通模块 PCH。
- 在本轮 ProjectTitan 观测中，SharedPCH `.pch` 总量约 `13.4 GiB`，普通 PCH `.pch` 总量约 `5.8 GiB`；SharedPCH 不只是"多一点"，而是传输与内存压力的大头。
- 如果瓶颈在 1Gbps 上行、远端同步 PCH、或本地 pagefile/commit 压力，那么禁用 SharedPCH 可能比继续追求共享 PCH 命中更划算。

### UE 5.8.0r + ProjectTitan 的定量样本

样本范围：

- Engine：`D:\UE\5.8.0r`
- Project：`D:\UE\tutorial\ProjectTitan`
- Target：`TitanEditor Win64 Development` + `ShaderCompileWorker Win64 Development`
- 证据：默认 SharedPCH 构建导出的 `XGETasks.export.xml`，以及当时生成的 `.pch` 实物大小。

注意口径：

- `唯一 PCH 体积` 表示磁盘上不同 `.pch` 文件的合计大小。
- `cpp action 引用数` 表示有多少编译 action 通过 `/Yu` 使用这个 PCH。
- `cpp action 引用数 * PCH 大小` 不能直接等同为真实网络字节，因为 XGE 可能有远端缓存和复用；但它能说明哪些 PCH 是潜在同步压力的大头。

```mermaid
flowchart TD
    A["默认 SharedPCH 构建"] --> S["生成 SharedPCH<br/>11 个文件 / 13403 MiB"]
    A --> P["生成普通 PCH<br/>7 个文件 / 6035 MiB"]
    S --> T["ProjectTitan cpp<br/>130 个 action 全部使用<br/>1 个 2456 MiB SharedPCH"]
    S --> U["UE5.8.0 Engine cpp<br/>1662 个 action 使用<br/>10 个 SharedPCH / 10947 MiB"]
    P --> V["UE5.8.0 Engine cpp<br/>187 个 action 使用<br/>7 个普通 PCH / 6035 MiB"]
    B["-NoSharedPCH 构建"] --> B1["SharedPCH = 0"]
    B --> B2["普通 PCH 仍保留<br/>Core / CoreUObject / Engine / UnrealEd 等"]
```

生成端的 PCH 体积：

| 类型 | 生成 action | 唯一文件 | 唯一 PCH 体积 | 最大单文件 | XGE 远端生成 |
| --- | ---: | ---: | ---: | ---: | --- |
| SharedPCH | 11 | 11 | `13403.06 MiB` (`13.09 GiB`) | `2458.19 MiB` | `AllowRemote=False` |
| 普通 PCH | 7 | 7 | `6035.06 MiB` (`5.89 GiB`) | `2071.44 MiB` | `AllowRemote=False` |

解释：

- SharedPCH 的唯一体积约为普通 PCH 的 `2.22x`。
- PCH / SharedPCH 生成 action 都是本地 `AllowRemote=False`，远端 cpp 编译若依赖它们，就需要面对同步/缓存/传输问题。
- 若只统计 `Engine\Intermediate\Build` 下的普通 PCH，不含 Engine plugin PCH，则普通 PCH 为 `5804.37 MiB`；这就是前面观察到的约 `5.8 GiB` 口径。

cpp action 对应的 PCH：

| 源码归属 | 使用 SharedPCH 的 cpp action | 对应 SharedPCH footprint | 使用普通 PCH 的 cpp action | 对应普通 PCH footprint | 结论 |
| --- | ---: | ---: | ---: | ---: | --- |
| ProjectTitan 自己的 cpp | `130` | `1` 个 / `2455.69 MiB` | `0` | `0` | Titan 自己的远端 cpp 编译集中依赖一个 2.4 GiB 级 SharedPCH |
| UE5.8.0 Engine cpp | `1662` | `10` 个 / `10947.37 MiB` | `187` | `7` 个 / `6035.06 MiB` | UE Editor/Engine 编译同时用 SharedPCH 和普通 PCH，但 SharedPCH 覆盖更多 cpp action |

这组数据支持的判断：

- 对 ProjectTitan 自己的 cpp 来说，默认 SharedPCH 模式下不是"很多小 PCH 分散使用"，而是 `130` 个 cpp action 集中依赖同一个 `2455.69 MiB` 的 `SharedPCH.UnrealEd.Project.ValApi.ValExpApi.Cpp20.h.pch`。
- 对 UE5.8.0 Engine cpp 来说，SharedPCH 不仅总量更大，引用它的 cpp action 也远多于普通 PCH（`1662` vs `187`）。
- 因此在 XGE + 1Gbps 网络下，`-NoSharedPCH` 的价值很具体：砍掉最大的跨机器共享 PCH 依赖，同时保留普通 PCH 机制，不等价于 `-NoPCH`。
- B 组 `-NoSharedPCH` 取证窗口中，`SharedPCHCount=0`、`PCHCount=6`，说明 SharedPCH 被移除，但普通 PCH 仍然存在。

### Session 019f22ba 的最终 A/B 证据

这轮数据更接近日常开发体验：A 组默认 SharedPCH cold build 先失败，随后不 clean 增量续编成功；B 组 `-NoSharedPCH` cold build 一次成功。

```mermaid
flowchart LR
    A["A: 默认 SharedPCH"] --> A1["cold 首段失败<br/>1611.74s / 165945.59 MiB"]
    A1 --> A2["不 clean 增量续编成功<br/>129.68s / 649.13 MiB"]
    A2 --> A3["累计完成口径<br/>1741.42s"]

    B["B: -NoSharedPCH"] --> B1["cold 一次成功<br/>509.84s / 31938.51 MiB"]
    B1 --> B2["SharedPCH action = 0"]

    A3 --> C["A / B 耗时 = 3.416x"]
    B2 --> D["A / B 上传量 = 5.216x"]
```

| 指标 | A 默认 SharedPCH | B `-NoSharedPCH` | 结论 |
| --- | ---: | ---: | --- |
| 完成口径 | cold 失败 + 增量成功 | cold 一次成功 | B 更贴近日常开发想要的"一次过" |
| 总耗时 | `1741.42s` | `509.84s` | A 是 B 的 `3.416x` |
| 上传量 | `166594.72 MiB` | `31938.51 MiB` | A 是 B 的 `5.216x` |
| `>=900 Mbps` 上行饱和 | 首段约 `885s` | 显著下降 | A 的中心分发压力更高 |
| SharedPCH action | 存在 | `0` | B 明确移除了 SharedPCH 构建/依赖 |

最大嫌疑项是 `SharedPCH.UnrealEd.Cpp20.h.pch`：

| 指标 | 数值 |
| --- | ---: |
| 文件大小 | `2455.5 MiB` |
| 使用 task | `501` |
| helper fanout | `128` |
| 中心分发理论下限 | `2668.34s` |
| break-even helper | `3.5` |
| A/B matched task 中位耗时差 | A 慢 `30.8s` |
| A/B matched task P95 耗时差 | A 慢 `719.95s` |

```mermaid
flowchart TD
    S["2455.5 MiB SharedPCH.UnrealEd"] --> H["128 个 helper 需要首次取用"]
    H --> U["本机中心上传<br/>1Gbps 级上行被长时间占满"]
    U --> T["使用该 SharedPCH 的 task 完成变慢"]
    T --> F["A cold 首段失败<br/>需要增量续编"]

    N["-NoSharedPCH"] --> Z["SharedPCH action = 0"]
    Z --> P["保留普通 PCH"]
    P --> R["上传量下降到 31938.51 MiB<br/>cold build 一次成功"]
```

这组数据能支撑的开发期判断：

| 开发期问题 | 默认 SharedPCH 的表现 | `-NoSharedPCH` 的价值 |
| --- | --- | --- |
| 冷启动稳定性 | 大 SharedPCH 触发上行饱和与 PCH/内存压力，A 首段失败 | B 一次 cold 成功 |
| 等待时间 | A 累计 `1741.42s` | B `509.84s` |
| 网络占用 | A 上传约 `162.69 GiB` | B 上传约 `31.19 GiB` |
| XGE 并行收益 | 大文件中心分发抵消 helper 并行 | 减少中心分发包，更容易吃到 helper 算力 |
| 源码扰动 | 不解决 include 健康，只追求 SharedPCH 命中 | 保留普通 PCH，不等价于 `-NoPCH` 的强体检 |

口径限制：

- A 不是单次 cold 成功样本，而是"cold 失败 + 增量续编成功"的实际完成口径。
- 这不能直接证明 IncrediBuild 对每个 helper 的真实上传次数。
- 但 `SharedPCH` 体积、helper fanout、上行饱和、A/B 上传量、A/B task 耗时差和 B 组 `SharedPCH action = 0` 已经形成强证据链。

第一性原理拆解：

```mermaid
flowchart TD
    F["事实层<br/>日志/CSV/文件大小"] --> M["模型层<br/>helper 首次取用大 PCH"]
    M --> D["决策层<br/>开发期默认倾向 -NoSharedPCH"]

    F1["A 上传 166594.72 MiB<br/>B 上传 31938.51 MiB"] --> F
    F2["A 累计 1741.42s<br/>B cold 509.84s"] --> F
    F3["B SharedPCH action = 0"] --> F
    F4["SharedPCH.UnrealEd = 2455.5 MiB<br/>helper fanout = 128"] --> F

    M1["不是每个 task 都必然重传<br/>更合理是 helper 冷缓存首次取用"] --> M
    M2["1Gbps 上行 + 大文件 fanout<br/>足以解释长时间饱和"] --> M

    D1["平时开发优先要一次成功、少等、少占网"] --> D
    D2["-NoSharedPCH 保留普通 PCH<br/>成本远小于 -NoPCH"] --> D
```

| 层级 | 可以强证明的内容 | 不能过度声称的内容 | 文档结论 |
| --- | --- | --- | --- |
| 事实 | A/B 耗时、上传量、B 无 SharedPCH action、PCH 文件体积、helper fanout、matched task 差异 | 每个 helper 实际传输了几次 `.pch` | 事实足以证明默认 SharedPCH 在本环境下很贵 |
| 模型 | helper 首次取用大 SharedPCH 会造成中心上传压力，且数量级能解释 1Gbps 饱和 | IncrediBuild 内部同步算法的精确行为 | 模型是解释，不是伪装成日志事实 |
| 决策 | B 一次 cold 成功，耗时/上传都显著低于 A 累计口径 | 所有机器、所有网络、所有项目都必然更快 | 对当前 XGE + 1Gbps 开发环境，`-NoSharedPCH` 适合作为常态候选 |

主程可能会挑战的点：

| 挑战 | 回答 |
| --- | --- |
| "A 不是单次 cold 成功样本，能比较吗？" | 可以，但口径是"实际开发完成一次构建"：A 失败后续编才完成，B cold 一次完成；不把它包装成严格统计学的单次 cold 成功 A/B。 |
| "你没有 IncrediBuild per-helper 文件传输日志。" | 对，所以文档只说"强证据链"和"中心分发模型"，不说"已精确证明每个 helper 上传次数"。 |
| "`-NoSharedPCH` 会不会只是这次 farm 状态好？" | 有 farm/cache 噪声风险，但 A/B 差距是 `3.416x` 耗时与 `5.216x` 上传量，且 B 的 SharedPCH action 为 `0`，不是小波动量级。 |
| "为什么不用 `-NoPCH`？" | `-NoSharedPCH` 保留普通 PCH，目标是构建稳定/吞吐；`-NoPCH -DisableUnity` 是低频代码健康检查，成本和目的不同。 |
| "本地非 XGE 也该关 SharedPCH 吗？" | 文档结论限定在当前 XGE + 1Gbps + 大 SharedPCH 环境；纯本地构建可能仍受益于 SharedPCH。 |

---

## 9. 使用建议

| 场景 | 建议 | 原因 |
| --- | --- | --- |
| 日常 Dev Editor + XGE 构建 | 默认候选使用 `-NoSharedPCH` | 当前 ProjectTitan + UE5.8.0r + 1Gbps 环境中，它显著降低上传量与完成时间，并避免 SharedPCH 首段失败 |
| 本地单机或高速内网构建 | 保留复测空间 | SharedPCH 在无中心上传瓶颈时仍可能带来本地编译收益 |
| include/IWYU 健康检查 | 低频运行 `-NoPCH -DisableUnity` | 它是代码健康体检，不是日常提速方案 |
| `-NoPCH -DisableUnity` 报错 | 就地补齐显式 `#include` / `.Build.cs` 依赖 | 不要靠回退提速标志"消除"真实依赖问题 |
| 评估是否回开 SharedPCH | 只在网络、helper、pagefile、PCH 体积明显变化后重测 | 当前证据链支持 `-NoSharedPCH`，但结论绑定当前硬件/网络/项目规模 |
