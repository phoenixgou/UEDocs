# Incredibuild (IB) 工作原理与机制图解

Incredibuild 的核心思想是：**把你公司/团队里所有闲置的电脑 CPU 核心，虚拟化成一台超级计算机来并行编译你的代码。**

---

## 1. 整体架构概览

Incredibuild 由三个核心角色组成：**Coordinator（协调器）**、**Initiator（发起者）** 和 **Helper（帮助者）**。

```mermaid
graph TB
    subgraph Network["公司内网 / 构建集群"]
        direction TB
        
        Coord["🖥️ Coordinator<br/>协调器 (1台)"]
        
        subgraph Initiator["发起者 (Initiator)"]
            I1["💻 开发者机器 A<br/>触发编译的那台机器<br/>运行 MSBuild / UBT"]
        end
        
        subgraph Helpers["帮助者池 (Helper Pool)"]
            H1["🖥️ Helper B<br/>8 核 / 16 线程"]
            H2["🖥️ Helper C<br/>12 核 / 24 线程"]
            H3["🖥️ Helper D<br/>16 核 / 32 线程"]
            H4["🖥️ Helper E<br/>4 核 / 8 线程"]
            H5["🖥️ ...<br/>更多空闲机器"]
        end
    end

    I1 -- "1. 我有 2000 个编译任务<br/>请分配帮手" --> Coord
    Coord -- "2. 这些机器目前有空闲核心<br/>分配给你" --> I1
    I1 -. "3. 分发编译任务 (Actions)" .-> H1
    I1 -. "3. 分发编译任务 (Actions)" .-> H2
    I1 -. "3. 分发编译任务 (Actions)" .-> H3
    I1 -. "3. 分发编译任务 (Actions)" .-> H4
    I1 -. "3. 分发编译任务 (Actions)" .-> H5
    H1 -. "4. 返回 .obj 文件" .-> I1
    H2 -. "4. 返回 .obj 文件" .-> I1
    H3 -. "4. 返回 .obj 文件" .-> I1
    H4 -. "4. 返回 .obj 文件" .-> I1
    H5 -. "4. 返回 .obj 文件" .-> I1
    I1 -- "5. 最终链接 (Link)" --> I1

    classDef coord fill:#005A9E,stroke:#0078D4,stroke-width:2px,color:#FFF;
    classDef init fill:#107C10,stroke:#00FF00,stroke-width:2px,color:#FFF;
    classDef helper fill:#1E1E1E,stroke:#666,stroke-width:2px,color:#FFF;
    
    class Coord coord;
    class I1 init;
    class H1,H2,H3,H4,H5 helper;
```

> [!IMPORTANT]
> **关键限制**：链接 (Link) 步骤**不能**被分布式执行，必须在 Initiator 本地完成。因此 UE 大型项目的链接阶段仍然可能很慢（尤其是启用 LTO 的 Shipping 构建）。

---

## 2. 单个编译任务的分发与执行细节

下面这张序列图展示了一个 `.cpp` 文件是如何被发送到远程 Helper 机器上编译，然后将结果返回的。

```mermaid
sequenceDiagram
    participant UBT as UnrealBuildTool<br/>(Initiator 本地)
    participant Agent as IB Agent<br/>(Initiator 本地)
    participant Net as 网络传输
    participant Remote as IB Agent<br/>(Helper 远程)
    participant CL as cl.exe<br/>(Helper 远程)

    Note over UBT: UBT 生成编译 Action 列表<br/>例如：cl.exe /c MyActor.cpp /Fo MyActor.obj

    UBT->>Agent: 提交 Action：编译 MyActor.cpp
    
    Note over Agent: IB 的核心魔法 —— 文件虚拟化

    Agent->>Agent: 拦截 Action 的文件依赖<br/>分析 MyActor.cpp 的 #include 依赖树

    Agent->>Net: 打包并传输必要文件：<br/>• MyActor.cpp<br/>• MyActor.h<br/>• 所有递归 #include 的头文件<br/>• PCH 文件 (如果有)<br/>• 编译器参数

    Net->>Remote: 接收文件包

    Remote->>Remote: 在本地沙箱中<br/>虚拟化还原目录结构<br/>(文件路径与 Initiator 完全一致)

    Remote->>CL: 启动 cl.exe<br/>使用与 Initiator 完全相同的参数

    CL->>CL: 编译 MyActor.cpp

    alt 编译成功
        CL->>Remote: 生成 MyActor.obj
        Remote->>Net: 回传 MyActor.obj
        Net->>Agent: 接收 .obj
        Agent->>UBT: Action 完成 ✅
    else 编译失败
        CL->>Remote: 输出错误信息
        Remote->>Net: 回传错误日志
        Net->>Agent: 接收错误
        Agent->>UBT: Action 失败 ❌ + 错误日志
    end
```

---

## 3. 文件虚拟化机制 (核心技术)

Incredibuild 最核心的技术是 **文件系统虚拟化**，它让远程 Helper 机器"以为"自己就是 Initiator。

```mermaid
graph LR
    subgraph Initiator["Initiator 机器 (实际文件系统)"]
        direction TB
        IA["E:\UEWS\5.7.4\Engine\Source\Runtime\Engine\Private\Actor.cpp"]
        IB_file["E:\UEWS\5.7.4\Engine\Source\Runtime\Engine\Public\Actor.h"]
        IC["E:\UEWS\5.7.4\Engine\Source\Runtime\Core\Public\UObject\Object.h"]
        ID["C:\Program Files\MSVC\include\vector"]
    end

    subgraph Transfer["IB 传输层"]
        direction TB
        T1["📦 仅传输本次编译<br/>实际需要的文件"]
        T2["🔄 使用增量缓存<br/>相同文件不重复传"]
    end

    subgraph Helper["Helper 机器 (虚拟化文件系统)"]
        direction TB
        HA["虚拟路径 E:\UEWS\...\Actor.cpp ✅"]
        HB["虚拟路径 E:\UEWS\...\Actor.h ✅"]
        HC["虚拟路径 E:\UEWS\...\Object.h ✅"]
        HD["虚拟路径 C:\Program Files\...\vector ✅"]
        HE["cl.exe 看到的路径与 Initiator 完全一致<br/>编译参数无需任何修改"]
    end

    Initiator --> Transfer --> Helper

    classDef real fill:#107C10,stroke:#00FF00,stroke-width:2px,color:#FFF;
    classDef virtual fill:#005A9E,stroke:#0078D4,stroke-width:2px,color:#FFF;
    classDef transfer fill:#A4262C,stroke:#FF6666,stroke-width:2px,color:#FFF;
    
    class IA,IB_file,IC,ID real;
    class HA,HB,HC,HD,HE virtual;
    class T1,T2 transfer;
```

> [!NOTE]
> **为什么不需要在 Helper 上安装 UE 源码？** 因为 IB 的内核驱动会拦截 `cl.exe` 的所有文件 I/O 系统调用（如 `CreateFile`、`ReadFile`）。当 `cl.exe` 试图读取一个文件时，IB Agent 会先检查本地虚拟缓存，如果没有就实时从 Initiator 拉取。对 `cl.exe` 来说，它完全感知不到自己在远程机器上运行。

---

## 4. 并行度对比：本地 vs Incredibuild

```mermaid
gantt
    title 编译 2000 个 .cpp 文件 (示意)
    dateFormat X
    axisFormat %s

    section 本地编译 (16核)
    cpp_1 ~ cpp_16  并行     :a1, 0, 10
    cpp_17 ~ cpp_32 并行     :a2, 10, 20
    cpp_33 ~ cpp_48 并行     :a3, 20, 30
    ...  漫长等待 ...        :a4, 30, 120
    最终链接 Link            :a5, 120, 130

    section IB 分布式 (16核 x 10台 = 160核)
    cpp_1 ~ cpp_160 并行分发  :b1, 0, 10
    cpp_161 ~ cpp_320 并行    :b2, 10, 20
    剩余全部完成              :b3, 20, 25
    最终链接 Link             :b4, 25, 35
```

> [!TIP]
> **典型加速比**：对于 UE5 全量编译（约 5000+ 编译单元），一台 16 核机器可能需要 **40-60 分钟**。接入 10 台 Helper 后，编译阶段可以压缩到 **5-8 分钟**，但链接阶段时间不变。

---

## 5. 与 UE Build System (UBT) 的集成方式

```mermaid
graph TD
    A["开发者执行编译命令<br/>Build.bat UnrealEditor Win64 Development"] --> B["UnrealBuildTool (UBT) 启动"]
    
    B --> C["UBT 扫描模块依赖<br/>生成 Action Graph"]
    C --> D["Action Graph 包含所有编译 / 链接步骤<br/>例如 2000 个 Compile + 50 个 Link"]
    
    D --> E{UBT 检测可用的<br/>构建执行器}
    
    E -- "检测到 XGE/IB" --> F["使用 XGE Executor<br/>(Incredibuild 的执行器接口)"]
    E -- "未检测到 / 加了 -NoXGE" --> G["回退到本地并行执行器<br/>(ParallelExecutor)"]
    
    F --> H["UBT 将所有 Compile Action<br/>写入 .xml 任务描述文件"]
    H --> I["调用 xgConsole.exe<br/>提交 .xml 给 IB"]
    I --> J["IB Coordinator 分配 Helpers"]
    J --> K["Helpers 并行编译"]
    K --> L["所有 .obj 回传到 Initiator"]
    L --> M["UBT 在本地执行 Link Action"]
    
    G --> N["本地 cl.exe 用所有核心并行编译"]
    N --> M
    
    M --> O["生成最终二进制<br/>UnrealEditor.exe / .dll"]

    classDef ib fill:#005A9E,stroke:#0078D4,stroke-width:2px,color:#FFF;
    classDef ubt fill:#107C10,stroke:#00FF00,stroke-width:2px,color:#FFF;
    classDef local fill:#1E1E1E,stroke:#666,stroke-width:2px,color:#FFF;
    
    class F,H,I,J,K,L ib;
    class A,B,C,D,E ubt;
    class G,N local;
```

> [!IMPORTANT]
> 在你的 `AGENTS.md` 中，编译命令使用了 `-NoUBA -NoFASTBuild -NoSNDBS` 但**没有** `-NoXGE`，这正是为了保留 Incredibuild 作为分布式执行器。UBT 日志中出现 `Distributing ... actions to XGE` 就说明 IB 已经在工作了。

---

## 6. IB 缓存与增量编译优化

```mermaid
graph LR
    subgraph FirstBuild["第一次编译 MyModule"]
        F1["MyActor.cpp<br/>+ 500 个头文件"] -- "全量传输" --> F2["Helper 编译<br/>生成 MyActor.obj"]
        F2 -- "缓存文件哈希" --> F3["IB 本地缓存<br/>(文件指纹表)"]
    end

    subgraph SecondBuild["第二次编译 MyModule (改了 1 个头文件)"]
        S1["MyActor.cpp<br/>+ 500 个头文件<br/>(1个有变化)"] -- "比对缓存指纹" --> S2{"哪些文件变了?"}
        S2 -- "仅 1 个文件变化" --> S3["只传输 1 个变化的头文件<br/>(增量传输)"]
        S3 --> S4["Helper 使用缓存 + 增量<br/>重新编译"]
    end

    FirstBuild -.-> SecondBuild

    classDef cache fill:#6B3FA0,stroke:#9B59B6,stroke-width:2px,color:#FFF;
    class F3,S2 cache;
```

---

## 总结：IB 的关键机制一览

| 机制 | 说明 |
|------|------|
| **分布式编译** | 将独立的编译单元 (.cpp → .obj) 分发到网络中的空闲机器并行执行 |
| **文件虚拟化** | 通过内核驱动拦截文件 I/O，让远程 cl.exe 以为自己在 Initiator 上运行 |
| **Coordinator 调度** | 中央协调器管理所有 Agent 的可用核心数，按需分配给 Initiator |
| **增量缓存** | 基于文件哈希的缓存机制，避免重复传输未变化的文件 |
| **透明集成** | 对 UBT/MSBuild 等构建系统透明，不需要修改项目配置 |
| **仅编译可分布** | 链接 (Link) 阶段**不能**分布式执行，仍在本地完成 |
