# NVIDIA Triton Inference Server

## Overview

**NVIDIA Triton Inference Server** is an open-source, enterprise-grade model serving platform. It is designed to host multiple machine learning models (e.g., LLMs, Vision models, Tabular models) on CPUs or GPUs. Triton standardizes model serving by providing HTTP/gRPC API endpoints, concurrent model execution, dynamic batching, and custom pipeline orchestration through Ensembles and Business Logic Scripting (BLS).

---

## Problem Statement

Deploying models to production using simple Python web frameworks (e.g., FastAPI, Flask) introduces several limitations:
1. **GPU Underutilization**: Single-threaded Python servers cannot easily parallelize execution. If a model is lightweight (e.g., an embedding model), the GPU will sit idle waiting for network request parsing.
2. **Framework Lock-in**: Serving an application that uses PyTorch, TensorFlow, and ONNX models requires maintaining multiple independent web servers and serialization pipelines.
3. **Inefficient Batching**: Applications must write custom queuing and thread locks to merge incoming asynchronous user queries into batches to leverage GPU tensor cores.

---

## Triton Architecture & Internal Features

Triton addresses these limitations by placing an optimized scheduler and execution gateway in front of raw model backends:

```
                      [Client Request (HTTP/gRPC)]
                                   │
                                   ▼
                       [Triton API Gateway & Ingress]
                                   │
                      ┌────────────┴────────────┐
                      ▼                         ▼
             [Dynamic Batcher]          [Model Scheduler]
                      │                         │
                      └────────────┬────────────┘
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        ▼                          ▼                          ▼
 [Model A (PyTorch)]      [Model B (TensorRT)]       [Model C (ONNX)]
 (GPU Instance 0)         (GPU Instance 1)           (GPU Instance 1)
```

### 1. Concurrent Model Execution
- Triton allows multiple models (or multiple instances of the same model) to run concurrently on a single GPU.
- If a GPU has 80 GB VRAM, Triton can allocate 10 GB to a PyTorch embedding model, 20 GB to an ONNX classification model, and 50 GB to a TensorRT-LLM model.
- Triton's execution engine automatically schedules requests to utilize idle streaming multiprocessors (SMs) across the active models.

### 2. Dynamic Batching
Dynamic batching is a server-side feature that groups individual inference requests into a single batch to maximize GPU processing efficiency.
- **Configurable Parameters**:
  * `max_queue_delay_microseconds`: How long Triton will wait to collect requests before starting inference.
  * `preferred_batch_size`: Target sizes (e.g., 4, 8, 16) that align with GPU hardware optimizations.
- **Execution**: If Triton receives 3 separate requests within the configured queue delay, it groups their tensors, executes a single batch forward pass, splits the output tensors, and returns the individual responses.

### 3. Pipeline Orchestration: Ensembles vs. BLS
Modern AI workflows require chaining multiple models together (e.g., automatic speech recognition -> translation -> LLM response -> text-to-speech). Triton offers two ways to chain models:

- **Ensemble Models**:
  - A static pipeline where outputs of one model are mapped directly to the inputs of the next model.
  - *Pros*: Zero serialization overhead. Data stays in GPU VRAM (no round-trips to CPU or host memory).
  - *Cons*: Cannot handle conditional routing (e.g., "if language is French, route to Model A, else Model B").
- **Business Logic Scripting (BLS)**:
  - Executes a custom Python or C++ script within Triton's process space.
  - Allows conditional branching, loops, and dynamic error handling.
  - *Pros*: Maximum flexibility.
  - *Cons*: Minor overhead compared to static ensembles, but still significantly faster than calling models across the network.

---

## Triton Model Repository Structure

Triton expects a structured folder repository to discover and load models automatically:

```
model_repository/
├── text_detector/
│   ├── config.pbtxt        # Model configuration (inputs, outputs, batching rules)
│   └── 1/
│       └── model.onnx      # Model weights file (Version 1)
└── text_recognizer/
    ├── config.pbtxt
    └── 1/
        └── model.pt        # PyTorch TorchScript weights file
```

---

## Design Decisions & Trade-offs

### Triton vs. FastAPI + PyTorch

- **Triton Inference Server**:
  * *Pros*: Native C++ engine, built-in dynamic batching, concurrent model execution, model versioning (hot reloading), metrics export (Prometheus/Grafana).
  * *Cons*: Steeper learning curve, requires writing verbose `config.pbtxt` files, containerized setup.
- **FastAPI / Python Serving**:
  * *Pros*: Simple to code, quick to deploy, flexible custom logic.
  * *Cons*: Poor concurrency (Python GIL), no native queuing/batching, slow tensor serialization.

---

## Interview Questions

### Q1: How does Dynamic Batching work in Triton? What parameters configure it, and what are the trade-offs?
**Answer**:
1. **Dynamic Batching** queues incoming individual requests over a short time window and groups them into a single batch to execute concurrently on the GPU.
2. **Parameters**:
   - `max_queue_delay_microseconds`: The maximum time the scheduler will wait to collect requests to form a preferred batch.
   - `preferred_batch_size`: The batch sizes that Triton tries to form (e.g., `[4, 8, 16]`).
3. **Trade-offs**:
   - **Higher Delay**: Increases throughput (high GPU utilization) but introduces latency overhead for the first request in the queue (since it must wait for the delay window to expire).
   - **Lower Delay**: Lowers single-request latency but decreases overall server throughput under heavy traffic because the GPU executes sub-optimal batch sizes.

### Q2: What is the difference between Triton Ensembles and Business Logic Scripting (BLS)? When should you use each?
**Answer**:
1. **Triton Ensembles**:
   - A static configuration (`config.pbtxt`) mapping the outputs of Model A to inputs of Model B.
   - **Use Case**: Simple pipelines without logic gates (e.g., Preprocess -> Model -> Postprocess).
   - **Benefit**: Extremely low latency; data remains in GPU memory, avoiding CPU memory copy overhead.
2. **Business Logic Scripting (BLS)**:
   - A Python/C++ script that calls models dynamically.
   - **Use Case**: Complex logic, loops, conditional routing (e.g., "if classifier score $< 0.8$, call fallback model"), or executing external API requests.
   - **Trade-off**: Slightly higher latency than ensembles due to scripting engine translation, but prevents writing complex application orchestration code.

---

## References

1. **NVIDIA Triton documentation**: *Triton Inference Server User Guide*. (NVIDIA Developer Portal).
2. **Model Serving Architecture**: *Triton Architecture Overview & High-Performance Pipelines*. (NVIDIA Deep Learning Institute).
