# GPU 与显存带宽硬件知识速成

> 目标读者：从 Agent 开发转大模型算法，需要快速补齐硬件认知短板。
> 不讲历史，不讲游戏卡，只讲训练和推理场景下你真正需要懂的东西。

---

## 1. 核心概念：计算瓶颈 vs 带宽瓶颈

大模型workload只有两种瓶颈：

| 瓶颈类型 | 含义 | 典型场景 |
|---------|------|---------|
| Compute-bound | 算力不够，ALU忙不过来 | 大batch训练、prefill阶段 |
| Memory-bound | 数据搬运跟不上计算速度 | decode阶段、小batch推理 |

### 到底什么是 Compute-bound 和 Memory-bound？

GPU 干活分两步：**搬数据**（从显存HBM搬到计算单元）和 **算数据**（Tensor Core/ALU做矩阵乘法等）。这两步是流水线式并行的，但总有一个更慢，更慢的那个就是瓶颈。

**Compute-bound（计算瓶颈）：**
- 数据已经搬到计算单元了，但计算单元算不过来，数据在排队等着被处理
- 类比：食材已经全摆在案板上了，但厨师切菜炒菜的速度跟不上
- 特征：GPU利用率很高（算力打满），加更强的GPU（更多CUDA核/更快的Tensor Core）能直接提速
- 典型场景：大batch的矩阵乘法、prefill阶段（一次处理整个prompt，矩阵很大，计算量巨大）

**Memory-bound（带宽瓶颈）：**
- 计算单元很闲，在空转等数据从HBM搬过来
- 类比：厨师刀工很快，但每次要跑去冰箱拿一样食材再跑回来，大部分时间花在跑腿上
- 特征：GPU算力利用率低（可能只用了10-20%），加更强的GPU没用，需要更大的显存带宽或减少数据搬运量
- 典型场景：decode阶段（每生成一个token，要把整个模型权重从HBM读一遍，但每个权重只做一次乘加）

**用数字说话：**

```
A100 80GB:
- FP16 算力: 312 TFLOPS（每秒 312 万亿次浮点运算）
- HBM 带宽: 2 TB/s（每秒搬 2 万亿字节）
- ops:byte 比值 = 312T / 2T = 156 FLOPs/Byte

含义：每从HBM搬1字节数据，GPU能做156次浮点运算。
如果你的任务每搬1字节只需要做1-2次运算（比如decode），
那GPU 99%的时间在等数据，算力完全浪费。
```

**为什么 prefill 是 compute-bound？用具体数字算一遍：**

Prefill 阶段一次性把整个 prompt（比如 2048 个 token）喂进去。对于每一层的权重矩阵 W，GPU 需要做矩阵乘法：

```
Prefill（2048 个 token）:

输入矩阵 X: [2048 × 4096]  （2048个token，每个4096维）
权重矩阵 W: [4096 × 4096]  （从HBM搬一次）
输出矩阵 O: [2048 × 4096]  （结果写回HBM）
计算: X × W = O

搬运量（完整）:
  读 W = 4096 × 4096 × 2 bytes = 32 MB   ← 权重（固定开销，不随token数变化）
  读 X = 2048 × 4096 × 2 bytes = 16 MB   ← 输入
  写 O = 2048 × 4096 × 2 bytes = 16 MB   ← 输出
  总搬运量 = 32 + 16 + 16 = 64 MB

计算量: 2048 × 4096 × 4096 × 2 = 68.7 GFLOPs
算术强度 = 68.7G / 64M ≈ 1074 FLOPs/Byte

A100 的 ops:byte 临界点 = 312 TFLOPS / 2 TB/s = 156 FLOPs/Byte
1074 >> 156 → 远超临界点 → compute-bound ✓

注：很多资料只算 W 的搬运量（得到算术强度≈2048），这是因为分析瓶颈类型时
W 是决定性因素 — W 的大小不随 token 数变化，而 X 和 O 虽然也要搬运，
但它们的搬运量和计算量同步线性增长，不改变 bound 类型的判断。
在 decode 场景下 X 只有 8KB 而 W 有 32MB，X 更是可以直接忽略。
```

**为什么只算一层的权重矩阵，不算整个模型 32 层？**

因为 GPU 是一层一层串行执行的，每一层都是一个独立的"搬运→计算→写回"周期：

```
第 1层: 搬 W1(32MB) → 算 X×W1 → 写回结果
第 2层: 搬 W2(32MB) → 算 X×W2 → 写回结果
...
第32层: 搬 W32(32MB) → 算 X×W32 → 写回结果
```

判断 bound 类型的粒度是"单次矩阵乘法"，不是整个前向传播。每一层的矩阵尺寸相同，
算术强度也相同，所以算一层就够了。

如果你非要算整个模型，结果也一样：

```
LLaMA-7B 完整前向（32层，每层约8个权重矩阵: Q/K/V/O + FFN up/gate/down 等）:

总搬运量 = 32层 × 8矩阵 × 64MB = 16,384 MB ≈ 16 GB（≈整个模型参数量）
总计算量 = 32层 × 8矩阵 × 68.7 GFLOPs = 17,587 GFLOPs ≈ 17.6 TFLOPs

算术强度 = 17.6T / 16G ≈ 1074 FLOPs/Byte  ← 和单层算出来一模一样
```

原因很简单：每层结构相同，乘以层数后分子分母同比例放大，比值不变。
所以算一层就是算全部，没必要乘 32。

**为什么 decode 是 memory-bound？同样算一遍：**

```
Decode（1 个 token，batch=1）:

输入向量 x: [1 × 4096]  （只有当前1个token）
权重矩阵 W: [4096 × 4096]  （同样从HBM搬一次，和prefill搬的量一样！）
计算: x × W = [1 × 4096]

搬运量: 读W一次 = 32 MB（和prefill完全一样）
计算量: 1 × 4096 × 4096 × 2 = 33.5 MFLOPs
算术强度 = 33.5M / 32M ≈ 1 FLOP/Byte

1 << 156 → 远低于临界点 → memory-bound ✓
```

**本质区别：权重搬运量是固定的（W多大就搬多少），但计算量和token数成正比。**

- Prefill 有 2048 个 token 复用同一份权重 → 搬一次数据做 2048 倍的计算 → 算力跟不上 → compute-bound
- Decode 只有 1 个 token → 搬了 32MB 数据只做极少量计算 → 带宽跟不上 → memory-bound

**增大 batch size 能缓解 decode 的 memory-bound：**

```
batch=1:  算术强度 ≈ 1    → GPU算力利用率 ~0.6%
batch=32: 算术强度 ≈ 32   → GPU算力利用率 ~20%
batch=128:算术强度 ≈ 128  → 接近临界点156，利用率 ~82%
```

这就是 continuous batching 的硬件层面动机 — 把更多请求凑成大 batch，让 decode 的算术强度逼近临界点，提高 GPU 利用率。但 batch 不能无限大，因为 KV Cache 会吃显存。

这也解释了为什么推理优化有两个完全不同的方向：
- prefill 优化：减少 HBM 读写（FlashAttention 将 attention 计算搬到 SRAM，IO 减少但计算量不变）、Tensor并行分摊计算
- decode 优化：减少数据搬运（量化减小模型体积、Speculative Decoding 减少decode次数、增大batch提高算术强度）

**关键认知：LLM推理的decode阶段几乎永远是memory-bound的。** 你的GPU算力再强，显存带宽跟不上，就是在空转等数据。这就是为什么推理优化的核心是"减少显存搬运"而不是"堆更多CUDA核"。

### Roofline Model（屋顶线模型）

这是判断你的workload是compute-bound还是memory-bound的标准工具：

```
算术强度 (Arithmetic Intensity) = FLOPs / Bytes（每搬运1字节数据能做多少次计算）
```

- 算术强度低 → memory-bound（decode、attention）
- 算术强度高 → compute-bound（大矩阵乘法、prefill）

面试常问：**为什么decode是memory-bound？** 因为每个token生成时，要把整个KV Cache从HBM读一遍，但每个参数只做一次乘加，算术强度极低（约1-2 FLOPs/Byte）。

---

## 2. GPU 显存层级

从快到慢：

```
寄存器 (Register File)
  ↓  ~19 TB/s（估算，A100）
共享内存 / L1 Cache (SRAM, 每个SM ~192KB on A100)
  ↓  ~19 TB/s
L2 Cache (A100: 40MB)
  ↓  ~5 TB/s（估算）
HBM (显存主体，A100 80GB: 2 TB/s)
  ↓  
CPU 内存 (通过 PCIe/NVLink)
```

**关键数字你必须记住：**

| GPU | HBM容量 | HBM带宽 | FP16算力 | 带宽/算力比 |
|-----|---------|---------|----------|------------|
| A100 80GB | 80 GB | 2.0 TB/s | 312 TFLOPS | 6.4 B/FLOP |
| H100 SXM | 80 GB | 3.35 TB/s | 989 TFLOPS | 3.4 B/FLOP |
| H200 | 141 GB | 4.8 TB/s | 989 TFLOPS | 4.9 B/FLOP |
| B200 | 192 GB | 8.0 TB/s | 2250 TFLOPS (FP4) | — |
| MI300X (AMD) | 192 GB | 5.3 TB/s | 1300 TFLOPS | 4.1 B/FLOP |

注意 H100 相比 A100：算力涨了 3x，但带宽只涨了 1.7x。**这意味着 H100 上 memory-bound 问题更严重。** 这就是为什么 H200 的核心卖点不是算力提升，而是显存容量和带宽的提升。

---

## 3. HBM（High Bandwidth Memory）

### 什么是 HBM？

HBM 是一种 3D 堆叠的 DRAM，通过硅中介层（silicon interposer）和 GPU die 直接相连。相比传统 GDDR，HBM 的核心优势是：

- **带宽高**：多个通道并行（HBM3 有 16 个通道，每通道 64bit）
- **功耗低**：短距离传输，电压低
- **物理紧凑**：堆叠设计，省PCB面积

### HBM 代际演进

| 代际 | 单stack带宽 | 典型容量/stack | 用在哪 |
|------|-----------|--------------|-------|
| HBM2e | 460 GB/s | 16 GB | A100 |
| HBM3 | 665 GB/s | 16 GB | H100 |
| HBM3e | 1.2 TB/s | 24 GB | H200, B200 |

A100 用 5 个 HBM2e stack → 5 × 16GB = 80GB, 5 × 460 = 2.0 TB/s（理论峰值，实际略低）

### 面试会问的

Q: 为什么不直接用更大的GDDR显存？
A: GDDR带宽上限低（GDDR6X ~1 TB/s），而且功耗高、pin数多。HBM通过堆叠+宽总线在有限面积内实现高带宽。代价是成本贵、良率要求高。

---

## 4. 互联拓扑：NVLink / NVSwitch / PCIe / InfiniBand

单卡显存不够 → 多卡并行 → 卡间通信成为新瓶颈。

### 卡间互联带宽

| 互联方式 | 单向带宽 | 双向带宽 | 说明 |
|---------|---------|---------|------|
| PCIe 4.0 x16 | 32 GB/s | 64 GB/s | 最慢，CPU-GPU或跨节点 |
| PCIe 5.0 x16 | 64 GB/s | 128 GB/s | H100支持 |
| NVLink 3 (A100) | 300 GB/s | 600 GB/s | 单机8卡全互联 |
| NVLink 4 (H100) | 450 GB/s | 900 GB/s | 通过NVSwitch全互联 |
| NVLink 5 (B200) | 900 GB/s | 1800 GB/s | — |
| InfiniBand NDR | 50 GB/s | 100 GB/s | 跨节点，多卡可bonding |

### NVSwitch 的作用

没有 NVSwitch 时，8卡之间是 mesh 拓扑，不是每两张卡之间都有直连NVLink。NVSwitch 让所有卡之间都能以全速NVLink通信，等效于一个"GPU交换机"。

**DGX H100 的拓扑：** 8×H100 通过 4个NVSwitch 全互联，任意两卡之间 900 GB/s 双向。跨节点走 InfiniBand（400 Gb/s = 50 GB/s per port，8 port bonding = 400 GB/s）。

### 为什么这些数字重要？

Tensor Parallelism（TP）要求卡间高频通信（每层都要AllReduce），所以 TP 只能在 NVLink 互联的卡之间做（通常同一台机器内）。Pipeline Parallelism（PP）通信量小，可以跨节点走 InfiniBand。

```
典型并行策略分配：
- TP: 同节点内（NVLink），通常 TP=8（一台机器8卡）
- PP: 跨节点（InfiniBand），PP=节点数
- DP/ZeRO: 跨节点（InfiniBand），通信量相对可控
```

---

## 5. 显存占用计算（面试高频）

### 模型参数到底怎么构成的？

**什么是 hidden_dim？**

`hidden_dim` 就是模型内部每个 token 的向量表示维度。一个 token（比如"猫"这个字）进入模型后，
会被转换成一个固定长度的数字向量，这个向量有多长就是 hidden_dim。模型内部所有计算（attention、FFN）
都在这个维度上做，从头到尾不变。

```
"猫" → Embedding → [0.12, -0.34, 0.56, ..., 0.78]  ← 这个向量有 4096 个数
                    \_________________________________/
                              hidden_dim = 4096
```

不同规模的模型 hidden_dim 不同，模型越大维度越高，表示能力越强：

| 模型 | hidden_dim | 参数量 |
|------|-----------|--------|
| LLaMA-7B | 4096 | 6.6B |
| LLaMA-13B | 5120 | 13B |
| LLaMA-33B | 6656 | 33B |
| LLaMA-65B | 8192 | 65B |
| GPT-3 175B | 12288 | 175B |

选 2 的幂次（4096 = 2¹²）是为了对齐 GPU 的内存和计算单元，硬件执行效率更高。

**权重矩阵的形状由什么决定？**

权重矩阵的形状 = 输入维度 × 输出维度。线性变换 `y = x × W`，x 是 [1 × input_dim]，
W 就是 [input_dim × output_dim]，y 是 [1 × output_dim]。

Attention 层的 Q/K/V/O 投影都是 hidden_dim → hidden_dim 的变换，所以权重是方阵。
但 FFN 层不是方阵，它先升维再降维：

```
Attention 层（方阵，input_dim == output_dim）:
  Q 投影: [4096 × 4096]   ← hidden_dim → hidden_dim
  K 投影: [4096 × 4096]
  V 投影: [4096 × 4096]
  O 投影: [4096 × 4096]
  每个矩阵 16.8M 参数，4个合计 67.1M

FFN 层（长方形，input_dim ≠ output_dim）:

  FFN = Feed-Forward Network（前馈神经网络），是 Transformer 每一层里
  和 Attention 并列的另一个核心组件：

  每层 Transformer Block 的结构就两步：
  1. Attention — token之间互相看，捕捉"谁和谁有关系"
  2. FFN — 每个token独立过一个小型神经网络，做非线性变换

  Attention 负责"token 之间的交互"，FFN 负责"单个 token 的特征变换"。

  LLaMA 用的 SwiGLU 结构：
  FFN(x) = down( SiLU(gate(x)) × up(x) )

  x: [4096]                    ← 输入，hidden_dim 维
  gate(x) = x × W_gate         → [11008]   ← 升维到 intermediate_dim
  up(x)   = x × W_up           → [11008]   ← 升维到 intermediate_dim
  SiLU(gate(x)) × up(x)        → [11008]   ← 逐元素相乘（门控机制）
  down(...)  = ... × W_down     → [4096]    ← 降回 hidden_dim

  整个过程: 4096 → 11008 → 4096（升维 → 非线性变换 → 降维）

  为什么先升维再降维？低维空间里线性变换能力有限，升到高维空间后
  做非线性激活（SiLU），能学到更复杂的特征模式，再压回原维度传给下一层。

  gate:   [4096 × 11008]  ← hidden_dim → intermediate_dim（先升维）
  up:     [4096 × 11008]
  down:   [11008 × 4096]  ← intermediate_dim → hidden_dim（再降回来）
  每个矩阵 45.1M 参数，3个合计 135.3M

FFN 的设计思路：先把维度升上去（4096→11008），在高维空间做非线性变换，再降回来。
intermediate_dim 通常是 hidden_dim 的 ~2.7 倍（LLaMA 用 SwiGLU 结构）。
FFN 参数量占每层总参数的约 2/3（135M / 202M）。
```

以 LLaMA-7B 为例拆解完整参数构成：

```
模型总参数 = Embedding + 32层 Transformer Block + RMSNorm + LM Head

1. Embedding 层（输入）:
   vocab_size × hidden_dim = 32000 × 4096 = 131M（1.31亿）

2. 每层 Transformer Block（×32层）:
   - Q 投影: 4096 × 4096 = 16.8M
   - K 投影: 4096 × 4096 = 16.8M
   - V 投影: 4096 × 4096 = 16.8M
   - O 投影: 4096 × 4096 = 16.8M
   - FFN gate:  4096 × 11008 = 45.1M
   - FFN up:    4096 × 11008 = 45.1M
   - FFN down:  11008 × 4096 = 45.1M
   - 2个 RMSNorm: 4096 × 2 = 8K（可忽略）
   单层合计 ≈ 202M

   32层合计 = 202M × 32 = 6,464M ≈ 6.46B

3. 最终 RMSNorm: 4096 → 可忽略

4. LM Head（输出层）:
   hidden_dim × vocab_size = 4096 × 32000 = 131M
   （LLaMA 中 LM Head 和 Embedding 共享权重，所以不额外算）

总计 ≈ 131M + 6,464M = 6,595M ≈ 6.6B → 约等于 7B
```

参数分布：
- 32 层 Transformer Block 占了 ~98% 的参数（6.46B / 6.6B）
- Embedding 占 ~2%
- Norm 层参数量可忽略

所以说"模型参数 ≈ 所有 Transformer 层参数之和"在直觉上没问题，
但严格来说要加上 Embedding 和 LM Head。面试时说"7B 模型有 32 层，
每层约 200M 参数"就够了。

### 模型参数显存

```
参数量 × 每参数字节数 = 显存占用

7B 模型：
- FP32: 7B × 4 bytes = 28 GB
- FP16/BF16: 7B × 2 bytes = 14 GB
- INT8: 7B × 1 byte = 7 GB
- INT4: 7B × 0.5 byte = 3.5 GB
```

### 训练显存（混合精度，以 AdamW 为例）

```
每个参数需要存储：
- FP16 参数: 2 bytes
- FP32 主参数（master weights）: 4 bytes
- FP32 梯度: 4 bytes
- Adam 一阶动量 (m): 4 bytes
- Adam 二阶动量 (v): 4 bytes
合计: 18 bytes/param（不含激活值）

7B 模型训练: 7B × 18 = 126 GB（仅参数+优化器，不含激活值）
→ 单卡 A100 80GB 放不下，至少需要 2 卡 ZeRO-3 或 TP
```

### KV Cache 显存（推理）

```
每token每层的 KV Cache = 2(K和V各一份) × hidden_dim × 2 bytes(FP16)

LLaMA-2 7B (32层, hidden=4096):

单层单token:
  K 向量: 4096 × 2 bytes = 8 KB
  V 向量: 4096 × 2 bytes = 8 KB
  KV 合计: 16 KB

32层全部:
  每token KV Cache = 16 KB × 32层 = 512 KB

2048 tokens context = 2048 × 512 KB = 1 GB
```

这就是为什么长上下文推理时 KV Cache 会爆显存，也是 PagedAttention（vLLM）、GQA、MQA 等技术存在的原因。

---

## 6. 关键优化技术与硬件的关系

| 技术 | 解决什么硬件瓶颈 | 原理一句话 |
|------|----------------|-----------|
| FlashAttention | HBM带宽瓶颈 | 把attention计算搬到SRAM上做，减少HBM读写次数 |
| PagedAttention (vLLM) | 显存碎片/浪费 | 像OS虚拟内存一样管理KV Cache，按需分配 |
| GQA/MQA | KV Cache显存 | 多个query head共享KV head，减少KV Cache大小 |
| 量化 (GPTQ/AWQ/GGUF) | 显存容量+带宽 | 用更少bit表示权重，减少存储和搬运量 |
| Tensor Parallelism | 单卡显存不够 | 把矩阵切分到多卡，需要高速互联 |
| Pipeline Parallelism | 多卡扩展 | 按层切分到多卡，通信量小，可跨节点 |
| ZeRO (DeepSpeed) | 训练显存 | 把优化器状态/梯度/参数分片到多卡 |
| Speculative Decoding | decode延迟 | 用小模型猜多个token，大模型一次验证，提高硬件利用率 |
| Continuous Batching | GPU利用率 | 动态组batch，避免短序列等长序列 |
| Prefix Caching | 重复计算 | 缓存公共前缀的KV Cache，避免重复prefill |

### FlashAttention 为什么有效（必须能讲清楚）

标准 Attention 的问题：
```
Q × K^T → S (N×N矩阵，写入HBM)
softmax(S) → P (N×N矩阵，写入HBM)  
P × V → O (写入HBM)

三次读写 HBM，中间矩阵 O(N²) 大小
```

FlashAttention：
```
把 Q, K, V 分块 (tiling)，每块加载到 SRAM
在 SRAM 内完成 QK^T → softmax → ×V 的全部计算
只把最终结果 O 写回 HBM

HBM 读写从 O(N²) 降到 O(N)，计算量不变但 IO 大幅减少
```

---

## 7. 数据类型与精度（面试必考）

| 类型 | 位数 | 范围 | 用途 |
|------|------|------|------|
| FP32 | 32 bit | ±3.4×10³⁸ | 优化器状态、master weights |
| TF32 | 19 bit (10 bit mantissa) | 同FP32范围 | A100+ Tensor Core默认 |
| BF16 | 16 bit (8 bit exp) | 同FP32范围，精度低 | 训练主流，不易溢出 |
| FP16 | 16 bit (5 bit exp) | ±65504 | 推理常用，训练需loss scaling |
| FP8 (E4M3) | 8 bit | ±448 | H100+ 训练/推理 |
| FP8 (E5M2) | 8 bit | ±57344 | H100+ 梯度 |
| INT8 | 8 bit | -128~127 | 推理量化 |
| INT4 | 4 bit | -8~7 | 推理量化（GPTQ/AWQ） |

**BF16 vs FP16：** BF16 指数位和 FP32 一样（8 bit），所以数值范围相同，不容易上溢/下溢，训练更稳定。FP16 精度更高但范围小，训练时需要 loss scaling。现在训练基本都用 BF16。

**TF32：** 不是一个存储格式，是 A100 Tensor Core 的计算模式。输入 FP32，内部用 19 bit 精度计算（截断 mantissa 到 10 bit），输出 FP32。对用户透明，`torch.backends.cuda.matmul.allow_tf32 = True`。

---

## 8. 实际集群配置参考

### 典型训练集群

```
训练 70B 模型：
- 硬件: 64× H100 80GB (8 nodes × 8 GPUs)
- 互联: 节点内 NVLink 4 (900 GB/s), 节点间 InfiniBand NDR 400Gb/s
- 并行策略: TP=8, PP=2, DP=4 (8×2×4=64)
- 显存分配: ZeRO-1 (优化器状态分片) + TP (参数分片)
```

### 典型推理部署

```
部署 70B 模型 (FP16):
- 参数显存: 70B × 2 = 140 GB
- 单卡放不下 → 至少 2× H100 80GB (TP=2)
- 或者 INT4 量化: 70B × 0.5 = 35 GB → 单卡 A100 80GB 可放下

部署 7B 模型 (FP16):
- 参数显存: 14 GB → 单卡轻松
- 瓶颈在 KV Cache 和 decode 带宽
```

---

## 9. 面试高频问答速查

**Q: A100 和 H100 最大的区别是什么？**
A: 算力提升 3x（FP16 312→989 TFLOPS），带宽提升 1.7x（2→3.35 TB/s），新增 FP8 支持，NVLink 带宽提升 1.5x。但带宽增速跟不上算力增速，memory-bound 问题更突出。

**Q: 为什么大模型推理慢？**
A: Decode 阶段是 memory-bound 的。每生成一个 token，要从 HBM 读取全部模型参数（7B 模型 = 14GB），但只做一次前向。带宽决定了 tokens/s 的上限：`max tokens/s ≈ HBM带宽 / 模型大小`。A100 上 7B FP16 模型理论上限 ≈ 2TB/s ÷ 14GB ≈ 143 tokens/s。

**Q: 什么是 Arithmetic Intensity？为什么重要？**
A: FLOPs/Byte，衡量每搬运一字节数据做多少计算。低于 GPU 的 ops:byte 比值就是 memory-bound，高于就是 compute-bound。这决定了你该优化计算还是优化数据搬运。

**Q: 为什么 TP 不能跨节点？**
A: TP 每层都需要 AllReduce（通信量 = 2×参数量/TP），延迟敏感。NVLink 900 GB/s vs InfiniBand 50 GB/s，差 18 倍。跨节点做 TP 通信开销会吃掉并行收益。

**Q: ZeRO 三个阶段分别做什么？**
A: ZeRO-1 分片优化器状态（省 4x），ZeRO-2 再分片梯度（省 8x），ZeRO-3 再分片参数（省 ~18x 但通信量最大）。

---

## 10. 一张图总结带宽层级

```
┌─────────────────────────────────────────────┐
│              GPU Die (SM × N)                │
│  ┌─────────┐                                │
│  │Register │ ~19 TB/s                       │
│  │  File   │────→ SRAM (Shared Mem/L1)      │
│  └─────────┘     192KB per SM               │
│                      │ ~19 TB/s              │
│                      ↓                       │
│                  L2 Cache                    │
│                  40-60 MB                    │
│                      │ ~5 TB/s               │
│                      ↓                       │
│              ┌──────────────┐               │
│              │  HBM (DRAM)  │               │
│              │  80-192 GB   │               │
│              │  2-8 TB/s    │               │
│              └──────┬───────┘               │
└─────────────────────┼───────────────────────┘
                      │
          ┌───────────┼───────────┐
          │ NVLink    │ PCIe      │
          │ 900 GB/s  │ 128 GB/s  │
          ↓           ↓           ↓
      Other GPUs   CPU/RAM    Network
      (同节点)     (Host)    (InfiniBand)
                              50-400 GB/s
```

每一层带宽差一个数量级，优化的本质就是让数据尽量待在快的层级，少往慢的层级搬。FlashAttention 让数据留在 SRAM，PagedAttention 优化 HBM 使用，TP 避免走 PCIe，PP 减少跨节点通信。所有优化都是在和这个带宽层级做斗争。
