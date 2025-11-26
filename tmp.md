```mermaid
graph TD
    %% 定义样式
    classDef appLayer fill:#f9f,stroke:#333,stroke-width:2px,color:black;
    classDef simPlatform fill:#e1f5fe,stroke:#0277bd,stroke-width:2px,color:black;
    classDef kernelHW fill:#fff3e0,stroke:#e65100,stroke-width:2px,color:black;
    classDef physicalBase fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px,color:black;

    %% ---------------- 第1层：用户应用层 ----------------
    subgraph User_Application_Layer ["层1：目标负载与控制前端 (Target Workload & Control)"]
        App["目标应用 Target Application<br>(e.g., Redis, Llama.cpp, Benchmarks)"]:::appLayer
        ConfigCLI["配置前端 Topology Definer<br>(JSON/YAML Config)"]:::appLayer
    end

    %% ---------------- 第2层：仿真平台核心层 (用户态) ----------------
    subgraph Simulation_Platform_Core ["层2：Epoch执行驱动仿真引擎 (Epoch-Based Emulation Engine)"]
        direction TB
        
        subgraph Tracing_Injection ["2.1 追踪与注入模块 (Tracing & Injection)"]
            Injector["延迟注入器 Injector<br>(Epoch边界暂停/减速)"]
            TracerAgg["追踪数据聚合器 Tracer Aggregator"]
        end

        subgraph Timing_Model ["2.2 时序与协议模型 (Timing & Protocol Modeling)"]
            EpochScheduler["Epoch 调度器<br>(时间片管理)"]
            Analyzer["时序分析器 Timing Analyzer<br>(计算总延迟)"]
            
            subgraph CXL_Stack ["CXL 协议栈建模"]
                CXLio["CXL.io 模型<br>(PCIe事务/配置)"]
                CXLcache["CXL.cache 模型<br>(Snoop/一致性)"]
                CXLmem["CXL.mem 模型<br>(Flit打包/解包)"]
            end
            
            CongestionModel["动态拥塞模型 Congestion Model<br>(排队/带宽墙模拟)"]
        end

        subgraph Memory_Backend ["2.3 内存后端管理 (Memory Backend)"]
            MapperManager["内存映射管理器 Memory Mapper"]
            VirtualTopology["虚拟 CXL 拓扑视图<br>(Interleaving/Tiering策略)"]
        end
    end

    %% ---------------- 第3层：内核与硬件接口层 ----------------
    subgraph Kernel_HW_Interface ["层3：硬件遥测与内核接口 (Kernel & HW Telemetry Interface)"]
        eBPF_Probe["eBPF 内核探针<br>(追踪 sys_mmap/page_fault, 建立VA-PA映射)"]:::kernelHW
        PEBS_LBR["Intel PEBS/LBR 驱动接口<br>(精确采样LLC Miss/指令流)"]:::kernelHW
    end

    %% ---------------- 第4层：物理基础设施层 ----------------
    subgraph Physical_Infrastructure ["层4：物理主机资源 (Physical Host Resources)"]
        HostCPU["Host CPU (Cores & LLC)"]:::physicalBase
        LocalDRAM["本地 standard DRAM<br>(NUMA Node 0)"]:::physicalBase
        SimulatedCXL["模拟 CXL 内存池<br>(NUMA Node X, Y... via libnuma)"]:::physicalBase
    end

    %% ================= 连接关系 =================

    %% 配置流
    ConfigCLI -->|定义拓扑结构与策略| VirtualTopology
    ConfigCLI -->|配置Epoch参数| EpochScheduler

    %% 执行与追踪流 (关键路径)
    App <-->|读写操作Load/Store| HostCPU
    HostCPU -->|触发事件| PEBS_LBR
    HostCPU -->|内存分配调用| eBPF_Probe
    
    PEBS_LBR -->|采样数据流| TracerAgg
    eBPF_Probe -->|地址映射更新| TracerAgg
    
    %% 时序计算流 (Epoch周期)
    TracerAgg -->|汇总Epoch内事件| Analyzer
    EpochScheduler -->|触发计算信号| Analyzer
    VirtualTopology -->|提供拓扑路由信息| Analyzer
    
    Analyzer -->|请求协议开销| CXL_Stack
    Analyzer -->|请求带宽状态| CongestionModel
    CXL_Stack -->|返回协议延迟| Analyzer
    CongestionModel -->|返回排队延迟| Analyzer
    
    %% 延迟注入流 (反馈回路)
    Analyzer -->|输出总计算延迟| Injector
    Injector -->|在Epoch边界施加延迟| App

    %% 物理内存管理流
    App -- 实际数据存储 --> MapperManager
    MapperManager -->|libnuma绑定| SimulatedCXL
    MapperManager -->|区分流量| LocalDRAM
