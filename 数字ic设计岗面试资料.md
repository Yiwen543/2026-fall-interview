# 何海林 — 硬件 / 数字 IC 面试详细复习资料

> 用法:第一部分「项目 + 简历数字」开场必背;第二部分「乱序处理器」是核心考区,从流程到各部件按关联顺序串起来;第三部分「时序/物理」;第四部分「验证 + 工具」;第五部分「手撕代码」考前默写;末尾「速查表」临场扫一眼。术语和代码保留英文,讲解用中文。

---

# 第一部分:项目介绍 + 简历数字

## 1.1 四个项目怎么讲


### 项目② 乱序 RISC-V 处理器设计(Columbia)
> 一句话:用 Verilog 设计了一个**乱序执行流水线**,32-entry ROB,集成 I/D Cache、BTB、双峰分支预测器,45nm 综合到 500MHz,CoreMark 上 CPI 1.3。

怎么讲:"我从头设计了一个乱序 RISC-V 核。前端取指译码后做**寄存器重命名**消除 WAW/WAR 假依赖;指令进**保留站(RS)**,操作数就绪后乱序发射,结果走 CDB 广播唤醒;后端用 **ROB 按程序顺序退休(commit)** 保证架构状态正确,并实现分支误预测恢复。我还集成了 I/D Cache、BTB 和双峰预测器,45nm 综合优化关键路径到 500MHz,CoreMark 上把 CPI 降到 1.3。"


### 项目④ 64-tap FIR 滤波器与形式验证(Columbia)
> 一句话:带 **异步 FIFO** 解决 10kHz↔1MHz 的 CDC;写 20+ SVA 用 JasperGold 形式验证证明 FSM 正确、FIFO 溢出不可达;10,000+ 随机回归对比 MATLAB 黄金模型。

怎么讲:"这是个数字信号处理 + 验证的项目。难点一是跨时钟域——输入 10kHz、处理 1MHz,我用异步 FIFO + 格雷码 + 两级同步器解决 CDC 和亚稳态。难点二是验证——我写了 20 多条 SVA,用 Cadence JasperGold 做形式验证,严格证明了状态机控制逻辑正确、FIFO 溢出状态不可达,且无反例;另外跑了上万组随机种子回归,和 MATLAB 黄金模型比对定点精度。"

可追问:见 CDC/亚稳态(3.3)、异步 FIFO(5.2)、**MATLAB 精度怎么对(4.3)**、形式验证 vs 仿真(4.1)。

## 1.2 简历数字都要能讲(常被追问"这个数怎么来的")

> 原则:每个写在 CV 上的数字,都要知道**怎么测的、为什么是这个值、好不好**。被追问时不能只报数,要能解释。

| 数字 | 怎么来的 / 怎么讲 | 好不好 / 追问防守 |
|---|---|---|
| **CPI 1.3**(CoreMark) | 跑 CoreMark 基准,用**总周期数 ÷ 总指令数**得到。仿真里从性能计数器读 retired instruction 数和 cycle 数相除。 | 乱序 + 分支预测 + 非阻塞 cache 把 CPI 从顺序核的 ~2 压到 1.3,接近理想 1.0。剩下的 >1 主要来自 cache miss、分支误预测冲刷、结构冒险。 |
| **500 MHz**(45nm) | Design Compiler 综合后,`create_clock` 设 2ns 周期,`report_timing` 确认 setup slack ≥ 0(WNS 转正)才算收敛。 | 45nm 教学工艺下,乱序核逻辑深、关键路径在唤醒–发射回路(见 3.1),500MHz 已是合理目标;更高要更激进流水或更先进工艺。 |
| **45nm** | 用的标准单元库工艺节点(Nangate 45nm 之类开源库)。 | 是学术/教学常用节点,面试官不会苛求先进工艺,重点是你会跑综合流程。 |
| **32-entry ROB** | 重排序缓冲的深度,决定**最多同时在途(in-flight)多少条指令**。 | 32 是性能/面积折中:太小限制乱序窗口(ILP 挖不出来),太大面积和 CAM 比较代价高。 |
| **推理延迟降 35%**(BERT) | 用**基线(未优化 HLS)的推理周期数**对比 **DSE 优化后的周期数**,`(base−opt)/base`。 | 为什么不是更多 → **Amdahl**:kernel(GEMM)级加速很大,但系统级延迟还含不可流水的数据搬运,所以我同时做了 DMA 带宽匹配(见 2.11)。 |
| **~60% LUT**(VC707) | Vivado 综合实现后的 utilization 报告里读 LUT 占用百分比。 | 说明设计规模适配这块板、还留了余量。可追问 DSP/BRAM 占用(GEMM 的 MAC 吃 DSP、权重缓存吃 BRAM)。 |
| **512-bit AXI** | AXI 数据通道位宽,调到和 **NoC DMA 平面等宽**,一拍搬更多权重。 | 位宽 = NoC 平面宽,消除 I/O 瓶颈;窄了带宽不够喂满 GEMM 阵列。 |
| **20+ SVA** | 手写的 SystemVerilog Assertion 条数,覆盖 FSM 状态合法性、FIFO 不溢出/不下溢等属性。 | 用 JasperGold **形式验证**证明,不是仿真跑通,是数学证明属性恒成立(见 4.1)。 |
| **10,000+ 随机回归** | 用随机种子生成上万组输入,DUT 输出和 **MATLAB 黄金模型**逐点比对。 | 这是仿真侧的定点精度验证;精度怎么设计对齐见 4.3。 |
| **10kHz ↔ 1MHz** | FIR 的输入采样率 10kHz、内部处理时钟 1MHz,两个异步时钟域。 | 正是 CDC 的来源,用异步 FIFO + 格雷码 + 两级同步器解决(见 5.2、3.3)。 |

一句话心态:"数字不是背出来的,是我做出来的——每个都能讲清怎么测、为什么这么设、瓶颈在哪。"



---

# 第二部分:乱序 RISC-V 处理器(核心考区)

> 建议按 2.1 的完整流程先建立全局图,再看各部件(重命名/RS/ROB/LSQ/分支/CSR)。

## 2.1 【重点】乱序处理器完整指令执行流程

**总览(9 个阶段):**
`Fetch → Decode → Rename → Dispatch → Issue(乱序)→ Execute → Writeback/Broadcast → Commit(顺序退休)`,访存指令另走 LSQ。

**1. Fetch(取指)—— 顺序、推测**
- 用 PC 从 **I-Cache** 取指令;同时查 **BTB**(是不是跳转、跳到哪)和 **BHT/双峰预测器**(跳不跳)。
- 预测跳转就把 PC 重定向到预测目标,**推测性**继续取——分支预测在这里起作用。

**2. Decode(译码)**
- 解析 opcode、源/目的寄存器、立即数,识别指令类型(ALU / 访存 / 分支 / 跳转 / CSR)。

**3. Rename(寄存器重命名)—— 消除假依赖**
- 查 **RAT(Register Alias Table)**:架构源寄存器 → 当前物理寄存器;
- 从 **Free List** 取空闲物理寄存器给目的寄存器,更新 RAT;
- 消除 **WAW/WAR 假依赖**,只留真正 RAW,乱序才能展开。
- **分支的 checkpoint 通常就在这一阶段打(见 2.8)。**

**4. Dispatch(分派)—— 进 ROB 和 RS**
- **ROB 尾部(tail)分配 entry**、`tail++`(记程序顺序,保证顺序退休);
- 同时送进 **保留站(RS / Issue Queue)** 等操作数;
- 访存指令还要在 **LSQ** 分配表项。

**5. Issue(发射)—— 乱序,这是心脏**
- RS 每条指令**监听 CDB**:等待的源操作数 tag 被广播命中 → 标 ready(**唤醒 wakeup**);
- **选择逻辑(select)** 从"两操作数都 ready"的指令里按年龄/优先级挑出本周期发射的;
- **谁先 ready 谁先发,和程序顺序无关** —— 这就是"乱序"。(关键路径在这,见 3.1。)

**6. Execute(执行)**
- 功能单元运算:ALU、乘除、分支比较、**AGU 算访存地址**;多周期单元可流水/占多拍。
- **分支在这里被解析**:算出真实方向/目标,和预测比对 → 误预测触发 flush。

**7. Writeback / Broadcast(写回 / 广播)**
- 结果写进**物理寄存器**;
- `<结果 tag, 数据>` 上 **CDB 广播**,唤醒 RS 里等它的指令(回到第 5 步形成唤醒–发射回路);
- ROB 对应表项标 **done**。

**8. Commit / Retire(提交退休)—— 顺序,保证正确**
- **ROB 从 head 按程序顺序**:head 指令 done 就退休、`head++`;
- 退休时才**真正写架构状态**、**store 才真正写内存**、**CSR 写和异常才生效** → 实现**精确异常**;
- 保证对外可见状态永远是**程序顺序**的。

**一句话串起来(现场背):**
"取指用 BTB+预测器推测取指;译码后**重命名**消除 WAW/WAR、并打 checkpoint;**分派**同时进 ROB(记程序顺序)和保留站(等操作数);操作数在 CDB 上被广播唤醒后**乱序发射**执行;结果写回物理寄存器并广播唤醒后续指令;最后 **ROB 从 head 顺序退休**,退休时才写架构状态、才真正 store、才处理 CSR/异常,实现精确异常。访存走 LSQ 靠 store-to-load forwarding;误预测/异常时不清数据,只回退指针 + 清 valid,并回滚 RAT、归还物理寄存器、清 speculative store。"

## 2.2 寄存器重命名(Rename)

- 目的:消除 **WAW / WAR 假依赖**(名字冲突,不是真数据依赖),只留 RAW 真依赖。
- 机制:**RAT** 存架构寄存器→物理寄存器的映射;**Free List** 管空闲物理寄存器。每条指令的目的寄存器分配一个新物理寄存器并更新 RAT,源寄存器查 RAT 拿到最新物理寄存器。
- 恢复:误预测时要把 RAT 恢复到分支点——**checkpoint 快照**(快)或**从 tail walk 回滚**(省面积)。

一句话:"重命名用 RAT + Free List 把架构寄存器映射到物理寄存器,消除 WAW/WAR,只留 RAW。"

## 2.3 保留站 / Issue Queue(RS)

- 作用:暂存已分派、但操作数还没齐的指令,等操作数就绪后**乱序发射**。
- **唤醒(wakeup)**:监听 CDB,广播的结果 tag 命中自己等的源 tag → 标该操作数 ready。
- **选择(select)**:从所有操作数齐的指令里,按年龄/优先级挑本周期发射的。
- 唤醒 + 选择必须**同周期**完成(否则背靠背相关指令发不连续),这条回路是关键路径(见 3.1)。

## 2.4 【新增】Issue Queue 在分支误预测时怎么操作

**核心认识**:分支误预测后,RS 里那些**比该分支更晚(program order 更年轻)、且是从错误路径进来的指令**必须被清掉;而更早的指令不受影响。

具体操作:
1. **按年龄清除错误路径的 RS 表项**:每条 RS 表项都带一个**标识它在 ROB 里位置 / 年龄的信息**(如 ROB index 或分支掩码 branch mask)。误预测的分支一解析,就把 RS 里**所有比它年轻的表项的 valid 位清掉**(相当于把它们 squash)。这些位置随后可被新指令(正确路径)重新分配。
2. **用 branch mask 精确清除(高性能做法)**:给每条在途指令打一个"依赖哪些未解析分支"的 mask。分支解析时:预测对 → 广播清掉大家 mask 里对应这一位;预测错 → **把 mask 里含这条分支的所有指令一次性 flush**。这样能精确、并行地清掉错误路径指令,不误伤正确路径。
3. **正在等待唤醒的操作数也一并失效**:被 squash 的指令即使操作数还没齐,也直接丢弃,不再参与唤醒/选择。
4. **和 ROB/RAT 恢复配套**:RS 清完的同时,ROB 回退 tail、RAT 回滚、物理寄存器归还、LSQ 清 speculative 表项(见 2.6、2.7)。

**简单核 vs 高性能核:**
- **简单(commit 点解析)**:等分支走到 ROB head 才确认误预测 → 直接**整个 RS + ROB 清空**,从正确目标重取。实现简单,但惩罚大(错误路径拖到 head 才清)。
- **高性能(分支一解析就清)**:分支在 Execute 一算出误预测,立刻用 branch mask **精确 squash 错误路径的 RS 表项**,更早止损。

一句话:"误预测时,Issue Queue 要把**所有比该分支年轻的、错误路径上的表项 valid 清掉**(简单核直接清空整个 RS,高性能核用 branch mask 精确 squash),这些表项随后被正确路径的新指令重新分配;同时配合 ROB 回退、RAT 回滚、物理寄存器归还。"

## 2.5 ROB(Reorder Buffer)—— 重点:flush 时指针怎么操作

**是什么**:**循环缓冲(circular buffer)**,按**程序顺序**记录所有在途指令。两个指针:
- **head(retire/commit)**:指向最老、即将退休的指令。
- **tail(allocate)**:指向下一个要分配的空位。
- 各带一个 **wrap bit**(多一位)区分空/满,和 FIFO 一个道理。

**正常流程**:dispatch 在 tail 分配、`tail++`;执行完标 `done`;commit 时 head 的指令 done 就顺序退休、`head++`,写回架构状态。

**flush 时——"空出一大段指针怎么操作":**
关键:**flush 不逐个清数据,只移指针 + 清 valid,被丢弃的 entry 下次分配直接覆盖。**
- **简单方案(commit 点解析)**:误预测分支走到 head 才确认 → **整个 ROB 清空**(`head = tail`,valid 全清),从正确目标重取。head 和 tail 重合,中间一大段整体丢弃。
- **高性能方案(分支一解析就 flush)**:执行单元算出误预测 → **tail 回退到该分支 entry 的下一个位置**,丢弃其后 dispatch 的所有 entry(清 valid);**head 不动**(分支及更早指令照常 commit)。被丢弃的"空指针"区间下次 allocate 直接覆盖。

**flush 配套恢复(体现深度):**
1. **RAT 恢复**:checkpoint 快照直接恢复,或从 tail walk 回滚。
2. **物理寄存器归还 Free List**。
3. **LSQ / Store Buffer**:清被丢弃指令的 **speculative store**。
4. 前端 PC 重定向到正确目标,重取。

一句话:"ROB 是循环缓冲,head 退休、tail 分配。flush 时不清数据,只 **tail 回退 + 清 valid**(或整表 head=tail),同时**回滚 RAT、归还物理寄存器、清 speculative store**,被丢弃指针段下次分配覆盖。"

## 2.6 Load-Store Queue(LSQ / 访存队列)

- 作用:维护访存指令间的内存依赖(RAW/WAW/WAR),保证乱序下访存语义正确。
- **Store-to-Load Forwarding**:Load 地址命中 LSQ 里**未提交的、更早的 Store** → 直接从队列转发数据,不访 Cache,省延迟。
- **内存消歧(disambiguation)**:Load 发射时前面 Store 地址可能未知 → 保守等待,或推测执行(speculative load),错了 flush 重做。
- 配合**非阻塞 Cache**:Load miss 不阻塞后续,用 **MSHR** 记在途 miss。

一句话:"LSQ 跟踪访存依赖;关键是 store-to-load forwarding——Load 命中前面未提交的 Store 就直接转发。"

## 2.7 CSR(Control and Status Register)

- **两大作用**:
  1. **CPU 的"控制面板"**:控制功能开关和状态(中断使能、特权级、性能计数器等)。
  2. **服务异常/中断的跳转**:异常/中断发生时**保存现场**(PC、原因存进 CSR),按优先级服务,完再恢复返回。
- **关键 CSR(RISC-V)**:`mepc`(被打断指令地址/返回地址)、`mcause`(原因)、`mstatus`(全局状态含 MIE)、`mtvec`(handler 入口)、`mie/mip`(中断使能/挂起)。
- **CSR 指令**:`csrrw`(读写)、`csrrs`(读后置位)、`csrrc`(读后清位)。
- **为什么 CSR 写必须在 commit**:改的是**全局架构状态**、难回滚;只有指令确认退休才能改,配合 ROB 实现**精确异常**——异常时清流水、从 ROB 回滚,保证 `mepc`/`mcause` 记录精确状态。
- **中断流程**:中断 → 存现场(`mepc`/`mcause`/`mstatus`)→ 跳 `mtvec` 的 handler → 按优先级处理 → `mret` 恢复返回。

一句话:"CSR 既是控制开关的'控制面板',又负责异常/中断的现场保存与跳转;因为改全局架构状态,写放 commit,靠 ROB 实现精确异常。"

## 2.8 分支预测(BTB + BHT)+【新增】checkpoint 在哪个阶段

- **BTB vs BHT**:BTB 回答"跳到哪(目标地址)",带 Tag、项大数少;BHT(双峰里的 PHT)回答"跳不跳(方向)",存 **2-bit 饱和计数器**,无 Tag、项小数多。
- **2-bit 饱和计数器** 4 态:`11` 强跳 / `10` 弱跳 / `01` 弱不跳 / `00` 强不跳,MSB 是预测。
- **BHT 大小** = 表项数 × 2 bit。索引位数 k → 2ᵏ 项。**默认报:1024 项 × 2 bit = 256 B,PC[11:2] 索引**(RV 无 C 扩展、4 字节对齐丢最低两位)。重点是能从索引位数推出大小。
- **超大程序够不够**:不够,**别名冲突(aliasing)**——上万条静态分支挤进 1024 项,多条共享计数器互相破坏。加分句:表只需覆盖某阶段**活跃分支工作集**,工作集大就扛不住。
- **优化三层递进**:① 加大 BHT(减冲突,代价面积/功耗/延迟);② **gshare**(索引 = PC ⊕ 全局历史 GHR,捕捉相关性又打散冲突);③ **Tournament / TAGE**(TAGE 用带 Tag、几何级数历史长度的多张表,化解别名,是 SOTA)。金句:**开芯院主导的香山用 TAGE 系列(TAGE-SC-L)**。

**【新增】分支恢复的 checkpoint(状态快照)打在哪个阶段?**
- **打在 Rename 阶段**。原因:重命名是**改架构状态映射(RAT)**的地方,而误预测恢复的核心就是把 **RAT / Free List / 相关指针**恢复到分支那一刻。所以在**分支指令重命名时,给它拍一份状态快照(checkpoint)**——主要是 **RAT(映射表)快照**,可能还有 Free List 指针、ROB/LSQ 尾指针等。
- 误预测时:直接**加载该分支的 checkpoint** 一步恢复 RAT,而不用逐条 walk 回滚,恢复快、惩罚小。
- 权衡:checkpoint 每个未解析分支存一份,**面积换速度**;所以通常**限制同时在途的未解析分支数**(比如最多 N 个分支 checkpoint)。省面积的替代是**从 ROB tail 往回 walk 回滚 RAT**(慢但省存储)。
- 也有设计把恢复放在 commit 用"退休时的架构 RAT(aRAT)"恢复——最简单但惩罚最大。

一句话:"**checkpoint 打在 Rename 阶段**——因为重命名是改 RAT 的地方,分支重命名时拍一份 RAT(及相关指针)快照;误预测时直接加载该快照一步恢复,比逐条 walk 快。代价是每个未解析分支存一份,所以限制在途分支数;省面积可改成从 ROB 回滚。"

## 2.9 AXI4 协议

- **五通道**:写 AW(地址)/ W(数据)/ B(响应);读 AR(地址)/ R(数据)。
- **握手**:每通道基于 **VALID/READY**——发送方拉 VALID、接收方拉 READY,同时为高的那拍完成传输。VALID 不能等 READY 才拉(避免死锁)。
- **突发(Burst)**:一次地址握手对应多拍数据,由 `AxLEN`(拍数)、`AxSIZE`(每拍字节)、`AxBURST`(FIXED/INCR/WRAP)描述。BERT 里 burst 大幅提升 DMA 搬权重带宽。
- **Outstanding**:用 `AxID` 支持多请求未完成时继续发新请求,响应可乱序返回(同 ID 保序、不同 ID 可乱序),掩盖长延迟。
- **AXI-Lite**:简化版,无 burst、单拍,用于配置寄存器。

一句话:"AXI 是分通道、基于 VALID/READY 握手的总线;靠 burst 提带宽,靠 ID 支持 outstanding 和乱序返回。"

## 2.10 JALR 与 Round-Robin

**JALR(Jump And Link Register):**
- 格式 `jalr rd, offset(rs1)`;行为:**目标 = (rs1 + offset) & ~1**(最低位清零保证对齐),**返回地址 PC+4 写入 rd**。
- vs **JAL**:JAL 是 **PC 相对**直接跳转(目标编译期已知);**JALR 是寄存器间接跳转**(目标运行时由 rs1 决定)。
- 用途:函数返回 `ret` = `jalr x0, 0(ra)`;函数指针/间接调用/虚函数;配 `auipc` 拼 32 位长跳转。
- 预测:间接跳转,目标不固定,靠 **BTB / RAS(Return Address Stack)**——函数返回用 RAS 最准。

一句话:"JALR 是寄存器间接跳转,目标=(rs1+offset)最低位清零,返回地址存 rd;ret 是其特例,预测靠 BTB 和 RAS。"

**Round-Robin(轮转仲裁):**
- **是什么**:多请求者竞争共享资源时的**公平**授权策略,轮流给,不让任何一个饿死(starvation-free)。
- **机制**:维护 `last_grant` 指针;每次**从 last_grant 下一个位置**按固定顺序找第一个有请求的授权,更新指针到它。刚被服务的下一轮排最后。
- **vs 固定优先级**:固定优先级永远先给编号小的,高优先级不断时低优先级饿死;round-robin 靠旋转起点消除饿死。
- **硬件实现**:"**旋转–优先编码–反旋转**"——请求向量按 last_grant 循环右移,普通优先编码找最低位 1,再移回去;或用 mask 分"高于/低于指针"两段。
- **在你项目里**:SPM 的 bank 仲裁,Matrix/Vector 用优先级,**DMA/Noc 之间用 round-robin** 保证公平。

一句话:"round-robin 是公平轮转仲裁,用指针记住上次授权者,每次从它下一个开始找请求,不饿死;硬件用旋转–优先编码–反旋转。我 SPM 里 DMA 和 Noc 就用它。"

---

# 第三部分:时序与物理实现

## 3.1 关键路径 / STA / TCL + 【重点】一条具体 critical path 例子

- **关键路径**:两寄存器间**延迟最长**的组合路径,决定最高频率(`f_max = 1/T_critical`)。
- **STA**:不靠激励,穷举所有时序路径,检查 setup/hold。工具:PrimeTime、Design Compiler。
- **常用 TCL(STA 流程)**:
  ```tcl
  create_clock -name clk -period 2.0 [get_ports clk]   ;# 定义时钟(500MHz=2ns)
  set_input_delay  0.5 -clock clk [all_inputs]          ;# 输入延迟约束
  set_output_delay 0.5 -clock clk [all_outputs]         ;# 输出延迟约束
  set_clock_uncertainty 0.1 [get_clocks clk]            ;# 时钟不确定性(jitter/skew 余量)
  report_timing -delay max -max_paths 10                ;# setup 路径报告(找 WNS)
  report_timing -delay min                              ;# hold 路径报告
  report_qor                                            ;# 总体质量(WNS/TNS)
  ```
- **优化思路**:从 `report_timing` 找 **WNS(Worst Negative Slack)** 路径 → gate sizing、buffer insertion、逻辑重构/流水打拍切长路径、调综合约束。

### 【重点】乱序核里一条具体 critical path(常被问,背熟)

**场景:唤醒–发射(wakeup-select)回路 + CDB 广播路径**

碰到的关键路径:`report_timing` 报的 WNS 出现在**发射阶段的唤醒–选择回路**:
> 功能单元算完结果 → 结果 tag 上 **CDB** → 广播到**保留站所有表项** → 每项拿广播 tag 和自己等的两个源 tag **比较** → 命中标 ready → **选择逻辑**判断哪些指令两操作数都 ready → **优先编码**选出本周期发射的 → 送功能单元。

这一整条**必须在一个周期内完成**(唤醒和选择同周期,否则背靠背相关指令发不连续),RS 几十个表项、每项两个比较器,广播扇出极大 + 比较 + 宽优先编码串联,组合延迟长,成为限制 500MHz 的关键路径。

**为什么是关键路径**:① CDB 扇出大(驱动所有表项比较器,负载大);② 比较(唤醒)→ ready 汇聚 → 宽优先编码(选择)**串行**;③ **不能简单打拍**(拆两拍破坏背靠背连续发射,IPC 掉)。

**怎么优化(讲这几点)**:
1. **比较逻辑并行前置 + 预解码**:缩短唤醒到 ready 的组合深度;
2. **CDB 广播分段插 buffer**(buffer insertion):降单级扇出负载,重载长线拆几段;
3. **选择逻辑改树形优先编码**(log 级):减仲裁逻辑级数;
4. **RS 分区**:按功能单元拆小 RS,减每个比较网络扇出/规模;
5. **gate sizing**:关键单元换高驱动强度标准单元。

**结果**:靠"比较前置 + 广播分段缓冲 + 树形选择编码"把这条路径延迟压下来,WNS 转正,满足 500MHz。

**一句话(现场说)**:"我碰到的关键路径是**发射阶段的唤醒–选择回路**:CDB 广播结果 tag 到保留站所有表项做比较唤醒,再经优先编码选出发射指令——必须同周期完成又不能打拍,扇出和逻辑深度都大。我用**广播总线分段插 buffer、选择用的优先编码改树形、比较逻辑并行前置**,压下延迟,最终满足 500MHz。"

**可能追问**:
- *为什么不流水切两拍?* → 唤醒和选择必须同周期,否则相关指令发不连续,IPC 降。
- *还有别的关键路径吗?* → 也可举 **ALU → 前递(bypass)网络 → 下一条 ALU 输入**,或 **LSQ 里 store-to-load forwarding 的地址比较**,优化思路类似(减扇出、树形化、加 buffer)。

## 3.2 setup / hold slack(STA 核心公式,记牢)

- **Setup**:数据要在(捕获沿 + 周期)前 setup 时间到达。
  `setup_slack = T_clk + T_skew − (T_cq + T_comb_max + T_setup)` ≥ 0。
  setup 违例 → **降频**能修(周期变长),升频更糟。
- **Hold**:数据要在捕获沿后保持 hold 时间(不能太早变)。
  `hold_slack = (T_cq + T_comb_min) − (T_hold + T_skew)` ≥ 0。
  hold 违例 → 和频率**无关**,升降频都修不了;靠加 buffer 延长短路径、调时钟树。
- **最大频率**:`T_min = T_cq + T_comb_max + T_setup − T_skew`,`f_max = 1/T_min`。

**为什么会有 setup 违例(物理实现视角)**:根本原因在**时钟的物理实现(时钟树)**——理想时钟同时到达所有触发器,实际做不到:
- **时钟偏斜(skew)**:时钟树布线不等长,到发送端/捕获端时刻不同;
- **时钟抖动(jitter)**:PLL/时钟源周期不稳,实际周期时长时短;
- **布线与单元延迟 + PVT**:互连 RC、单元驱动、工艺/电压/温度波动使组合延迟变大。
这些叠加使数据赶不上(捕获沿 + setup 时间),slack 变负。修:降频、优化时钟树减 skew、gate sizing/buffer 缩短数据路径、加流水打拍。

一句话:"setup 违例主要来自**时钟树的物理实现**——skew、jitter 加布线/单元延迟和 PVT 波动,让数据赶不上捕获沿前的 setup 窗口;降频或缩短数据路径可修。hold 和频率无关,靠加 buffer / 调时钟树。"

## 3.3 CDC / 亚稳态(上芯片验证常问)

- **亚稳态**:信号在时钟沿附近变化,触发器进入既非 0 也非 1 的不稳定态,可能传播错误值。
- **防护**:① 单 bit 用 **2-FF(或 3-FF)同步器**;② 多 bit 总线跨域用**格雷码**(一次只变一位)或**握手 / 异步 FIFO**;③ 绝不让多 bit 二进制总线直接打两拍(各位到达不一致会采到不存在的中间值)。
- 我的 FIR 项目用异步 FIFO + 格雷码 + 两级同步解决 10kHz↔1MHz 的 CDC。

**慢时钟域 → 快时钟域的方案(除异步 FIFO 外)【新增】:**
- **方案 A:握手(类似 AXI 的 VALID/READY)——双向可用**:发送方拉 VALID、接收方拉 READY,**同时为高那拍完成一次传输**;每次安全传一拍,靠握手确认双方就绪。代价:每笔几个周期握手往返,吞吐低,适合控制信号/低速数据。
- **方案 B:慢域挂 valid、快域来取——仅慢→快**:慢域**数据一准备好就拉 valid 并保持**;快域(采样快不漏)看到 valid 就取。为什么只适用慢→快:**快域采样频率高,慢域 valid 持续多个快周期,快域一定采到**;反过来快→慢慢域可能漏采。valid 单 bit 仍过两级同步器,数据在 valid 期间保持稳定。

一句话:"慢→快除异步 FIFO,可用 **VALID/READY 握手**(握上传一拍,双向可用),或 **慢域挂 valid、快域看到就取**(仅慢→快,因快域采样快不漏 valid);后者数据在 valid 期间稳定,单 bit valid 仍需两级同步。"

---

# 第四部分:验证方法与信号处理

## 4.1 形式验证 vs 仿真

- **仿真(simulation)= 暴力轮询**:testbench 塞大量激励(定向 + 随机),一个个跑看结果对不对。覆盖率靠激励堆,**只能覆盖你跑到的情况**,角落发现不了。
- **形式验证 = 数学证明**:不靠激励,用 SVA 写属性,工具穷举**所有可能状态**,要么证明恒成立、要么给反例(counterexample)。是"对所有输入成立"的严格保证。
- **各自适用规模**:
  - **仿真**:适合**大**设计(整个 SoC、处理器全系统),状态空间太大 formal 跑不动,靠仿真 + 覆盖率。
  - **形式验证**:适合**小而关键**模块(FSM、仲裁器、FIFO 控制、协议接口),状态空间可控能穷举;太大会**状态爆炸(state explosion)**。
- 我的 FIR 项目正是用 formal 证明 **FSM 正确、FIFO 溢出不可达**(小而关键);整体定点精度用 10000+ 随机回归(仿真)对比 MATLAB。

一句话:"仿真像暴力轮询,靠海量激励测你跑到的情况,适合大设计;形式验证是数学穷举证明对所有情况成立,适合小而关键模块,大了会状态爆炸。"

## 4.2 FIR vs IIR(信号处理)

- **核心区别:有没有反馈**。FIR 只用输入(`y[n]=Σ b_k·x[n-k]`),无反馈;IIR 把过去输出反馈回来(`y[n]=Σ b_k·x[n-k] − Σ a_k·y[n-k]`)。
- **由此的差异**:FIR 冲激响应有限、**永远稳定**(只有零点)、能做**严格线性相位**(系数对称)、无反馈好流水/好 CDC;IIR 冲激响应无限、**可能不稳定**(极点跑出单位圆)、一般非线性相位、有反馈难流水。
- **阶数区别(重点)**:**同样频响指标,IIR 需要的阶数远少于 FIR(常少 5~10 倍)**。因为 FIR 只有零点,要陡峭滚降只能堆 tap;IIR 有极点,能在某处急剧拉升/骤降,少数阶就做出 FIR 要几十 tap 的效果。
- 我项目选 64-tap FIR 是对的:要**稳定 + 线性相位 + 无反馈好做 CDC 和 formal**,正是 IIR 做不到、而我的 async FIFO + JasperGold 场景需要的。

一句话:"FIR 没反馈、永远稳定、能线性相位,但陡滚降要堆阶数;IIR 有反馈、阶数省很多,但可能不稳定、相位失真。"

## 4.3 【新增】MATLAB 黄金模型的精度怎么对齐 / 怎么设计?

**背景**:硬件是**定点(fixed-point)**,MATLAB 黄金模型通常先用**浮点(double)**算"理论正确值"。两者不可能逐 bit 相等,所以要**设计一套定点精度方案,并定量比对误差**。

**怎么设计精度(讲这几步):**
1. **定字长格式 Q(整数位.小数位)**:先分析信号动态范围定**整数位**(防溢出),再按精度需求定**小数位**。比如 Q1.15(1 符号 + 15 小数)。字长是**精度 vs 面积/功耗**的折中——位数越多越准但硬件越大。
2. **系数量化**:FIR 的 tap 系数从浮点量化到定点,量化误差会改变频响 → 在 MATLAB 里**量化后重新画频响**,确认通带/阻带指标(如阻带衰减)仍满足要求。
3. **中间运算的位增长处理**:乘加会让位宽增长(两个 Q1.15 相乘变 Q2.30,64 个累加还要加 guard bits 防溢出)→ 设计**累加器位宽**,最后**舍入(rounding)/ 截断(truncation)+ 饱和(saturation)** 回目标输出字长。
4. **在 MATLAB 里建两版模型**:一版纯浮点(理论参考),一版**用 `fi` / Fixed-Point Designer 模拟和硬件完全一样的定点行为**(同样的字长、同样的舍入/饱和模式)。

**怎么"对精度"(定量比对):**
- **硬件 RTL 输出 vs MATLAB 定点模型**:理论上应**逐 bit 一致**(bit-exact)——这是首要检查,一致说明 RTL 实现和定点算法没写错。
- **定点 vs 浮点参考**:算**误差指标**量化精度损失,常用 **SQNR(信噪量化比)、最大绝对误差、均方误差**;确认误差在可接受范围(比如 < 1 LSB,或 SQNR 达到设计门槛)。
- **10000+ 随机回归**就是喂上万组随机输入,逐点比对 RTL、定点模型、浮点参考三者,统计误差分布,确认没有哪组输入误差超标。

**一句话(现场说):**
"硬件是定点、MATLAB 参考是浮点,不能逐 bit 等,所以我**设计定点字长格式(Q 格式)**——按动态范围定整数位防溢出、按精度需求定小数位,系数量化后在 MATLAB 重画频响确认指标仍满足,中间累加加 guard bits、最后舍入+饱和回目标字长。比对分两层:**RTL 和 MATLAB 定点模型要 bit-exact**(验证实现正确),**定点 vs 浮点参考算 SQNR / 最大误差**(量化精度损失在门槛内)。10000+ 随机回归就是喂上万组输入统计这个误差分布。"

---

# 第五部分:手撕代码题

## 5.1 【新增】阻塞 vs 非阻塞赋值(a / b / c 波形 + 代码)

> 最经典的语言基础题。一句话结论:**时序逻辑必须用非阻塞 `<=`,组合逻辑用阻塞 `=`,两者不要在同一个 always 块里混用。**

### 两段代码只差一个符号,综合出的电路完全不同

```systemverilog
// ===== 写法 A:非阻塞 <= =====  → 真正的两级移位寄存器
always_ff @(posedge clk or negedge rst_n) begin
  if (!rst_n) begin b <= 0; c <= 0; end
  else begin
    b <= a;      // 右边取的都是"沿之前的旧值"
    c <= b;      // 这里的 b 是【旧 b】,不是刚赋的 a
  end
end

// ===== 写法 B:阻塞 = =====   → 退化成一级,第二个 FF 白给
always_ff @(posedge clk or negedge rst_n) begin
  if (!rst_n) begin b = 0; c = 0; end
  else begin
    b = a;       // 立即生效,b 马上变成 a
    c = b;       // 此时 b 已经是 a 了 → c 直接等于 a
  end
end
```

### 采样值对照(设 a 只在第①个沿之前为 1)

| 时刻 | a(沿前) | 非阻塞 b | 非阻塞 c | 阻塞 b | 阻塞 c |
|---|---|---|---|---|---|
| 复位后 | 0 | 0 | 0 | 0 | 0 |
| 第①个沿后 | 1 | **1** | 0 | **1** | **1** |
| 第②个沿后 | 0 | 0 | **1** | 0 | 0 |
| 第③个沿后 | 0 | 0 | 0 | 0 | 0 |

### 波形图

```
clk    __|‾|__|‾|__|‾|__|‾|__
          ①    ②    ③    ④

a      ‾‾‾‾‾|__________________     只在①之前为 1

── 写法 A:非阻塞  b<=a; c<=b; ──────────────
b      _____|‾‾‾‾|_____________     ①后拉高
c      __________|‾‾‾‾|________     ②后才拉高 ← 比 b 晚一拍 ✅ 两级移位

── 写法 B:阻塞    b=a;  c=b;  ──────────────
b      _____|‾‾‾‾|_____________     同上
c      _____|‾‾‾‾|_____________     和 b 完全重合 ❌ 第二级 FF 没起作用
```

### 为什么会这样(讲清机理)

- **非阻塞 `<=`**:所有右边表达式在时钟沿**同时采样"旧值"**,等本次时间步结束再统一更新左边。所以 `c <= b` 拿到的是 **b 更新前的值** → 真的隔了一拍 → 两级移位寄存器。
- **阻塞 `=`**:像 C 语言一样**顺序执行、立即生效**。执行到 `c = b` 时,上一句已经把 b 改成 a 了 → c 直接拿到 a → b 和 c 同时变,等于两个 FF 装了同一个值。

### 最关键的一点:阻塞的结果**依赖语句顺序**

```systemverilog
// 把两句调换顺序,阻塞又变回移位寄存器了!
c = b;    // 先用旧 b
b = a;
```
同样的意图、只是换个顺序,电路就变了 —— 这就是**竞争(race condition)**。而非阻塞写法无论怎么调换顺序,结果都一样。**这正是时序逻辑必须用非阻塞的根本原因。**

### 使用规则(背这张表)

| 场景 | 用哪个 | 原因 |
|---|---|---|
| 时序逻辑 `always_ff` | **非阻塞 `<=`** | 与语句顺序无关,消除竞争,正确建模"同时更新" |
| 组合逻辑 `always_comb` | **阻塞 `=`** | 需要顺序求值,模拟组合信号的即时传播 |
| 同一 always 块混用 | **禁止** | 仿真/综合行为可能不一致,是典型 bug 源 |
| 同一变量被多个 always 块赋值 | **禁止** | 多驱动,综合报错或行为不定 |

**一句话(现场说):**"非阻塞在时钟沿统一采样旧值、统一更新,所以 `b<=a; c<=b` 是两级移位;阻塞是顺序立即生效,`b=a; c=b` 执行到第二句时 b 已经是 a,c 就直接等于 a,退化成一级、而且结果依赖语句顺序会产生竞争。所以时序逻辑一律用非阻塞,组合逻辑用阻塞。"

---

## 5.2 同步 FIFO(读写同一时钟)
核心:指针用 **ADDR_W + 1 位**,多出的最高位区分空和满。

```systemverilog
module sync_fifo #(
  parameter DATA_W = 8,
  parameter DEPTH  = 16,
  parameter ADDR_W = $clog2(DEPTH)
)(
  input  logic              clk, rst_n,
  input  logic              wr_en, rd_en,
  input  logic [DATA_W-1:0] wr_data,
  output logic [DATA_W-1:0] rd_data,
  output logic              full, empty
);
  logic [DATA_W-1:0] mem [0:DEPTH-1];
  logic [ADDR_W:0]   wr_ptr, rd_ptr;   // 多一位

  always_ff @(posedge clk or negedge rst_n)
    if (!rst_n)              wr_ptr <= '0;
    else if (wr_en && !full) begin
      mem[wr_ptr[ADDR_W-1:0]] <= wr_data;
      wr_ptr <= wr_ptr + 1'b1;
    end

  always_ff @(posedge clk or negedge rst_n)
    if (!rst_n)               rd_ptr <= '0;
    else if (rd_en && !empty) rd_ptr <= rd_ptr + 1'b1;

  assign rd_data = mem[rd_ptr[ADDR_W-1:0]];
  assign empty = (wr_ptr == rd_ptr);                       // 空:完全相等
  assign full  = (wr_ptr[ADDR_W-1:0] == rd_ptr[ADDR_W-1:0]) &&
                 (wr_ptr[ADDR_W]     != rd_ptr[ADDR_W]);    // 满:低位等、wrap bit 不同
endmodule
```
边写边讲:"指针多一位。**空**是两指针完全相等;**满**是低位地址相等、但最高位翻转一次(写指针多绕一圈)。"

## 5.3 异步 FIFO(读写不同时钟)—— 带详细注释 + 空满判断在哪个时钟域

比同步版多三件事:**① 指针用格雷码 ② 跨域两级 FF 同步 ③ 空/满各在自己的时钟域判断**。

```systemverilog
module async_fifo #(
  parameter DATA_W = 8,
  parameter DEPTH  = 16,
  parameter AW     = $clog2(DEPTH)          // 地址位宽=4;指针用 AW+1=5 位
)(
  // ===== 写时钟域接口 =====
  input  logic              wclk, wrst_n, winc,
  input  logic [DATA_W-1:0] wdata,
  output logic              wfull,
  // ===== 读时钟域接口 =====
  input  logic              rclk, rrst_n, rinc,
  output logic [DATA_W-1:0] rdata,
  output logic              rempty
);
  logic [DATA_W-1:0] mem [0:DEPTH-1];      // 双端口 RAM:写域写入、读域读出

  // 指针都用 AW+1 位:低 AW 位当地址用,最高位是"绕圈标志(wrap bit)"
  logic [AW:0] wbin, wgray, wbin_n, wgray_n;   // 写指针:二进制版 + 格雷码版
  logic [AW:0] rbin, rgray, rbin_n, rgray_n;   // 读指针:二进制版 + 格雷码版

  // 跨时钟域同步用的两级寄存器(打两拍,防亚稳态)
  logic [AW:0] rgray_at_w1, rgray_at_w2;   // 【读指针】格雷码 → 同步进【写域】
  logic [AW:0] wgray_at_r1, wgray_at_r2;   // 【写指针】格雷码 → 同步进【读域】

  //===================================================================
  // 写时钟域(wclk):写数据、更新写指针、判断【满】
  //===================================================================
  assign wbin_n  = wbin + (winc & ~wfull);   // 没满才允许指针+1
  assign wgray_n = (wbin_n >> 1) ^ wbin_n;   // 二进制转格雷码:右移一位再异或

  // 写指针寄存器:二进制版用来寻址,格雷码版用来跨时钟域传递
  always_ff @(posedge wclk or negedge wrst_n)
    if (!wrst_n) {wbin, wgray} <= '0;
    else         {wbin, wgray} <= {wbin_n, wgray_n};

  // 真正写 RAM:用二进制指针的低 AW 位当地址
  always_ff @(posedge wclk)
    if (winc & ~wfull) mem[wbin[AW-1:0]] <= wdata;

  // 把【读指针】同步进【写域】—— 两级打拍
  always_ff @(posedge wclk or negedge wrst_n)
    if (!wrst_n) {rgray_at_w2, rgray_at_w1} <= '0;
    else         {rgray_at_w2, rgray_at_w1} <= {rgray_at_w1, rgray};

  // 判满:本地写指针(实时) vs 同步过来的读指针(滞后)
  // 格雷码下"写指针比读指针多绕了一圈" = 最高两位都相反、其余位相同
  assign wfull = (wgray_n == {~rgray_at_w2[AW:AW-1], rgray_at_w2[AW-2:0]});

  //===================================================================
  // 读时钟域(rclk):读数据、更新读指针、判断【空】
  //===================================================================
  assign rbin_n  = rbin + (rinc & ~rempty);  // 没空才允许指针+1
  assign rgray_n = (rbin_n >> 1) ^ rbin_n;

  always_ff @(posedge rclk or negedge rrst_n)
    if (!rrst_n) {rbin, rgray} <= '0;
    else         {rbin, rgray} <= {rbin_n, rgray_n};

  assign rdata = mem[rbin[AW-1:0]];          // 用二进制指针低位读 RAM

  // 把【写指针】同步进【读域】—— 两级打拍
  always_ff @(posedge rclk or negedge rrst_n)
    if (!rrst_n) {wgray_at_r2, wgray_at_r1} <= '0;
    else         {wgray_at_r2, wgray_at_r1} <= {wgray_at_r1, wgray};

  // 判空:本地读指针(实时) vs 同步过来的写指针(滞后);完全相等即为空
  assign rempty = (rgray_n == wgray_at_r2);
endmodule
```

### 【重点】判空/判满时,读写指针分别在哪个时钟域?

> 口诀:**"满在写域判,空在读域判;自己的指针是实时的,对方的指针是同步过来、滞后的。"**

| | **full(满)** | **empty(空)** |
|---|---|---|
| **在哪个时钟域算** | **写时钟域 wclk** | **读时钟域 rclk** |
| **写指针从哪来** | **本地原生** `wgray_n`(实时准确) | **同步过来** `wgray_at_r2`(滞后约 2 拍) |
| **读指针从哪来** | **同步过来** `rgray_at_w2`(滞后约 2 拍) | **本地原生** `rgray_n`(实时准确) |
| **判断条件** | 写格雷码 == 读格雷码「高两位取反、其余相同」 | 读格雷码 == 写格雷码(完全相等) |
| **谁被同步** | 读指针 rgray → 打两拍进写域 | 写指针 wgray → 打两拍进读域 |

**为什么必须这样安排(这是答题的核心):**

同步要打两拍,所以**每个域看到的"对方指针"永远是过时的旧值**。而这个"旧"恰好让判断偏向**保守**,因此是安全的:

- **写域判满**:看到的读指针偏旧 → 以为读侧读走的比实际少 → **偏早判满**(其实读侧已经腾出位置了)→ 结果只是少写几拍、损失一点吞吐,但**绝不会写溢出、覆盖还没被读走的数据**。
- **读域判空**:看到的写指针偏旧 → 以为写侧写入的比实际少 → **偏早判空**(其实已经有新数据了)→ 结果只是晚读几拍,但**绝不会读到还没写好的无效数据**。

**反过来就是致命 bug**:如果拿"滞后的指针"当自己的实时指针去判(比如在写域用同步来的写指针判满),误差方向就变成**乐观**的,会出现"该满没判满 → 覆盖未读数据"或"该空没判空 → 读出垃圾"。

**配套的两个基础点(顺带讲):**
1. **为什么用格雷码**:相邻值只变一位。跨域采样时即使正好撞上翻转,最多采错这一位,拿到的**要么是旧值要么是新值,都是合法值**;如果用二进制多位同跳,可能采到一个**根本不存在的中间值**,直接判错空满。
2. **为什么打两级 FF**:第一级可能进亚稳态,第二级再给它一个完整周期稳定下来,把亚稳态传播概率(MTBF)压到可接受范围。

**一句话(现场说):**"**满在写时钟域判、空在读时钟域判**。判满时用的是本地实时的写指针 + 打两拍同步过来的读指针;判空时用的是本地实时的读指针 + 同步过来的写指针。因为同步有 2 拍延迟,每个域看到的对方指针都偏旧,这让判断天然**偏保守**——偏早判满、偏早判空,只损失一点吞吐,但绝不会溢出覆盖或读到无效数据。指针用格雷码是为了跨域采样时最多错一位、非旧即新。"

## 5.4 【新增】序列检测状态机:检测 "1010"

> 手撕 FSM 的经典题。先问面试官一句:**"要重叠检测还是不重叠?"** —— 问这一句就赢了一半,说明你知道有区别。

### 状态定义(按"已匹配的前缀"来命名,最不容易错)

| 现态 | 含义(已匹配前缀) | in=0 → 次态 | in=1 → 次态 | 输出(Moore) |
|---|---|---|---|---|
| S0 | 无 | S0 | S1 | 0 |
| S1 | `1` | S2 | S1 | 0 |
| S2 | `10` | S0 | S3 | 0 |
| S3 | `101` | **S4** | S1 | 0 |
| S4 | `1010` ✓ | S0 | **S3** | **1** |

**几个关键转移的道理(会被追问):**
- **S1 收到 1 → 还是 S1**:`11` 的尾巴 `1` 仍是 `1010` 的前缀,重新从"已匹配 1"开始。
- **S2 收到 0 → 回 S0**:`100` 的任何后缀都不是前缀,彻底重来。
- **S3 收到 1 → 回 S1**:`1011` 的尾巴 `1` 是前缀,不用退到 S0。
- **S4 收到 1 → 回 S3(重叠检测的精髓)**:刚检测到 `1010`,它的尾巴 `10` 又能当新序列的前缀,再来个 `1` 就凑成 `101`。所以 `101010` 会检测到 **2 次**。
  - 若是**不重叠**检测:S4 直接当作重新开始 → in=1 回 S1、in=0 回 S0,`101010` 只检测到 1 次。

### 代码(标准三段式,本身就是考点)

```systemverilog
module seq_det_1010 (
  input  logic clk,
  input  logic rst_n,
  input  logic din,
  output logic det          // 检测到 1010 时拉高一拍
);
  typedef enum logic [2:0] {S0, S1, S2, S3, S4} state_e;
  state_e cs, ns;           // cs=当前态, ns=次态

  // ---- 第一段:状态寄存器(时序逻辑,用非阻塞 <=)----
  always_ff @(posedge clk or negedge rst_n)
    if (!rst_n) cs <= S0;
    else        cs <= ns;

  // ---- 第二段:次态逻辑(组合逻辑,用阻塞 =)----
  always_comb begin
    ns = cs;                          // 默认保持,防止综合出 latch
    case (cs)
      S0: ns = din ? S1 : S0;
      S1: ns = din ? S1 : S2;         // 收到1还停在S1(尾巴1仍是前缀)
      S2: ns = din ? S3 : S0;
      S3: ns = din ? S1 : S4;         // 收到0 → 凑成 1010,进接受态
      S4: ns = din ? S3 : S0;         // 重叠:尾巴"10"可复用 → 回S3
      default: ns = S0;
    endcase
  end

  // ---- 第三段:输出逻辑(Moore:只看状态,不看输入)----
  assign det = (cs == S4);
endmodule
```

### Moore vs Mealy(可能被追问)

| | Moore(上面这版) | Mealy |
|---|---|---|
| 输出取决于 | **只看现态** | 现态 **+ 当前输入** |
| 状态数 | 5 个(要一个专门的接受态 S4) | 4 个(S3 收到 0 时直接输出 1,次态回 S2) |
| 输出时机 | 比输入**晚一拍**(要等状态寄存器更新) | **同拍**就输出,更快 |
| 输出毛刺 | 无(输出是寄存器的函数,干净) | 可能有(输入的组合毛刺会传到输出) |

Mealy 版关键改动:去掉 S4,把输出写成 `assign det = (cs == S3) && (din == 1'b0);`,同时 S3 在 in=0 时次态直接回 **S2**(因为 `1010` 的尾巴就是 `10`)。

**一句话(现场说):**"我用**按已匹配前缀命名状态**的方法建 FSM,`1010` 需要 5 个状态(Moore)。关键在**失配时退到哪**——不是一律退回 S0,而是退到"当前尾巴能匹配上的最长前缀",比如 S3 收到 1 退到 S1、S4 收到 1 退到 S3。**S4 收到 1 回 S3 就是重叠检测的精髓**,因为 `1010` 的尾巴 `10` 能复用。写法用标准三段式:时序段用非阻塞、组合段用阻塞并给默认值防 latch、输出段 Moore 只看状态。"

---

## 5.5 信号上升沿检测 / 打一拍输出(用 reg)
**(a) 上升沿检测**(a 从 0→1 输出一拍脉冲):
```systemverilog
logic a_d;
always_ff @(posedge clk or negedge rst_n)
  if (!rst_n) a_d <= 1'b0;
  else        a_d <= a;
assign a_pulse = a & ~a_d;                  // 当前=1 且 上一拍=0
```
**(b) 打一拍寄存**(组合信号采样后输出对齐):
```systemverilog
always_ff @(posedge clk or negedge rst_n)
  if (!rst_n) a_q <= 1'b0;
  else        a_q <= a;
```
讲法:"检测拉高(上升沿)用 reg 存上一拍,`a & ~a_prev` 就是脉冲;只打拍寄存直接 `a_q <= a`。用 `always_ff` 是为同步、消毛刺、时序对齐。"

## 5.6 找最高位 / 最低位的 1 的索引(priority encoder)
```systemverilog
module lsb_index #(parameter W = 8)(
  input  logic [W-1:0]          x,
  output logic [$clog2(W)-1:0]  idx,
  output logic                  valid
);
  always_comb begin
    idx = '0; valid = 1'b0;
    for (int i = W-1; i >= 0; i--)          // 从高扫到低,最后命中=最低的 1
      if (x[i]) begin idx = i; valid = 1'b1; end
  end
endmodule
```
- **最低位**:for 从高到低,最后一次赋值落最低 set bit;**最高位**:反过来从低到高。
- **位运算**:最低位 1 的 one-hot = `x & (-x)`;清最低位 1 = `x & (x-1)`。
- 讲法:"本质是 priority encoder。找最低位从高往低扫,找最高位反着扫;也可用 `x & -x` 取最低位 1 的 one-hot 再编码。"

### 5.7 异步复位，同步释放（Asynchronous Reset, Synchronous Release）

* **为什么不能直接用异步复位？**
* 异步复位的撤销（Recovery/Removal）是随时发生的。如果撤销时刻刚好撞上时钟上升沿，触发器内部电路无法确定是采样旧值还是复位值，会导致 **亚稳态（Metastability）**。


* **电路结构与代码**：
* 使用两级 D 触发器（2-DFF Synchronizer），复位端接入外部异步复位信号 `rst_n`，输入端接死 `1'b1`。


```systemverilog
// 异步复位，同步释放电路
always_ff @(posedge clk or negedge async_rst_n) begin
    if (!async_rst_n) begin
        sync_rst_n_stage1 <= 1'b0;
        sync_rst_n        <= 1'b0;
    end else begin
        sync_rst_n_stage1 <= 1 meb1;
        sync_rst_n        <= sync_rst_n_stage1; // 同步释放后的复位信号
    end
end

```


* **一句话口诀**：“**复位生效异步（响应快、不依赖时钟），复位撤销同步（过两级 FF，避开 Recovery/Removal 违例）**。”

# 第六部分:高频追问速查

| 追问 | 一句话答 |
|---|---|
| 为什么用 Chisel / 比 SV 好在哪? | 内嵌 Scala 的硬件生成器,elaboration 出 Verilog;强参数化 + 类型安全(编译期抓位宽/连接错),适合大项目批量生成可配置模块。分清 Scala 世界(生成时求值)和硬件世界(UInt/Reg/when)。 |
| WAW/WAR 怎么解? | 寄存器重命名(RAT / Map Table)。 |
| 乱序怎么变顺序退休? | ROB 按程序顺序从 head 提交(commit)。 |
| Issue Queue 误预测时怎么办? | 清掉所有比该分支年轻的错误路径表项 valid(简单核清空整个 RS,高性能核用 branch mask 精确 squash),随后被正确路径重分配。 |
| 分支 checkpoint 打在哪个阶段? | Rename 阶段——重命名是改 RAT 的地方,分支重命名时拍 RAT 快照;误预测直接加载快照一步恢复。 |
| flush 时 ROB 怎么处理? | 不清数据;tail 回退 + 清 valid(或 head=tail),并回滚 RAT、归还物理寄存器、清 speculative store。 |
| Store-to-Load Forwarding? | Load 命中 LSQ 中未提交的更早 Store,直接转发,不访 Cache。 |
| CSR 写为什么在 commit? | 改全局架构状态,放 commit 才能配合 ROB 实现精确异常。 |
| exception vs misprediction? | 误预测是硬件猜错重来、不碰架构状态、不进 CSR;异常是陷入 handler、经 CSR 存现场跳转返回。 |
| JALR 干嘛的? | 寄存器间接跳转,目标=(rs1+offset)最低位清零,返回地址存 rd;ret 是特例,预测靠 BTB+RAS。 |
| round-robin 怎么做? | 指针记上次授权者,每次从它下一个找请求,不饿死;硬件用旋转–优先编码–反旋转。 |
| 阻塞 vs 非阻塞? | 时序用非阻塞 `<=`(沿上统一采旧值、与语句顺序无关);组合用阻塞 `=`。`b<=a;c<=b` 是两级移位,`b=a;c=b` 退化成一级且依赖语句顺序(竞争)。 |
| 同步 FIFO 判空满? | 指针多一位;空=完全相等,满=低位相等且 wrap bit 不同。 |
| 异步 FIFO 三要点? | 格雷码 + 两级 FF 同步 + 空/满各在自己时钟域判。 |
| 异步 FIFO 空满在哪个域判? | **满在写域**(本地实时写指针 + 同步来的读指针),**空在读域**(本地实时读指针 + 同步来的写指针);对方指针滞后使判断偏保守,绝不溢出/读到无效数据。 |
| 1010 序列检测怎么做? | 按"已匹配前缀"命名 5 个状态(Moore);失配退到"尾巴能匹配的最长前缀";**S4 收到 1 回 S3** 就是重叠检测精髓;三段式写法。 |
| DataShuffle 怎么实现的? | 实现 `dout[i]=din[i^mode]`,拆成 log₂N 级蝶形网络(每级只看 mode 一位决定是否互换),面积 O(N·logN) 而非 crossbar 的 O(N²);XOR 自逆,读写复用同一网络。 |
| 为什么用格雷码? | 相邻只变一位,跨域采样最多错一位,非旧即新。 |
| 亚稳态怎么防? | 单 bit 用 2-FF/3-FF 同步器,多 bit 用格雷码或异步 FIFO。 |
| 慢→快 CDC 除异步 FIFO? | ① VALID/READY 握手(握上传一拍,双向);② 慢域挂 valid、快域看到就取(仅慢→快)。 |
| BHT 多大 / 超大程序够吗? | 1024×2bit=256B(PC[11:2]);不够,别名冲突,用 gshare/TAGE。 |
| 一条具体 critical path 例子? | 唤醒–发射 CDB 广播回路;靠比较前置 + 广播分段插 buffer + 树形选择编码切。 |
| setup 违例物理成因 / 怎么修? | 时钟树 skew/jitter + 布线单元延迟 + PVT;降频或缩短数据路径可修。hold 和频率无关,加 buffer / 调时钟树。 |
| HLS II=1 的障碍? | RAW 循环携带依赖,或双端口 RAM 端口不足;靠 partition / 归约打断依赖。 |
| FIR vs IIR / 阶数? | FIR 无反馈、稳定、能线性相位但要堆阶数;IIR 有反馈、阶数省 5~10 倍但可能不稳定。 |
| MATLAB 精度怎么对? | 设计定点 Q 格式;RTL 和 MATLAB 定点模型要 bit-exact,定点 vs 浮点算 SQNR/最大误差。 |
| BERT 加速器做什么? | 加速 Transformer encoder 的矩阵乘(QKᵀ、softmax 加权 V、FFN 线性)+ softmax/layernorm。 |
| CV 数字怎么来的? | 见 1.2;CPI=周期/指令,500MHz 靠综合 WNS 转正,35% 是基线对比优化后周期,每个都能讲怎么测。 |

---

# 第七部分：查漏补缺——瑞芯微 MAC 调优、QLoRA 部署与二面高频实战

---

## 7.1 【项目扩展】瑞芯微 MAC（乘累加/算力单元）前后端调优实战

> **项目切入场景**：在面向 AI / DSP 的算力阵列（如 NPU / GEMM 引擎）中，MAC（Multiply-Accumulate）单元是**面积、功耗和关键路径时序（PPA）的最核心瓶颈**。

### 1. 前端（RTL / 架构）调优手段：

* **压缩算术树（Wallace Tree / Dadda Tree）**：
* 普通乘法器是加法链，组合逻辑延迟极长（$O(N)$）。RTL 调优使用 **Wallace / Dadda Tree 压缩结构**，利用 3:2 Compressor（Full Adder）和 2:2 Compressor（Half Adder）将 partial products（部分积）在 $O(\log N)$ 逻辑级数内压缩为两行，最后送给 CLA（超前进位加法器）。


* **Booth 编码（Booth-2 / Modified Booth）**：
* 采用 Booth 算法对乘数进行分组重编码（如 3-bit 窗口 Booth-2），将需要相加的部分积数量直接**减少一半**，大幅降低加法器树的面积与延迟。


* **Pipeline 切割（流水线切拍）**：
* 在 **乘法阵列（Multiplication Array）与 累加器（Accumulator）** 之间，或者在 Wallace Tree 压缩中途插入流水线寄存器（Pipeline Register）。
* *关键考点*：MAC 切拍会导致 1~2 拍的 **Read-After-Write (RAW) Data Hazard**。例如连续对同一个累加器加值：$Acc_{n} = Acc_{n-1} + (A_n \times B_n)$。
* *解决方案*：在 RTL 中增加 **Accumulator Bypass / Interleaving（通道交错）**，或者在硬件层实现多通道（Multi-channel/Multi-context）轮流累加，消除流水线停顿（Stall）。



### 2. 后端（Synthesis / P&R）调优手段：

* **DesignWare 库与 Arithmetic Optimization**：
* 在 Synopsys Design Compiler 综合时，避免让工具把 MAC 综合成低效的门电路，显式调用或自动映射到 **Synopsys DesignWare 乘法器库（如 DW_mult_seq, DW_mac）**，并开启 `set_dp_smart_generation`（Datapath 智能重构）。


* **Clock Gated & Operand Isolation（操作数隔离）**：
* 在 MAC 输入端加入 **Operand Isolation（操作数隔离 MUX/AND 门）**。当 MAC 处于空闲（Idle）或其输出不被下游采纳时，封锁输入端的数据翻转，防止数据进入乘法树内部引发海量电容充放电，**降低 30% 以上的动态功耗**。


* **Retiming（重定时优化）**：
* 综合/P&R 时开启 **`set_optimize_registers` / Register Retiming**。工具会自动将 MAC 内部和前后流水线寄存器向前或向后跨越组合逻辑推移，平衡乘法树与累加器之间的 Timing Slack，解决 WNS 违例。



---

## 7.2 【项目扩展】离线部署 QLoRA、剪枝、反量化与降低模型延迟

> **项目切入场景**：在资源受限端侧（如树莓派 5、NPU、Edge AI 芯片）上，部署 3B/7B 等 LLM 模型的核心瓶颈在于 **Memory Bandwidth Limit（内存带宽墙）** 和 **Compute Latency（计算延迟）**。

### 1. QLoRA 与 4-bit 量化 / 反量化（Dequantization）机制：

* **为什么量化能降延迟？**
* 端侧 LLM 推理（尤其是 Decode 阶段）是典型的 **Memory-Bound（访存受限型）** 任务。参数从 16-bit（FP16）压缩到 4-bit（NF4/INT4），**内存带宽需求直接降低 75%**，从而将瓶颈从“等 DDR 搬权重”转移回“计算”。


* **反量化（Dequantization）的时机与延迟**：
* 权重以 4-bit 形式存放在 DRAM/SRAM 中，计算时在 CPU/NPU 内部的向量寄存器/MAC 阵列前，**实时反量化（On-the-fly Dequantization）** 为 FP16/INT8 再进行点乘。
* *反量化公式*：$W_{fp16} = \text{Scale} \times W_{int4} + \text{ZeroPoint}$。
* *硬件/算子调优*：将“反量化 + 矩阵乘（GEMM）”融合写成 **Fused Kernel（融合算子）**，避免反量化后的中间结果写回内存，消除访存开销。



### 2. 结构化剪枝（Structured Pruning）与硬件友好度：

* **非结构化剪枝 vs 结构化剪枝**：
* *非结构化剪枝*：随机把权重设为 0。**对硬件极其不友好**，产生稀疏矩阵，导致 GPU/NPU 的 SIMD/Tensor 单元大量等待与分支预测失败，内存访问不连续，**实际延迟反而恶化**。
* *结构化剪枝*：按 Channel / Head / Layer 整体裁剪（如 N:M 稀疏性，如 2:4 稀疏）。


* **2:4 稀疏性（Sparse Tensor Core）**：
* 连续 4 个权重中恰好有 2 个为 0。硬件直接跳过零值的乘法，**计算吞吐量直接翻倍（2x Speedup）**，且内存只存储非零权重和 2-bit 索引。



---

## 7.3 【查漏补缺】二面高频补充八股与实战考点


---

### 1. 格雷码（Gray Code）与二进制转换（Hand-written Verilog）

#### (a) 二进制转格雷码（Binary to Gray）

* **原理**：$G[i] = B[i] \oplus B[i+1]$（最高位保持不变）。
* **Verilog 实现**：
```systemverilog
assign gray = (bin >> 1) ^ bin;

```



#### (b) 格雷码转二进制（Gray to Binary）

* **原理**：二进制的最高位等于格雷码最高位，后续每一位是当前格雷码位与前一位已计算出的二进制位的异或（级联 XOR）。
* **Verilog 实现（可综合）**：
```systemverilog
always_comb begin
    bin[WIDTH-1] = gray[WIDTH-1];
    for (int i = WIDTH-2; i >= 0; i--) begin
        bin[i] = bin[i+1] ^ gray[i];
    end
end

```



---

### 2. 门控时钟（Integrated Clock Gating - ICG）与 Glance

* **原理**：为了消除寄存器在不更新数据时的动态功耗，使用 **ICG 单元** 切断时钟。
* **为什么不能直接用 `clk & en` 门控？**
* 如果 `en` 信号在 `clk` 为高电平时发生变化（产生毛刺），会导致输出的时钟产生**严重毛刺（Glitch）**，引发后续时序逻辑误动作！


* **标准 ICG 单元结构**：
* 由 **低电平锁存器（Negative-edge Latch） + 与门（AND Gate）** 组成。
* `en` 信号首先在 `clk` 低电平时被 Latch 锁存住，确保只有当 `clk` 完全处于低电平时，门控开关才切换，从而**输出绝对无毛刺的时钟**。


* **设计规范**：在 RTL 中**绝不手写组合逻辑门控制时钟**，而是通过编写带 `if(en)` 条件的 `always_ff`，让综合工具自动插入库里的标准 ICG 单元（如 Synopsys `compile_ultra -gate_clock`）。

---

### 3. 浮点数加法与定点化（Fixed-point Conversion）

* **浮点数加法器的 5 个步骤（面试必背流程）**：
1. **对阶（Align Exponents）**：对比两个数的阶码（Exponent），将阶码较小的数的尾数（Mantissa）向右移位，使两数阶码看齐（对齐到大阶码）。
2. **尾数相加/减（Add/Subtract Mantissas）**：根据符号位，对对齐后的尾数进行定点加/减计算。
3. **结果规格化（Normalize）**：如果尾数溢出则右移并增加阶码；如果最高有效位不是 1，则左移（Left Shift）并减少阶码，保持 $1.xxxx$ 格式。
4. **舍入（Rounding）**：按 IEEE 754 标准（如 就近舍入 Round to Nearest）处理多余的尾数位。
5. **溢出检查（Overflow / Underflow Check）**：检查阶码是否超过最大/最小值（NaN / Inf / Denormalized）。



---

## 7.4 【速查】二面高频追问补充表

| 追问 | 一句话答 |
| --- | --- |
| **为什么 MAC 需要 Booth 编码？** | 将乘数分组重编码，直接把**部分积（Partial Products）数量砍半**，大幅降低加法器树的面积与组合延迟。 |
| **Wallace Tree 是怎么工作的？** | 用 3:2 Compressor（全加器）和 2:2 Compressor 并行压缩部分积，把 $O(N)$ 的加法链延迟降低到 $O(\log N)$。 |
| **MAC 累加器的 RAW 冲突怎么处理？** | 硬件增加 Accumulator Bypass（前递旁路）或者将多通道计算进行时间交错（Interleaving）。 |
| **4-bit 权重量化为什么能降端侧延迟？** | 端侧 LLM 是访存受限（Memory-Bound）任务，压缩权重大幅降低 DDR 带宽压力，减少等数据的停顿。 |
| **非结构化剪枝为什么对硬件不友好？** | 产生不规则稀疏矩阵，导致 SIMD/Tensor 单元大量等待、分支预测失效及内存非连续访问，实际延迟恶化。 |
| **为什么必须“异步复位，同步释放”？** | 异步复位生效快；同步释放可避免复位信号撤销时撞上时钟沿，从而引发 Recovery/Removal 违例和亚稳态。 |
| **为什么不能直接用 `assign gated_clk = clk & en`？** | `en` 信号在 `clk` 高电平变化时会引发**时钟毛刺（Glitch）**；必须用带 Low-latch 的标准 ICG Cell。 |
| **浮点加法器的 5 个步骤？** | **对阶**（小阶向大阶对齐） $\rightarrow$ **尾数加减** $\rightarrow$ **规格化**（移位恢复 $1.x$ 格式） $\rightarrow$ **舍入** $\rightarrow$ **溢出检查**。 |