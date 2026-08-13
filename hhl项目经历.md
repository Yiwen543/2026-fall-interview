### 项目① 开芯院（BOSC）SPM 实习 —— IP 设计工程师(当前)
> 一句话:面向 RISC-V(香山)系统,用 Chisel/Scala 设计一个 **bank-conflict-free 的 Scratchpad Memory（SPM）** 模块。

怎么讲:"我在开芯院做数字前端 IP 设计,主要负责一个面向 RISC-V(香山)系统的便签存储器 SPM 模块,用 Chisel/Scala 开发。这个 SPM 的核心目标是**消除 bank 冲突**——通过 XOR 地址扰动 + DataShuffle 蝶形网络做对角线存储,让向量/矩阵访问能并行打到不同 bank。我负责 SRAM 接口、bank 仲裁(按优先级 Matrix=Vector > DMA=Noc,DMA/Noc 之间用 round-robin)和 ECC(SECDED,1 位纠错 / 2 位检错)的微架构设计,把规格转成可综合 RTL,并写设计文档。"

可追问:**为什么会有 bank 冲突 / DataShuffle 怎么实现(见 5.7,能手写代码)**、SECDED 原理、**为什么用 Chisel(见 1.3)**、round-robin(见 2.10)。

### 项目③ BERT 硬件加速器与 SoC 集成(Columbia)

> 一句话:用 HLS 开发可综合 SystemC 的 **BERT 加速器**,集成进基于 ESP 的 SoC,和 Ariane RISC-V 核 + 2D-mesh NoC 协同,DSE 把推理延迟降 35%,VC707 上占约 60% LUT。

**BERT 层在整个 AI 加速器里做什么(这道一定背):**
BERT 是 Transformer 的 **encoder**,做的是 NLP 里"理解一段文本/句子的语义表示"。它的计算可拆成几类算子,我的加速器加速的就是这些计算密集的部分:
- **Q/K/V 投影**:输入向量分别乘三个权重矩阵,得到 Query、Key、Value —— 本质是矩阵乘(GEMM)。
- **Self-Attention**:算 `QKᵀ` 得到注意力分数 → softmax 归一化 → 再乘 V 加权求和。核心是两次大矩阵乘 + 一个 softmax。
- **Feed-Forward Network（FFN）**:两层全连接(线性层),中间一个激活 —— 还是 GEMM。
- 外加 **LayerNorm**、残差连接。

一句话回答:"**BERT 加速器加速的核心是 Transformer encoder 里的矩阵乘(attention 的 QKᵀ、softmax 加权 V,以及 FFN 的两层线性),外加 softmax、layernorm 这些向量算子。** 加速器作为一个 accelerator tile 专跑这些 GEMM 密集计算,RISC-V 核负责控制和调度,数据通过 NoC 的 DMA 平面在主存和加速器之间搬运。"

怎么讲(系统层面):"我用 HLS 把 BERT 加速器写成可综合的 SystemC,实现 memory-mapped AXI4 接口(最高 512-bit)喂权重;集成进 ESP SoC,和 Ariane RISC-V 核、2D-mesh NoC 协同做软硬件协同设计。然后做系统级 DSE,把最热的 GEMM 内层循环优化到 II=1、把 DMA 位宽调到和 NoC 平面等宽,整体推理延迟比基线降 35%,VC707 上占约 60% LUT。"

可追问:见 DSE(2.11)、AXI(2.9);为什么 35% 不是更多 → Amdahl(系统级还含不可流水的数据搬运)。

## 1.3 为什么用 Chisel?Chisel 相比 SystemVerilog 的好处【新增】

**先给关系**:Chisel 不是"另一种 Verilog",它是一套**内嵌在 Scala 里的硬件构造 DSL**。你写的是 Scala 程序,运行(elaboration,精细化)后**生成 Verilog/SystemVerilog**,再交给下游综合工具。所以 Chisel 和 SV 不是竞争关系,而是**"高层生成器 → 低层 RTL"**的关系。

**关键区分(这点讲出来很加分):要分清"Scala 世界"和"硬件世界"**
- **Scala 世界**(`Int`、`Boolean`、`if/else`、`for` 循环):在**生成时**就被求值掉,用来**参数化和控制怎么生成电路**,本身不产生硬件。
- **硬件世界**(`UInt`、`Bool`、`when/otherwise`、`Reg`、`Wire`):这些才真正**对应生成的 Verilog**。
- 例:`for (i <- 0 until n)` 是 Scala 循环,生成时**展开**成 n 份硬件;这和 SV 的 `generate for` 类似,但 Chisel 有整个 Scala 语言的能力来控制。

**Chisel 相比 SV 的好处:**
1. **强大的参数化 / 生成能力**:用 Scala 的类型、函数、面向对象和函数式特性,可以写**高度可复用、可配置的硬件生成器**。比如一个 SPM 模块,bank 数、位宽、深度全用参数,换配置只改参数不改 RTL——香山这种大项目正是靠这个批量生成不同配置的模块。
2. **类型安全,编译期抓错**:Chisel 基于 Scala 强类型系统,很多在 SV 里要仿真才暴露的低级错误(位宽不匹配、连接错误)在 **elaboration 时就报错**,提前拦截。
3. **更高抽象、代码更短**:像 `Decoupled`(自带 ready/valid 的握手接口)、`Bundle`(结构化接口)这些库,把常见模式封装好,写起来比 SV 手搓省很多样板代码,也更不容易出错。
4. **面向对象复用**:模块可以继承、组合,用 Scala 的 trait 混入公共逻辑,复用粒度比 SV 的 module 实例化更灵活。
5. **生成的仍是标准 Verilog**:下游综合、时序、后端流程完全不变,不绑定特殊工具。

**Chisel 的代价 / 什么时候还是 SV(诚实一点):**
- **学习曲线**:要会 Scala,团队门槛高。
- **调试**:出问题时要在"生成的 Verilog"和"Chisel 源码"之间对应,不如直接写 SV 直观;生成的 Verilog 可读性一般。
- **精细控制**:要对某几个门做极致手工优化时,直接写 SV 更可控。
- **验证生态**:UVM 等成熟验证方法学还是 SV 的主场。

一句话:"**Chisel 是内嵌在 Scala 里的硬件生成器,elaboration 后产出标准 Verilog。** 相比 SV,它的最大优势是**强参数化和类型安全**——用 Scala 的能力写高度可复用、可配置的硬件,编译期就抓位宽/连接错误,像香山这种大项目能靠它批量生成不同配置的模块。代价是要会 Scala、调试要在生成的 Verilog 和源码间对应。核心心智模型是分清 **Scala 世界(生成时求值、控制怎么生成)** 和 **硬件世界(UInt/Reg/when,真正对应 Verilog)**。"


## 2.11 设计空间探索 DSE(BERT)

- **一句话**:系统遍历 HLS 编译指令配置,在 FPGA 资源预算下找**延迟–面积帕累托最优**。
- **4 个旋钮**:① 循环流水 II 目标(II=1 每周期启动一次新迭代);② unroll 展开;③ array partition(拆多 bank 凑读端口,是 II=1 前提);④ 接口位宽 / AXI 突发。
- **方法**:每组配置看 HLS 报告的延迟周期 + LUT/DSP/BRAM → 画帕累托前沿 → 挑满足时序又在预算内的点。
- **BERT 故事线**:最热 kernel = 矩阵乘 MAC → 起初 II≠1,瓶颈是双端口 RAM 端口不够 → partition 拆 bank 凑读端口 → 压到 II=1;同时 DMA 位宽 512b = NoC 平面宽,消除 I/O 瓶颈 → 延迟降 35%,占约 60% LUT。
- **为何只 35%(Amdahl 防守)**:kernel 级快很多,但系统级含不可流水的数据搬运,所以同时做带宽匹配。
- **达不到 II=1 的原因**:RAW 循环携带依赖,或存储端口冲突;解法 partition / 把累加改部分和、树形归约打断依赖。


---

## 5.7 【新增】SPM 的 DataShuffle 逻辑(简化 SystemVerilog 展示)

> 这是你实习项目里的核心模块,被问到"你那个消除 bank 冲突具体怎么做的"时,能手写出这段就非常加分。

### 第一步:先讲清为什么会有 bank 冲突

假设 SPM 有 8 个 bank,数据按行主序、最朴素地映射:`bank = col % 8`。

```
        col0  col1  col2  col3 ...           bank 编号
row0  [ b0    b1    b2    b3  ... ]
row1  [ b0    b1    b2    b3  ... ]     ← 每行都一样
row2  [ b0    b1    b2    b3  ... ]
```

- **按行访问**(同一行、连续列):col 递增 → 落在不同 bank → **可以并行** ✅
- **按列访问**(同一列、不同行):col 固定 → **全部落在同一个 bank** → 只能串行,8 倍延迟 ❌

而 AI 加速器里矩阵运算**行、列两种访问都要用**(比如矩阵转置、`QKᵀ`),所以列访问的冲突必须解决。

### 第二步:XOR 扰动的 bank 映射

把映射改成:**`bank = col ^ row`**(bank 数是 2 的幂时用 XOR 最省;也可用 `(col+row) % N`)

```
        col0  col1  col2  col3 ...
row0  [ b0    b1    b2    b3  ... ]
row1  [ b1    b0    b3    b2  ... ]     ← 每行错开(对角线存储)
row2  [ b2    b3    b0    b1  ... ]
row3  [ b3    b2    b1    b0  ... ]
```

现在**按列访问**(col 固定、row 变):`bank = col ^ row` 随 row 变化 → 又散开到不同 bank → **并行恢复** ✅ 而按行访问依然是散开的,两种都不冲突。

**DataShuffle 网络的作用**:数据在计算侧是**逻辑顺序**排列的,但要按上面的 XOR 映射摆进**物理 bank**,中间就需要一个硬件把 32 路数据重新排列 —— 这就是 DataShuffle。

### 第三步:用蝶形(butterfly)网络实现,而不是大 crossbar

要实现 `dout[i] = din[i ^ mode]`,直接做 32×32 全交叉开关的话面积和延迟都爆炸。**利用 XOR 可以逐位拆解的性质**,拆成 `log₂(32)=5` 级,每级只处理 mode 的一位:

```systemverilog
module data_shuffle #(
  parameter int NUM_LANE = 32,                 // 通道数(=bank 数),必须是 2 的幂
  parameter int DATA_W   = 64,                 // 每通道位宽
  parameter int SEL_W    = $clog2(NUM_LANE)    // = 5,控制信号位宽
)(
  input  logic [DATA_W-1:0] din  [NUM_LANE],   // 逻辑顺序的数据
  input  logic [SEL_W-1:0]  mode,              // 扰动量(通常来自行地址 row)
  output logic [DATA_W-1:0] dout [NUM_LANE]    // 打散后送往各 bank
);
  // 目标:dout[i] = din[i ^ mode]
  // 拆成 log2(N) 级蝶形网络,第 k 级只看 mode[k]:
  //   mode[k]=1 → 把"索引第 k 位不同"的两路互换
  //   mode[k]=0 → 直通
  // 各级叠加起来,净效果正好等于按 mode 做一次 XOR 置换

  logic [DATA_W-1:0] stage [SEL_W+1][NUM_LANE];

  assign stage[0] = din;                       // 第 0 级输入 = 原始数据

  genvar k, i;
  generate
    for (k = 0; k < SEL_W; k++) begin : g_stage      // 共 5 级
      for (i = 0; i < NUM_LANE; i++) begin : g_lane  // 每级 32 路 2:1 mux
        assign stage[k+1][i] = mode[k] ? stage[k][i ^ (1 << k)]  // 交换
                                       : stage[k][i];             // 直通
      end
    end
  endgenerate

  assign dout = stage[SEL_W];                  // 最后一级输出
endmodule
```

**逐级在干什么:**

| 级 | 控制位 | 动作 | 交换距离 |
|---|---|---|---|
| 0 | `mode[0]` | `dout[i] = din[i^1]` | 相邻两路互换 |
| 1 | `mode[1]` | `dout[i] = din[i^2]` | 隔 2 路互换 |
| 2 | `mode[2]` | `dout[i] = din[i^4]` | 隔 4 路互换 |
| 3 | `mode[3]` | `dout[i] = din[i^8]` | 隔 8 路互换 |
| 4 | `mode[4]` | `dout[i] = din[i^16]` | 前后 16 路整体对调 |

### 两个高分点(一定要讲)

**① 为什么用蝶形网络而不是 crossbar:**
- 全交叉开关:面积 **O(N²)**(32×32=1024 个开关),延迟大、布线拥塞。
- 蝶形网络:只要 **log₂N = 5 级**、每级 N 个 2:1 mux → 面积 **O(N·logN)**、延迟 **O(logN)**。这就是为什么关键路径能压得住。

**② XOR 是自逆的(self-inverse)—— 读写可以复用同一个网络:**
```
(i ^ mode) ^ mode = i
```
写进去时用 `mode` 打散,读回来时**用同一个模块、同一个 mode 再过一遍就自动还原**了,不需要另做一个"反向网络"。**面积直接省一半**,这是 XOR 方案相比 `(col+row)%N`(需要单独的减法还原)的一大优势。

**一句话(现场说):**"最朴素的 `bank=col%N` 映射下,按列访问会全部撞进同一个 bank、只能串行。我们改成 **`bank = col ^ row` 的 XOR 扰动映射**做对角线存储,让行访问和列访问都能散开到不同 bank。**DataShuffle 就是把逻辑顺序的数据搬到这个物理 bank 顺序的硬件**,实现的是 `dout[i] = din[i ^ mode]`。它不用大 crossbar,而是拆成 **log₂N 级蝶形网络**,每级只看 mode 的一位、决定"索引该位不同的两路要不要互换",面积从 O(N²) 降到 O(N·logN)、延迟只有 logN 级。而且 **XOR 自逆**,读回来用同一个网络同一个 mode 就能还原,读写路径复用同一个模块。"

> 📌 对应你项目的真实参数:`DataShuffle.sv` 是 **32 路 × 64-bit**,控制信号 `io_mode[4:0]`,正好 **5 级**蝶形(stage0 换 `i^1`、stage1 换 `i^2`、…、stage4 换 `i^16`),输出取最后一级。
