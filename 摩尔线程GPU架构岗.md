在 GPU 的计算单元（SM / Compute Engine）内部，这四个模块构成了 **从指令调度到数据就绪、再到执行单元发射的核心数据通路**。它们协同解决了超高并发下的**乱序准备、顺序发射、多操作数竞争与极高计算吞吐**的问题。

---

### 1. Warp Scheduler（线程束调度器）

* **核心功能**：每个周期从驻留（Resident）的几十个 Warp 中，挑选出状态就绪（Ready）的一个或多个 Warp，提取指令发射到下级流水线。
* **Warp 状态管理**：
* **Ready**：无数据/结构冒险（Scoreboard 判定通过），操作数/执行单元空闲。
* **Stalled**：由于长延迟操作（如 L1/L2 Cache Miss、Texture 访存、Barrier 同步）未完成而阻塞。
* **Eligible**：满足发射基本条件，参与仲裁。


* **主流调度策略**：
* **LRR (Loose Round Robin)**：轮询调度，保证线程束间公平推进，适合均衡型通用计算。
* **GTO (Greedy-Then-Oldest)**：优先一直调度同一个 Warp 直到其 Stall，随后调度等待时间最久（Oldest）的 Warp。优势在于能更快释放先达指令占用的局部资源，并提升 Cache 局部性。


* **CModel 建模要点**：
* 数据结构：维护一个 `std::vector<WarpState>`，包含 `warp_id`、`pc`、`active_mask`、`stall_reason`。
* 调度逻辑：每个周期遍历筛选 `status == READY` 的候选池，根据选择算法返回选中的 `warp_id`，若均阻塞则记录 `Stall_Cycle` 统计指标。



---

### 2. Scoreboard（记分板）

* **核心功能**：追踪正在执行的长/短延迟指令的目标寄存器，**检测并消除 RAW（Read-After-Write）和 WAW 数据冒险**。
* **硬件机制**：
* **位图/表项结构**：每个 Warp 维护一组寄存器占用状态表。当指令发射且目标寄存器为 $R_d$ 时，Scoreboard 将 $R_d$ 标记为 Pending（置 1）。
* **RAW 检查**：Warp Scheduler 尝试发射新指令时，提取源操作数 $R_s, R_t$，查询 Scoreboard。若任意源寄存器为 Pending，指令被阻塞。
* **清除（Clear/Writeback）**：当内存加载返回或长周期计算完成并写回寄存器时，Scoreboard 对应位清零，唤醒等待该寄存器的 Warp。


* **CModel 建模要点**：
* 数据结构：`std::vector<std::bitset<NUM_PHYSICAL_REGS>> reg_scoreboard;`（按 Warp 划分维度）。
* 接口设计：
* `bool check_collision(warp_id, src_regs[])`：调度前检查。
* `void set_pending(warp_id, dst_reg)`：指令发射时占用。
* `void clear_pending(warp_id, dst_reg)`：写回阶段释放。





---

### 3. Operand Collector（操作数收集器）

* **为什么需要**：GPU 寄存器堆（SRAM）容量巨大，为节省面积和功耗，通常做成**单端口（1 Read/Write Port per cycle）的多 Bank 结构**。一条 SIMT 指令（如 FMA $R_d = R_a \times R_b + R_c$）需要 3 个源操作数，若 3 个操作数落在同一个 SRAM Bank，会产生 **Bank Conflict**。
* **核心机制**：
* **Collector Units (CU)**：包含多个小缓冲槽（Buffer / Register File Cache）。
* **多周期收集**：指令发射后不直接进入执行单元，而是先分配一个 CU。
* **Crossbar 交叉开关与仲裁**：通过 Crossbar 连接 Bank 与 CU。每个周期 Arbitrator 为每个 Bank 挑选一个读请求，无冲突的操作数立即读取并存入对应 CU。
* **发射就绪**：当一条指令的 3 个操作数（可能跨越 1~3 个时钟周期）全部收集齐后，CU 才将完整数据包打入 ALU/Tensor 流水线。


* **CModel 建模要点**：
* 设计一个 `CollectorUnit` 类，包含 `src_ready_flags[3]` 和操作数数据缓冲。
* 每周期遍历所有未就绪 CU，提取待读 `bank_id = reg_id % NUM_BANKS`，执行 Bank 仲裁；胜出者推进 `ready_flags`，全部置位后推入 `Execution_Ready_Queue`。



---

### 4. ALU / Tensor 流水线机制

* **ALU 流水线（SIMD Vector Execution）**：
* **结构划分**：通常按数据类型分为 SP（Single Precision / FP32）、DP（FP64）、INT32 及 SFU（Special Function Unit，用于三角函数、开方等）。
* **流水级推进**：典型的 4~8 级深度流水线（Fetch $\to$ Decode $\to$ Read $\to$ Execute1..N $\to$ Writeback）。
* **Active Mask 过滤**：ALU 内部以 32-lane SIMD 物理阵列并行执行，只有对应 `active_mask[lane] == 1` 的计算结果才会在写回级写入寄存器。


* **Tensor Core / 矩阵加速流水线**：
* **执行逻辑**：面向 MMA（Matrix Multiply-Accumulate，$D = A \times B + C$）操作，以块（Tile）为粒度执行。
* **脉动阵列（Systolic Array）vs 乘加树（Tree-MAC）**：
* **Tree-MAC**：单/多周期内将整个 Tile 的向量点乘聚合，适合中低维度快速计算。
* **Systolic Flow**：数据在 Processing Elements (PE) 之间按拍（Cycle）向右/向下流动，复用内部寄存器权重，极大降低寄存器堆读写带宽压力。


* **流水线深度与吞吐**：Tensor 指令通常具有较长延迟（数十个 Cycle），依靠深流水线维持每个周期发射一个 MMA Tile 的高吞吐。


* **CModel 建模要点**：
* 使用双向队列或固定长度数组 `std::deque<PipelineStage>` 模拟每一级流水寄存器。
* 每个 `tick()` 遍历流水线，从后往前（Stage N 到 Stage 0）依次执行 `compute()` 并向下传递状态，防止单周期内数据直通引发逻辑时序错误。



---

### 四者协同数据流示意

```
[ Warp Scheduler ] ──(根据 Scoreboard 挑选 Ready Warp)
        │
        ▼
[ Instruction Decode & Allocation ] ──(设置 Scoreboard Pending 位)
        │
        ▼
[ Operand Collector ] ◄──(分拍仲裁读取 Multi-Bank Register File)
        │ (所有操作数就绪)
        ▼
[ Execution Pipelines: ALU / SFU / Tensor Core ]
        │ (流水线计算完成)
        ▼
[ Writeback Stage ] ──► (写回 Register File & 清除 Scoreboard Pending 位)

```

针对现代 **GPGPU 体系结构 / 芯片系统架构设计（CModel / RTL / 性能建模）** 的面试需求，以下将原文拆解重构为 **高频考点、微架构机制、设计权衡（Trade-offs）与可直接复述的面试应答框架**。

---

## 核心主线：一句话定义 SIMT 的本质

> **“SIMT 不是替代 SIMD，而是对底层宽 SIMD 硬件的线程级虚拟化抽象。”**
> 程序员以标量（SPMD）思维写单线程逻辑，硬件（Warp Scheduler + Coalescing + Active Mask）在运行时动态聚合、调度并发射到宽物理向量执行通路。其本质是 **用极高的硬件复杂度（面积、控制逻辑）换取软件开发的高生产力与低编程门槛**。

---

## 模块一：SIMT vs. SIMD / MIMD 核心对比（微架构视角）

在架构面试中，面试官常通过对比考察你对硬件控制流与数据通路的理解：

| 维度 | CPU SIMD (AVX/NEON) | GPU SIMT (CUDA/MUSA) | GPU 跨 SM 级 (MIMD) |
| --- | --- | --- | --- |
| **编程模型** | **向量编程**（显式控制向量宽、手动对齐、手写掩码） | **标量编程（SPMD）**（编写单线程逻辑，隐式多线程） | **多核多任务**（多 Block/Kernel 独立） |
| **PC 与控制流** | 1 个 Core 对应 1 个 PC，指令级操作宽数据 | **物理聚合**（1 个 Warp 共享或每线程独立 PC 动态聚合） | 独立控制流，各 SM 异步执行 |
| **寄存器堆** | 少量、固定物理宽度的向量寄存器（如 32 个 512-bit） | 海量 32-bit 标量寄存器堆（SRAM Banks），动态分配 | 独立 SM 拥有独立的 Register File |
| **访存行为** | 显式向量 Load/Store（未对齐或 Gather 会触发多次周期） | **动态合并**（Coalescing 单元将 32 个地址折叠为最小事务） | 独立 L1/L2 缓存与显存通道 |
| **延迟隐藏机制** | 依赖深流水线、乱序执行（OoO）、分支预测、大容量 Cache | **极高并发多线程（Fine-grained MT）**，硬件零开销切换 | 任务级/网格级调度器（Grid Management） |

---

## 模块二：四大微架构核心机制（系统设计要点）

### 1. 指令发射与延迟隐藏（Warp Scheduler）

* **硬件特点**：顺序发射、无分支预测、无推测执行、无寄存器重命名。
* **延迟隐藏机制**：每个 SM 驻留多个 Warp（通常 48~64 个，即 1536~2048 个活跃线程）。每个周期调度器维护一个就绪池（Ready Pool），当当前 Warp 发生长延迟访存（Memory Stall）或同步阻塞（Sync Stall）时，**单周期零开销**切换到另一个 Ready Warp 发射，保持 ALU 满载。

### 2. 分支发散与收敛（Branch Divergence）

* **传统机制（Pascal 及之前 - SIMT Stack）**：
* 整个 Warp 共享 1 个 PC。
* 发生 `if-else` 分歧时，硬件通过 **Active Mask**（谓词掩码）串行化执行各分支（先激活 Then 路径线程，Disable 其余；再反转 Mask 执行 Else 路径），最终在汇合点（Reconvergence Point）出栈恢复全掩码。最坏情况性能衰减至 $\frac{1}{2^n}$。


* **Volta 革命（独立线程调度 - Independent Thread Scheduling）**：
* **硬件重构**：每个线程维护独立的 PC 和调用栈（Call Stack）。
* **动态收敛**：Warp 调度器在运行时扫描所有线程，**将当前 PC 相同的活跃线程动态打包为一个 SIMT 向量指令**发射。
* **收益与代价**：消除了 Warp 内锁死隐患，支持细粒度线程协作；但打破了隐式 Warp 锁步（Lock-step）假设，软件必须显式调用 `__syncwarp()`。



### 3. 访存合并（Memory Coalescing）

* **硬件机制**：32 个线程生成 32 个独立的虚拟/物理内存地址，输入至访存合并单元（Address Coalescer）。
* **优化目标**：将尽可能多的相邻、连续地址聚合为最少次数的 32B / 64B / 128B 内存事务（Memory Transaction）。
* **架构边界**：
* 连续且对齐访问（Stride = 1）：32 个 4B 访问合并为 **1 次 128B 事务**（100% 效率）。
* 非连续稀疏访问（Stride $\ge$ 64B）：退化为 **32 次 32B 事务**（总传输 1024B，有效数据仅 128B，带宽利用率跌至 3.125%）。



### 4. 寄存器堆（Register File）与上下文切换

* **物理成本**：每个 SM 配备 64K 个 32-bit 寄存器（256KB SRAM），占芯片相当大的面积与静态功耗。
* **零开销切换原理**：Block 分配到 SM 时，硬件根据编译期声明为所有线程**静态独占分配寄存器槽位**，上下文常驻在 SRAM Bank 中，切换 Warp 仅需切换寄存器基址指针，不需要类似 CPU 的 Push/Pop 内存存盘。
* **架构权衡（Occupancy vs. Spilling）**：
* 每线程分配寄存器过多 $\to$ SM 能容纳的 Resident Warp 减少 $\to$ **Occupancy 下降**，延迟难以隐藏。
* 强制限制每线程寄存器 $\to$ 寄存器不足发生 **Register Spilling**（数据溢出至 Local Memory / L1 / DRAM），带来几百周期的访存延迟。



---

## 模块三：不同技术路线演进与对比

* **NVIDIA 路线**：激进的硬件虚拟化。由统一 PC 演进到 Volta 独立线程调度（每线程独立 PC），并在 SIMT 通路上叠加专用硬件加速单元（Tensor Core、Transformer Engine、FP8 动态缩放、结构化稀疏）。
* **AMD 路线 (RDNA/CDNA)**：更接近宽 SIMD 向量机哲学。引入 **Wavefront（32 或 64 线程）**，采用显式掩码控制（Execution Mask），强调极宽 SIMD 单元与双精度（FP64）/矩阵（WMMA）吞吐。
* **宏观体系结构定性**：现代 GPU 是 **两级混合架构**——跨 SM/CU 之间是 **MIMD**（异步执行不同 Block/Kernel），SM 内部是 **SIMT 抽象驱动的 SIMD 物理流水线**。

---

## 模块四：面试问答速记模板（Q&A 准备）

### Q1: “请从硬件架构角度，解释为什么 GPU 的上下文切换（Context Switch）是零开销的？”

* **回答要点**：
1. **静态资源预留**：SM 拥有庞大的物理寄存器堆（如 64K×32-bit），每个线程块在 Launch 时被一次性分配专属的物理寄存器切片，状态全程常驻 SRAM。
2. **调度器即时轮询**：Warp Scheduler 维护所有 Warp 的就绪状态表，切换时仅需改变下一拍发射的 Warp ID 和寄存器基址映射，无需像 CPU 那样向内存/栈读写保存现场。



### Q2: “GPU 是如何处理 `if-else` 分支发散的？Volta 架构做了什么根本性改变？”

* **回答要点**：
1. **传统架构（SIMT Stack）**：统一 PC 模型。利用 Active Mask 进行硬件串行化（Serial Execution），依次执行 If 路径和 Else 路径，未命中分支的 Lane 硬件置 0 屏蔽（Predication），在收敛点弹出 Mask 恢复。
2. **Volta 架构（独立线程调度）**：每个线程配备独立的 PC 和栈，调度器动态识别并聚合处于相同 PC 的活跃线程并发射。支持了亚线程束（Sub-warp）粒度的分歧与交错推进，彻底消除了 Warp 级死锁风险。



### Q3: “什么是 Memory Coalescing？硬件在什么情况下会发生性能断崖式下跌？”

* **回答要点**：
1. **机制**：硬件访存合并单元将一个 Warp 内 32 个线程的独立访存地址按 Cache Line（32B/64B/128B）进行地址折叠，合并为最少次数的总线事务。
2. **性能陷阱**：当程序出现跨步访问（Stride $\gg$ 1）或指针离散跳转（如稀疏矩阵、图计算）时，合并机制失效，1 次 128B 的聚合事务退化为最多 32 次离散事务，总线有效带宽利用率可能骤降至个位数。



### Q4: “在做 GPU 性能建模或 CModel 设计时，如何评估 Occupancy 与性能的关系？”

* **回答要点**：
1. **平衡点而非越高越好**：Occupancy（活跃 Warp 比例）受限于每线程寄存器用量、每个 Block 的 Shared Memory 需求和最大线程限制。
2. **隐藏延迟的阈值效应**：当 Occupancy 达到一定阈值（通常 40%~50%），流水线已足以完全隐藏指令与基础访存延迟；此时盲目追求更高 Occupancy 可能会迫使编译器压低单线程寄存器导致 Register Spill，反而因为内存流量爆炸而恶化整体性能。