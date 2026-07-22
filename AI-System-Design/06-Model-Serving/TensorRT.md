# NVIDIA TensorRT: Graph Compilation

## Overview

**NVIDIA TensorRT** is a high-performance deep learning inference library and compiler designed for NVIDIA GPUs. It takes a trained neural network model (typically in ONNX format or compiled via PyTorch) and optimizes it for execution on target hardware by restructuring the execution graph, fusing layers, selecting optimal CUDA kernels, and applying precision calibration.

---

## Problem Statement

Trained deep learning model checkpoints (e.g., PyTorch `.pt` or TensorFlow graph files) are designed for flexibility and training backpropagation, making them sub-optimal for raw inference execution:
1. **Redundant Memory I/O**: Sequential operations (e.g., a Convolution followed by an Activation followed by Batch Normalization) force the GPU to write intermediate tensors back to VRAM and read them again in the next step, wasting memory bandwidth.
2. **Sub-optimal Kernel Selection**: General-purpose matrix operations are not optimized for the specific dimensions, layer types, and batch sizes of a target GPU (e.g., compiling for an A100 vs. a T4).
3. **Graph Overhead**: Parsing and running the model graph dynamically in Python adds significant CPU execution latency to the critical path.

---

## TensorRT Compiler Optimizations

TensorRT processes the model graph and compiles it into an optimized serialization runtime called a **Plan/Engine File**:

```
[PyTorch / ONNX Model] ──> [TensorRT Parser]
                                │
                                ▼
                   [Optimization Pipeline]
                   ├─ Layer & Tensor Fusion
                   ├─ Kernel Auto-Tuning
                   └─ Precision Calibration (INT8/FP8)
                                │
                                ▼
                  [Compiled Engine (.plan file)]
```

### 1. Layer & Tensor Fusion
- **Vertical Layer Fusion**: Combines sequential operations into a single CUDA kernel. For example, a Conv2D, Bias addition, and ReLU activation are combined into one fused kernel. The GPU reads the weights once, executes all three math steps in SRAM/Registers, and writes the output back to VRAM, eliminating two VRAM write/read round trips.
- **Horizontal Layer Fusion**: Identifies parallel paths in the graph (e.g., Multi-Head Attention where Q, K, and V projections run on the same input vector) and merges them into a single wide matrix multiplication kernel, maximizing parallel GPU core utilization.

---

### 2. Kernel Auto-Tuning
- GPUs contain multiple mathematical algorithms for executing a single matrix multiplication (e.g., FFT, Winograd, Direct Convolution).
- During the compilation phase, TensorRT runs benchmark sweeps of all available algorithms on the target physical GPU using the exact tensor dimensions specified in the model config.
- The compiler selects and embeds the absolute fastest executing CUDA kernels for the target chip architecture (e.g., Hopper vs. Ada Lovelace).

---

### 3. Precision Calibration (INT8/FP8 Entropy Calibration)
- To convert a model to INT8 precision, TensorRT uses a calibration dataset to determine the dynamic range of activations.
- It uses **Kullback-Leibler (KL) Divergence** to calculate and minimize the information loss (entropy) between the original FP32 activation distribution and the quantized INT8 activation distribution, setting the scale factors ($S$) to preserve maximum model accuracy.

---

## TensorRT-LLM Extensions

**TensorRT-LLM** is a specialized framework designed to scale LLM serving:
- **Paged KV-Cache**: Combines TensorRT graph execution with vLLM-style virtual page memory management.
- **In-flight Batching**: Supports scheduling requests at the iteration level, injecting new requests during the decode phase of active queries.
- **FlashAttention & Custom Kernels**: Integrates hand-tuned CUDA kernels for Multi-Head Attention, Rotary Position Embeddings (RoPE), SwiGLU activations, and Grouped Query Attention (GQA), bypassing standard PyTorch graph tracing overhead.

---

## Design Decisions & Trade-offs

### TensorRT Engines vs. Raw PyTorch Checks

- **TensorRT Engine**:
  * *Pros*: Maximum possible inference speed (typically $2\times$ to $5\times$ PyTorch), zero Python runtime overhead (can run in pure C++ inside Triton).
  * *Cons*: Compilation is hardware-locked (a `.plan` compiled on an A100 cannot run on an H100), long compilation times (up to an hour for large models), restricted dynamic shape support (must compile with defined min/max/optimal dimension profiles).
- **Raw PyTorch Checkpoint**:
  * *Pros*: Flexible, works on any hardware, supports arbitrary dynamic shapes, instant start time (no compilation step).
  * *Cons*: Slower execution speed, higher memory overhead, Python interpreter dependency.

---

## Interview Questions

### Q1: Explain Layer Fusion in TensorRT. How does it improve GPU inference speed?
**Answer**:
1. **Concept**: Layer Fusion merges multiple adjacent mathematical layers in a neural network graph into a single execution kernel.
2. **Vertical Fusion**: In standard PyTorch, executing a Convolution layer, a Bias add, and a ReLU activation requires three separate GPU kernel launches.
   - Kernel 1 reads input from VRAM, performs Convolution, and writes intermediate output to VRAM.
   - Kernel 2 reads intermediate output, adds Bias, and writes to VRAM.
   - Kernel 3 reads bias output, applies ReLU, and writes final output to VRAM.
   - This process incurs substantial memory write-read latency.
3. **Fusion Solution**: TensorRT fuses these layers. The compiled GPU kernel reads the input once, executes Convolution, Bias add, and ReLU entirely on-chip within SRAM and Registers, and writes only the final output to VRAM.
4. **Benefit**: Eliminates redundant memory transfers, dramatically reducing memory bandwidth utilization which is the primary bottleneck in GPU inference.

### Q2: Why is a TensorRT compiled engine file (.plan/.engine) non-portable across different GPU architectures?
**Answer**:
1. **Kernel Selection**: During compilation, TensorRT performs Kernel Auto-Tuning, testing different mathematical implementations of layers directly on the physical host GPU. The selection of the fastest algorithm is highly dependent on the target chip's architecture, including the number of Streaming Multiprocessors (SMs), shared memory (SRAM) size, register limits, and the generation of Tensor Cores.
2. **Microarchitecture Differences**: An engine compiled for an Ampere GPU (A100) assumes specific layout properties and instruction behaviors that differ from a Hopper GPU (H100) or Ada Lovelace GPU (L4).
3. If you copy a compiled `.plan` file to a different GPU type, it will fail to load because the embedded CUDA kernel binaries are incompatible with the target chip's hardware configuration.

---

## References

1. **TensorRT Performance Analysis**: *NVIDIA TensorRT Developer Guide: Optimizing Deep Learning Inference*. (NVIDIA Developer Portal).
2. **In-Flight Batching & TensorRT-LLM**: *Fast LLM Inference on NVIDIA GPUs with TensorRT-LLM*. (NVIDIA Technical Blog).
