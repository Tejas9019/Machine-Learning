# Model Quantization: Theory & Formats

## Overview

**Model Quantization** is a critical optimization technique that reduces the numerical precision of neural network weights and activations. By converting high-precision floating-point values (e.g., FP32 or FP16) to lower-precision formats (e.g., FP8, INT8, or INT4), quantization reduces the model's memory footprint, increases arithmetic throughput, and alleviates the memory bandwidth bottleneck on hardware platforms.

---

## Mathematical Foundations of Quantization

Quantization maps continuous real numbers in a large range to discrete integers or lower-precision representations.

### 1. Uniform Quantization Equation
Given a continuous real value $x \in [\alpha, \beta]$ and target integer range $[q_{\text{min}}, q_{\text{max}}]$, the quantized value $x_q$ is calculated as:
$$x_q = \text{clip}\left(\text{round}\left(\frac{x}{S}\right) + Z, q_{\text{min}}, q_{\text{max}}\right)$$
The dequantized value $\hat{x}$ (reconstructed approximation) is given by:
$$\hat{x} = S \times (x_q - Z)$$

- **Scale ($S$)**: A floating-point number representing the step size of the quantizer:
  $$S = \frac{\beta - \alpha}{q_{\text{max}} - q_{\text{min}}}$$
- **Zero-point ($Z$)**: An integer offset ensuring that the real value $0.0$ maps exactly to a quantized integer value (crucial for zero-padding in convolutional/transformer layers):
  $$Z = \text{round}\left(\frac{-\alpha}{S}\right) + q_{\text{min}}$$

### 2. Symmetric vs. Asymmetric Quantization
- **Asymmetric Quantization**: Maps the actual dynamic range $[\min(x), \max(x)]$ to the target range. The zero-point $Z$ is non-zero.
- **Symmetric Quantization**: Restricts the range to be symmetric around zero: $[-\gamma, \gamma]$, where $\gamma = \max(|x|)$. 
  * In this case, the zero-point $Z = 0$, simplifying downstream matrix multiplication math:
    $$x_q = \text{round}\left(\frac{x}{S}\right)$$

---

## Ingestion Pipelines: PTQ vs. QAT

Quantization can be applied during training or as a post-processing step:

| Feature | Post-Training Quantization (PTQ) | Quantization-Aware Training (QAT) |
| :--- | :--- | :--- |
| **Workflow** | Apply quantization directly to a pre-trained model checkpoint. | Fine-tune or train the model while simulating lower precision. |
| **Calibration** | Requires a small calibration dataset ($100$-$1000$ samples) to determine scales ($S$). | Uses straight-through estimators (STE) to bypass rounding gradients during backprop. |
| **Accuracy** | Minor drop in accuracy for large models; severe drop in small models ($<7$B parameters). | Preserves accuracy near $100\%$ of the baseline float model. |
| **Compute Cost** | Extremely low (minutes to hours on a single GPU). | Very high (requires a complete training run or long fine-tuning). |

---

## Modern Quantization Formats & Algorithms

As models scaled, simple uniform quantization failed due to precision collapse. Specialized formats and algorithms were introduced:

### 1. INT8 & The Outlier Problem
- **Weight-Only Quantization**: Quantizes weights to INT8 but keeps activations in FP16. High speedup but limits throughput improvements because activations remain large.
- **Weight-Activation (W8A8) Quantization**: Quantizes both weights and activations to INT8, enabling high-speed INT8 Tensor Core math.
- **The Outlier Feature Problem (LLM.int8())**: In LLMs beyond 6.7B parameters, a small set of activation channels (outliers) develop extreme magnitudes (e.g., $100\times$ normal values). If scaled uniformly, non-outliers are crushed to 0. 
- *Solution*: LLM.int8() separates outlier channels, processes them in FP16, processes the remaining $99.9\%$ channels in INT8, and merges the final tensors.

---

### 2. INT4 Algorithms (AWQ vs. GPTQ)
To compress models to 4-bit precision without losing reasoning capabilities, advanced algorithms protect "salient" weights:

- **AWQ (Activation-aware Weight Quantization)**:
  - **Observation**: Not all weights are equally important. Weights corresponding to high-magnitude activation channels dictate model output quality.
  - **Mechanics**: AWQ does not perform mathematical updates to weights. Instead, it scales up salient weight channels by a grid-searched scale factor ($s \ge 1$) and divides the activations by the same factor, reducing quantization noise on the most sensitive paths before applying uniform INT4 rounding.
- **GPTQ (Generalized Post-Training Quantization)**:
  - **Observation**: Quantizing one weight introduces error; adjusting neighboring weights can compensate for this error.
  - **Mechanics**: Based on second-order Taylor expansion optimization (Optimal Brain Surgeon). It calculates the inverse Hessian matrix ($H^{-1}$) of activations, quantizes weights layer by layer, and dynamically updates the remaining unquantized weights to minimize mean squared error (MSE) loss.

---

### 3. FP8 (Floating-Point 8-bit)
FP8 is native hardware quantization supported by modern architectures (Nvidia Hopper H100, Blackwell). It offers two formats:
- **E4M3 (4 exponent bits, 3 mantissa bits)**: Offers high precision, ideal for forward pass activations and weights where dynamic range is relatively tight.
- **E5M2 (5 exponent bits, 2 mantissa bits)**: Mimics FP16 dynamic range with fewer precision bits, ideal for gradients and backward pass calculations.

---

## Interview Questions

### Q1: Why do activation outliers make W8A8 quantization difficult in Large Language Models? How does LLM.int8() solve this?
**Answer**:
1. **Outlier Problem**: In models with $>6.7$B parameters, specific hidden dimensions develop systematic outliers with magnitudes up to $100\times$ the standard activation range.
2. If we apply standard symmetric quantization, the scale factor $S$ must span the outlier range (e.g., $-100$ to $100$). For standard activation values (which lie between $-1$ and $1$), dividing by this large scale factor and rounding maps them all to $0$ or $-1$, destroying the representation power of the network.
3. **LLM.int8() Solution**:
   - Outliers are identified by checking if a channel's absolute value exceeds a threshold (e.g., $6.0$).
   - The activation matrix is split column-wise: the outlier channels (typically $<0.1\%$ of the total) are extracted.
   - The outlier activations and their corresponding weights are multiplied in high-precision FP16.
   - The remaining standard channels are quantized and multiplied in INT8.
   - The output of the FP16 and INT8 operations are added together to construct the final layer output, maintaining FP16 accuracy at reduced VRAM consumption.

### Q2: What is the difference between AWQ and GPTQ?
**Answer**:
1. **AWQ (Activation-aware Weight Quantization)**:
   - Does not modify weight values. It performs channel-wise scaling based on activation magnitudes to shield salient weights from quantization noise.
   - *Pros*: Fast, calibration-dataset independent (no overfitting), easily generalizes to new domains.
2. **GPTQ**:
   - Modifies weight values. It uses the inverse Hessian matrix of activations to adjust unquantized weights after each quantization step to minimize output reconstruction error.
   - *Pros*: Excellent compression ratios (can achieve 3-bit quantization).
   - *Cons*: Slower calibration process, risks minor overfitting to the calibration dataset.

---

## References

1. **LLM.int8()**: Dettmers, T., et al. (2022). *LLM.int8(): 8-bit Matrix Multiplication for Transformers at Scale*. NeurIPS.
2. **GPTQ Paper**: Frantar, E., et al. (2022). *GPTQ: Accurate Post-Training Quantization for Generative Pre-trained Transformers*. ICLR.
3. **AWQ Paper**: Lin, J., et al. (2023). *AWQ: Activation-aware Weight Quantization for LLM Compression and Acceleration*. arXiv preprint.
