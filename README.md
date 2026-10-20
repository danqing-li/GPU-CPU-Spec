# GPU-CPU-GPGPUs-Spec

Roofline, Computational intensity, FLOPs, Memory Bound,

### Finding Computational Intensity for a Specific Chip

Short answer: you usually can't look up "computational intensity" as a chip spec — it's a property of the *kernel/workload*, not the chip.** What you *can* find per-chip is the two quantities that define the machine's **ridge point** (the minimum arithmetic intensity needed to be compute-bound):

$$I_{\text{ridge}} = \frac{\text{Peak FLOPS}}{\text{Peak Memory Bandwidth}} \quad [\text{FLOP/byte}]$$

That's the closest standard per-chip number to what you're describing.

### 1. Peak FLOPS

From the vendor datasheet or:

- **NVIDIA**: CUDA C Programming Guide, "Arithmetic Instructions" table (per-SM FLOP/clock × SMs × clock); or the GPU specs page
- **AMD**: ROCm / AMD Instinct datasheets
- **CPU**: Intel ARK / AMD product pages, or `lscpu` + core count × FMA width × 2 × clock

> ⚠️ Beware: distinguish FP32/FP64/tensor-core numbers, and "with/without sparsity".

### 2. Peak memory bandwidth

Also from the datasheet (GB/s), e.g. HBM stack count × per-stack bandwidth, or DDR channels × speed × width. This is the denominator you need for bytes/s.

Then compute the ridge point yourself: e.g. an **H100 SXM** (FP64 ~34 TFLOPS, ~3.35 TB/s) gives ~**10 FLOP/byte**; a **B200-class** part lands in a similar few-to-tens range depending on precision.

## If you want measured, not theoretical

- **Empirical roofline**: run STREAM (or `bandwidthTest`) for achievable bandwidth and a LINPACK/HPL run for achievable FLOPS, then divide.
- **Per-kernel intensity**: use Nsight Compute (`ncu`) — it reports "Arithmetic Intensity" and "Compute/Memory throughput" per kernel directly, and plots the roofline. This is the correct tool if you actually want *I* for a workload on that chip.

## Caveats

- The "FLOPS/Bytes" ratio is only meaningful with **matched units and matched peaks** (both theoretical or both achievable). Mixing datasheet FLOPS with STREAM bandwidth gives an optimistic ridge point.
- For AI workloads, people often quote intensity in **MACs/byte** or **OPs/byte** (1 MAC = 2 FLOP), and use tensor-core FLOPS — check which convention a source uses.
- **Cache hierarchy matters**: intensity against DRAM vs. against L2/SRAM gives very different ridge points.

> **So:** look up FLOPS and bandwidth on the datasheet, divide → ridge-point intensity; use `ncu`/roofline if you need the actual intensity of a specific kernel.

---

# 脊点：为什么是"最小"计算强度？

好问题——直觉上"计算强度越大越该是计算受限"没错，但**脊点说的是"要进入计算受限区，强度至少要多大"**，所以它是下限（阈值），不是上限。关键在于分清"强度"是自变量、而"受限类型"是它的函数。

## 用 roofline 公式看

$$\text{可达性能} = \min(P,\; B \times I)$$

- 当 **I 很小**：$B \times I < P$，性能被内存带宽卡住 → **内存受限**，性能随 I 线性上升。
- 当 **I 很大**：$B \times I > P$，性能顶到 P 封顶 → **计算受限**，性能不再随 I 增长。

两者相等的分界点就是脊点：

$$I_{\text{ridge}} = \frac{P}{B}$$

- $I < I_{\text{ridge}}$ → 内存受限
- $I > I_{\text{ridge}}$ → 计算受限

所以"成为计算受限"的条件是 **$I \geq I_{\text{ridge}}$**，$I_{\text{ridge}}$ 自然就是这个条件的**最小**强度。强度越大越计算受限，这与你的直觉一致；只是"跨过门槛"这件事对应的是下限。

## 为什么容易觉得该是"最大"

因为"最大强度"对应的是"**最**计算受限 / 最不内存受限"的极端，而不是"**刚好进入**计算受限"的临界。脊点是**分界/临界值**，问"至少多大才够"就得到下限。

> 类比：限速 60，问"要多快才算超速"——答案是" > 60"，是下限不是上限。

## 一个容易混的点

如果你把问题反过来问：

> "给定强度 I，芯片在什么算力/带宽比下才计算受限？"

那条件变成 $P/B \leq I$，即芯片的脊点要**小于等于** I——这时 I 扮演的是上界。两种问法方向相反，但都在说同一件事：

- **脊点低**（算力相对带宽富余）的芯片更容易计算受限；
- **强度高**的 kernel 更容易计算受限。

> **记住一句话即可：**
> - **脊点是芯片属性（P/B）**，是"进入计算受限所需的**最小**强度"；
> - **强度是 kernel 属性**，越大越偏计算受限。
