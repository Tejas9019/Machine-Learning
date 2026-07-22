# Docker in MLOps: Containerization & CUDA Isolation

## Overview

**Docker** is the foundational containerization technology used in MLOps to package machine learning applications, libraries, and system dependencies into isolated, reproducible runtime environments. When designing Docker containers for machine learning, engineers focus on optimizing large GPU-accelerated base images, leveraging multi-stage builds, and establishing clean interfaces between the containerized application and physical GPU drivers.

---

## Problem Statement

Deploying machine learning models in standard Docker containers introduces size and hardware bottlenecks:
1. **Extreme Container Sizes**: Combining deep learning frameworks (PyTorch is $\sim 2$ GB), CUDA libraries (CUDA runtime is $\sim 3$ GB), and model weights can lead to massive images ($>10$ GB), causing slow network transfers and deployment delays.
2. **CUDA Dependency Conflicts**: Models compiled against specific CUDA toolkit versions (e.g., CUDA 11.8) will fail to run or experience memory segmentation faults if deployed in environments running incompatible base images.
3. **Host GPU driver coupling**: Containers must access the physical host's GPU hardware. Tightly coupling the GPU host driver version inside the container limits the container's portability across cloud providers.

---

## Docker Core Mechanics for ML

To build portable and lightweight model containers, MLOps engineers employ optimized image hierarchies:

```
                  [Physical GPU Hardware (Host)]
                                │
                  [NVIDIA Host Driver (Host)]
                                │
                 [NVIDIA Container Runtime]
                                │ (Binds driver libraries dynamically)
    ┌───────────────────────────┴───────────────────────────┐
    ▼                                                       ▼
[Build Container Stage]                         [Runtime Container Stage]
(Uses CUDA-Devel image)                         (Uses CUDA-Runtime/Base image)
├─ Compiles C++/CUDA extensions                  ├─ Copies compiled binary wheels
└─ Downloads heavy build compilers              └─ Contains minimal PyTorch dependencies
```

### 1. CUDA Base Image Selection
NVIDIA provides structured Docker base images on DockerHub split into three categories:
- **`base`**: Contains the bare minimum libraries to run applications that use CUDA. Does not contain the compiler toolchain or runtime libraries. (Smallest size, ideal for final deployment).
- **`runtime`**: Includes the CUDA runtime, math libraries, and communication libraries (NCCL). (Medium size, ideal for general model serving).
- **`devel`**: Includes the full CUDA compiler toolchain (nvcc), headers, and debugging tools. (Largest size, required strictly for building custom C++/CUDA extensions).

---

### 2. Multi-Stage Builds
Multi-stage builds allow developers to compile custom CUDA libraries or install heavy build tools in one container, and copy only the compiled binaries into a clean, lightweight final container:

```dockerfile
# Stage 1: Build Environment
FROM nvidia/cuda:12.1.0-devel-ubuntu22.04 AS builder
RUN apt-get update && apt-get install -y build-essential python3-dev pip
COPY requirements.txt .
# Compile custom CUDA kernels (e.g. FlashAttention)
RUN pip wheel --no-cache-dir --wheel-dir=/root/wheels -r requirements.txt

# Stage 2: Minimal Runtime Environment
FROM nvidia/cuda:12.1.0-runtime-ubuntu22.04
RUN apt-get update && apt-get install -y python3 pip && rm -rf /var/lib/apt/lists/*
# Copy compiled wheels from Stage 1
COPY --from=builder /root/wheels /root/wheels
RUN pip install --no-index --find-links=/root/wheels /root/wheels/* && rm -rf /root/wheels
```

---

### 3. GPU Driver Isolation & NVIDIA Container Toolkit
- Containers run in isolated user namespaces and cannot see host hardware directly.
- **NVIDIA Container Toolkit**: Modifies the Docker daemon configuration so that when a container is started with the `--gpus all` flag, the engine dynamically mounts the host's physical GPU driver user-space libraries (e.g., `libcuda.so`) into the container.
- **Benefit**: The container does *not* contain the GPU driver. It only contains the CUDA compiler/runtime libraries, ensuring it can run on any host server regardless of the exact physical GPU card installed, as long as the host's driver version supports the container's CUDA version.

---

## Design Decisions & Trade-offs

### Alpine Linux vs. Ubuntu for ML Containers

- **Alpine Linux (`python:3.10-alpine`)**:
  * *Pros*: Extremely small base image size ($<10$ MB), minimal security attack surface.
  * *Cons*: Lacks `glibc` (uses `musl`), which is required by most pre-compiled Python binary wheels (including NumPy, PyTorch, and TensorFlow). Forcing compilation from scratch inside Alpine requires installing heavy compilation tools, resulting in larger final images and hours of build time.
- **Ubuntu/Debian (`nvidia/cuda:xx-runtime-ubuntu`)**:
  * *Pros*: Full `glibc` support, matches NVIDIA's official CUDA base distributions, pre-compiled wheels install instantly.
  * *Cons*: Larger base image footprint (starting at $\sim 500$ MB).

---

## Cost Optimization

- **Docker Layer Caching**: Order instructions in the `Dockerfile` from least-frequently changed to most-frequently changed. Copy `requirements.txt` and run `pip install` *before* copying the application source code. This prevents rebuilds of the entire dependency layer on every minor code change.
- **Apt Cache Pruning**: Clean up package manager caches in the same `RUN` command where they are installed:
  ```dockerfile
  RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*
  ```

---

## Interview Questions

### Q1: Why is it bad practice to install host NVIDIA drivers inside a Docker container? How does the NVIDIA Container Toolkit solve this?
**Answer**:
1. **Bad Practice**: Installing the host NVIDIA GPU driver inside a Docker image binds the image to the exact hardware kernel version of the build machine. If that container is deployed to a cloud server with a different host GPU driver version, the mismatch will cause driver load errors and prevent the application from accessing the GPU.
2. **NVIDIA Container Toolkit Solution**:
   - The toolkit decouples the container from the host GPU driver.
   - The Docker image is built containing only user-space CUDA runtime libraries (e.g., PyTorch CUDA binaries).
   - When launching the container (e.g., via `docker run --gpus all`), the NVIDIA container runtime intercept mounts the host's physical GPU driver user-space library files (like `libcuda.so`) directly into the container directory structure at runtime.
   - This ensures the container remains portable, running on any host machine that has the NVIDIA Container Toolkit and a compatible driver version installed.

### Q2: Walk through how a multi-stage Docker build reduces the final model image size.
**Answer**:
1. **Build Phase (Stage 1)**: To build custom C++ or CUDA extensions (like compilation of custom FlashAttention kernels), we require development tools: compilers (`gcc`, `g++`), build libraries (`make`, `cmake`), and the full NVIDIA CUDA compiler development toolkit (`nvcc`), which occupies several gigabytes (`nvidia/cuda:devel`).
2. **Runtime Phase (Stage 2)**: Once compilation finishes, we only need the generated `.so` binaries or Python `.whl` files to execute the model. We do not need the development compilers or CUDA headers anymore.
3. **Execution**: In a multi-stage build, we start a fresh, lightweight runtime base image (`nvidia/cuda:runtime` or simple `ubuntu`) in the second stage, copy only the compiled binaries from the first stage, and discard the heavy build environment. This can reduce the final Docker image size from $10+$ GB down to $2$-$3$ GB.

---

## References

1. **NVIDIA Container Toolkit Guide**: *Architecture Overview of the NVIDIA Container Toolkit*. (NVIDIA Developer Documentation).
2. **Docker Best Practices**: *Best practices for writing Dockerfiles: Multi-stage builds*. (Docker Documentation).
