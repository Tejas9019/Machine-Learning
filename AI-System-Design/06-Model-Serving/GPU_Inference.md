# GPU Inference: Hardware & Memory Mechanics

## Overview

Deploying deep learning models—particularly Large Language Models (LLMs)—requires a deep understanding of GPU hardware execution. Unlike CPU execution, which focuses on low-latency sequential instruction execution, GPU execution excels in mass parallel mathematical processing. This guide explores the physical memory hierarchy of GPUs, the distinction between compute-bound and memory-bound operations, and the mathematical constraints of hosting LLMs at scale.

---

## GPU Memory Architecture & Performance Bounds

To optimize inference, one must understand how data moves within the physical chip:

- **High Bandwidth Memory (HBM/VRAM)**: The main off-chip memory bank on the GPU board (e.g., 80 GB on an Nvidia H100). HBM has high latency compared to on-chip memory but provides massive capacity to hold model weights, KV caches, and intermediate tensors.
- **SRAM (L1/L2 Cache & Shared Memory)**: Extremely fast on-chip memory located directly next to the GPU's streaming multiprocessors (SMs). Data must reside in SRAM/Registers to be computed on by Tensor Cores.
- **Memory Bandwidth Bottleneck**: The physical connection between HBM and SRAM is the primary rate-limiting factor for many deep learning workloads. The rate of data transfer is measured in Gigabytes/second (GB/s).
- **Arithmetic Intensity**: Defined as the ratio of floating-point operations (FLOPs) performed per byte of memory read/written:
  $$\text{Arithmetic Intensity} = \frac{\text{FLOPs}}{\text{Bytes Transferred}}$$

### Roofline Model: Compute-Bound vs. Memory-Bound

Workloads are classified into two execution regimes based on their arithmetic intensity relative to the GPU's hardware limits:

1. **Compute-Bound Regime**:
   - Occurs when arithmetic intensity is high. The processor spends more time computing than waiting for memory.
   - Performance is limited by the GPU's peak computational capability (TFLOPS).
   - *Example*: Large matrix-matrix multiplications ($GEMM$) during model pre-training or prompt prefill.
2. **Memory-Bound Regime**:
   - Occurs when arithmetic intensity is low. The processor spends most of its time waiting for weights or tokens to load from HBM to SRAM.
   - Performance is limited by the GPU's memory bandwidth (GB/s).
   - *Example*: Autoregressive token decoding, where weight matrices must be loaded from HBM to generate a single token.

---

## Prefill vs. Decode Phases

LLM inference is split into two distinct phases with different computational characteristics:

```
                          [User Input Prompt]
                                  │
                                  ▼
                        ┌──────────────────┐
                        │  Prefill Phase   │ (Compute-Bound, Parallel Token Processing)
                        └──────────────────┘
                                  │
                                  ▼
                        ┌──────────────────┐
                        │   Decode Phase   │ (Memory-Bound, Autoregressive Token Generation)
                        └──────────────────┘
```

### 1. Prefill Phase
- **Operation**: Processes the entire input prompt at once.
- **Mechanics**: Computes the Key-Value (KV) values for all input prompt tokens concurrently. This involves massive Matrix-Matrix multiplications ($GEMM$).
- **Bottleneck**: **Compute-bound**. The GPU can parallelize operations across all tokens, maximizing Tensor Core utilization.

### 2. Decode Phase
- **Operation**: Generates output tokens one by one autoregressively.
- **Mechanics**: To generate token $t$, the system must read the entire model weights matrix and all previously computed KV values (KV Cache) from HBM to SRAM. This represents a Matrix-Vector multiplication ($GEMV$).
- **Bottleneck**: **Memory-bound**. For every single token generated, gigabytes of model weights must travel from HBM to SRAM, leaving the Tensor Cores idle most of the time.

---

## KV Cache Memory Consumption Math

To avoid recomputing Key-Value matrices for previous tokens at each decoding step, we store them in VRAM as the **KV Cache**. The KV cache size grows linearly with the context length and batch size.

### Memory Formula per Token
For a model running in half-precision format (FP16 or BF16 - 2 bytes per parameter):
$$\text{Memory per token (Bytes)} = 2 \times 2 \times n_{\text{layers}} \times n_{\text{heads}} \times d_{\text{head}}$$
- The first factor of $2$ accounts for storing both the **Key** and **Value** vectors.
- The second factor of $2$ accounts for the precision ($2 \text{ bytes}$ for FP16/BF16).
- $n_{\text{layers}}$: Total number of transformer layers.
- $n_{\text{heads}}$: Number of attention heads (or key-value heads if using GQA/MQA).
- $d_{\text{head}}$: Head dimension (typically $d_{\text{model}} / n_{\text{heads}}$).

### Multi-Batch KV Cache Calculation
For a serving deployment with:
- Batch Size ($B$) = 32
- Context Length ($L$) = 2048
- Llama-3-8B parameters ($n_{\text{layers}} = 32$, Grouped Query Attention key-value heads $n_{\text{kv\_heads}} = 8$, head dimension $d_{\text{head}} = 128$):

$$\text{KV Cache Memory} = B \times L \times (2 \times 2 \times n_{\text{layers}} \times n_{\text{kv\_heads}} \times d_{\text{head}})$$
$$\text{KV Cache Memory} = 32 \times 2048 \times (4 \times 32 \times 8 \times 128) \text{ Bytes}$$
$$\text{KV Cache Memory} = 65,536 \times 131,072 \text{ Bytes} \approx 8.59 \text{ GB}$$
If using multi-head attention (where $n_{\text{kv\_heads}} = 32$ instead of 8), the memory would quadruple to **$34.36 \text{ GB}$**, highlighting why Grouped Query Attention (GQA) is crucial for scaling serving capacity.

---

## Distributed Inference Parallelism

When a model is too large to fit in a single GPU's VRAM, it must be partitioned using distributed parallelism:

### 1. Tensor Parallelism (TP)
- **Mechanism**: Splits individual weight matrices across multiple GPUs (e.g., Megatron-LM style).
- **Execution**:
  - The Self-Attention projection matrices ($W_Q, W_K, W_V$) are split column-wise.
  - The Output Projection matrix ($W_O$) is split row-wise.
- **Communication**: Requires two high-bandwidth synchronization steps (`All-Reduce`) per transformer layer over physical NVLink bridges.
- **Best Use Case**: Split-second low-latency within a single multi-GPU node (intra-node).

### 2. Pipeline Parallelism (PP)
- **Mechanism**: Splits layers sequentially across multiple GPUs (e.g., GPU 0 runs layers 1-8, GPU 1 runs layers 9-16).
- **Execution**: Activations from the end of one GPU stage are sent via network links to the next GPU.
- **Bubble Overhead**: GPUs can sit idle ("bubbles") waiting for preceding stages to finish. To mitigate this, 1F1B (One Forward, One Backward) or micro-batch scheduling is applied.
- **Best Use Case**: Scaling massive models across separate server enclosures (inter-node).

---

## Interview Questions

### Q1: Why is LLM text generation memory-bandwidth bound? Walk through the math.
**Answer**:
1. During the decode phase, to generate a single token, we perform a Matrix-Vector multiplication between the input token vector $x$ (shape $1 \times d$) and the model weights matrix $W$ (shape $d \times d$).
2. Let's assume a 70B parameter model ($70 \times 10^9$ weights). In 16-bit precision, the weight file size is $140 \text{ GB}$.
3. To compute the output state, we must load all $140 \text{ GB}$ of weights from HBM to SRAM.
4. On an Nvidia A100 GPU, the peak memory bandwidth is $2 \text{ TB/s}$ ($2000 \text{ GB/s}$).
5. Therefore, the absolute minimum time required to read the weights for one step is:
   $$\text{Time} = \frac{140 \text{ GB}}{2000 \text{ GB/s}} = 70 \text{ ms}$$
6. This places an upper limit on decoding speed of $\approx 14 \text{ tokens/second}$ per user request on a single A100 GPU, regardless of how many Tensor Cores or FLOPs the GPU has, because the cores spend most of their time idle, waiting for the weight bytes to arrive.

### Q2: What is Grouped Query Attention (GQA) and how does it optimize KV Cache memory?
**Answer**:
1. In standard **Multi-Head Attention (MHA)**, every Query head ($Q$) has a corresponding Key ($K$) and Value ($V$) head.
2. In **Grouped Query Attention (GQA)**, query heads are grouped, and each group shares a single Key and Value head (e.g., a 4:1 ratio of query-to-KV heads).
3. This reduces the number of Key and Value projections by a factor of 4.
4. Because the KV Cache only stores Key and Value states, GQA reduces the memory footprint of the KV Cache by $4\times$. This allows serving deployments to run larger batch sizes ($B$) or support longer context windows ($L$) before hitting VRAM limits.

---

## References

1. **Roofline Model**: Williams, S., Waterman, A., & Patterson, D. (2009). *Roofline: an insightful visual performance model for multicore architectures*. Communications of the ACM.
2. **Megatron-LM Tensor Parallelism**: Shoeybi, M., et al. (2019). *Megatron-LM: Training Multi-Gillion Parameter Language Models Using Model Parallelism*. arXiv preprint.
3. **Grouped Query Attention**: Ainslie, J., et al. (2023). *GQA: Training Generalized Multi-Query Transformer Models from Multi-Head Checkpoints*. EMNLP.
