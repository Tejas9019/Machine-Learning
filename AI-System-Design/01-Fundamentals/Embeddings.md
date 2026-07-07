# Text Embeddings

## Overview

Text embeddings are dense, continuous, low-dimensional vector representations that capture the semantic and syntactic relationships of text sequences. An embedding model projects discrete characters and words into a continuous vector space $\mathbb{R}^d$ (where $d$ typically ranges from 384 to 3072 dimensions), where geometric proximity correlates with semantic similarity.

---

## Problem Statement

Computer models cannot natively process raw human characters. Early NLP systems attempted to map words using **Sparse Representations** like TF-IDF or BM25. 

* **TF-IDF (Term Frequency-Inverse Document Frequency)**:
$$\text{TF-IDF}(t, d, D) = \text{TF}(t, d) \times \log\left(\frac{|D|}{1 + |\{d \in D : t \in d\}|}\right)$$
* **BM25 (Best Matching 25)**: Evaluates query terms $q_i$ against document $D$ by scaling term frequency relative to document length:
$$\text{score}(D, Q) = \sum_{i=1}^{n} \text{IDF}(q_i) \cdot \frac{f(q_i, D) \cdot (k_1 + 1)}{f(q_i, D) + k_1 \cdot \left(1 - b + b \cdot \frac{|D|}{\text{avgdl}}\right)}$$
  * *Where $f(q_i, D)$ is term frequency, $|D|$ is document length, $\text{avgdl}$ is average document length in the corpus, $k_1$ regulates term frequency saturation (usually 1.2 to 2.0), and $b$ controls length normalization (usually 0.75).*

While powerful for exact keyword matching, sparse representations suffer from:
1. **High Dimensionality**: Vectors scale with the size of the vocabulary ($|V|$), leading to sparse matrices that consume excessive memory.
2. **Orthogonality**: No semantic relationships are captured (e.g., the dot product between the sparse vectors for "car" and "automobile" is 0).
3. **Context Blindness**: Cannot differentiate homonyms (e.g., "financial bank" vs. "river bank").

Dense embeddings solve these limitations by compressing semantic relationships and context into fixed-size dense vectors.

---

## Architecture

Generating a dense vector embedding from raw text through a Transformer encoder structure:

```mermaid
flowchart LR
    A["Raw Text String"] --> B[Tokenizer]
    B --> C["Token IDs [T1, T2, T3]"]
    C --> D[Transformer Encoder]
    D --> E["Token Embeddings (Sequence of Vectors)"]
    E --> F["Pooling Layer (Mean/CLS)"]
    F --> G["L2 Normalization"]
    G --> H["Semantic Vector [X1, X2, ... Xd]"]

    style A fill:#2b2d42,stroke:#8d99ae,stroke-width:1px,color:#fff
    style H fill:#3d5a80,stroke:#98c1d9,stroke-width:2px,color:#fff
```

---

## Components

### 1. Distance Metrics
To determine similarity between two embedding vectors $u$ and $v$ in $\mathbb{R}^d$:
* **Cosine Similarity**: Measures the cosine of the angle between two vectors, focusing on direction rather than magnitude:
$$\text{Cosine Sim}(u, v) = \frac{u \cdot v}{\|u\| \|v\|} = \frac{\sum_{i=1}^d u_i v_i}{\sqrt{\sum_{i=1}^d u_i^2} \sqrt{\sum_{i=1}^d v_i^2}}$$
* **L2 Distance (Euclidean)**: Measures the straight-line distance between two points in Euclidean space. Highly sensitive to vector magnitude differences:
$$d(u, v) = \sqrt{\sum_{i=1}^d (u_i - v_i)^2}$$
* **Dot Product**: Measures both direction and magnitude. If vectors are L2-normalized ($\|u\| = \|v\| = 1$), the Dot Product is mathematically equivalent to Cosine Similarity.

### 2. Pooling Strategies
Transformer encoders output a vector for each input token. To compress this sequence of vectors into a single document-level embedding, a pooling layer is applied:
* **CLS Token Pooling**: Takes the final hidden state of the special classification token (`[CLS]`) as the sequence representation.
* **Mean Pooling**: Computes the average of all token vectors. To prevent padding tokens from skewing the representation, the token vectors must be multiplied by the attention mask before averaging:
$$\text{Embedding} = \frac{\sum_{i=1}^s (\text{Vector}_i \times \text{Mask}_i)}{\sum_{i=1}^s \text{Mask}_i}$$

### 3. Contrastive Training (InfoNCE Loss)
Embedding models are fine-tuned using contrastive learning. The model is trained to pull matching query-document pairs close together in the vector space while pushing non-matching pairs far apart. This is optimized using the **InfoNCE Loss**:

$$\mathcal{L}_{\text{InfoNCE}} = -\log \frac{\exp(\text{sim}(q, d^+) / \tau)}{\exp(\text{sim}(q, d^+) / \tau) + \sum_{i=1}^K \exp(\text{sim}(q, d^-_i) / \tau)}$$

*Where $q$ is the query vector, $d^+$ is the positive (matching) document vector, $d^-_i$ are negative document vectors, $\text{sim}$ is cosine similarity, and $\tau$ is a temperature hyperparameter controlling the scale of predictions.*

---

## Design Decisions

### Bi-Encoders vs. Cross-Encoders
* **Bi-Encoder**: Encodes the query and document into separate embeddings independently.
  * *Pros*: Document vectors can be calculated and indexed in advance. Query similarity is computed using simple dot products, enabling sub-millisecond retrieval speeds.
  * *Cons*: Cannot capture token-to-token interactions between the query and document.
* **Cross-Encoder**: Feeds the query and document together into the transformer attention layers.
  * *Pros*: Computes cross-attention between all query and document tokens, yielding high reranking accuracy.
  * *Cons*: Extremely slow and computationally expensive. Cannot be pre-computed, making it unsuitable for first-stage retrieval over large datasets.

```
Bi-Encoder:
Query ───► [Encoder] ───► Vector A ┐
                                   ├──► Fast Dot Product (O(1))
Doc   ───► [Encoder] ───► Vector B ┘

Cross-Encoder:
Query ───┐
         ├──► [Encoder (Cross-Attention)] ───► Relational Score (O(N^2))
Doc   ───┘
```

---

## Scaling

### Matryoshka Representation Learning (MRL)
Standard embedding models project text into a fixed dimensionality. Storing large, high-dimensional vectors (e.g., 1536 dimensions) consumes significant memory and slows down nearest-neighbor searches. 
* **Matryoshka Representation Learning (MRL)** solves this by training the model to pack the most important semantic information into the early dimensions of the vector.
* During training, loss is calculated and summed across multiple nested dimensions (e.g., $d \in \{128, 256, 512, 1024, 2048\}$):
$$\mathcal{L}_{\text{MRL}} = \sum_{m \in \mathcal{M}} c_m \mathcal{L}(\theta; m)$$
  * *Where $\mathcal{M}$ is the set of active nested dimensions, and $c_m$ is a weight scaling parameter for each dimension.*
* In production, developers can truncate the output vectors to lower dimensions (e.g., from 1536 to 256). This reduces storage size by **83%** while retaining over **98%** of the model's retrieval accuracy.

---

## Failure Handling

* **Negation Blindness**: Embedding models often place negation antonyms close together in the vector space (e.g., mapping "is toxic" and "is NOT toxic" near each other).
  * *Solutions*: Combine semantic search with sparse keyword search (**Hybrid Search**), or use specialized rerankers (Cross-encoders) to catch syntax details during validation.

---

## Security

* **Vector Reconstruction Attacks**: Attackers can train decoder networks to reconstruct the original raw text from target embedding vectors.
  * *Mitigation*: Restrict public access to raw embedding coordinates. Apply custom non-linear projection matrices to vectors before exposing them to external client APIs.

---

## Cost Optimization

### Embedding Quantization
To reduce storage and infrastructure costs, floating-point vectors (FP32) can be quantized:
* **Scalar Quantization (SQ8)**: Quantizes 32-bit floats to 8-bit integers (INT8) by mapping values to 256 steps. Reduces memory footprint by **75%**.
* **Binary Quantization (BQ)**: Quantizes floats to 1-bit representations based on the sign of the coordinates:
$$f(x_i) = \begin{cases} 1 & x_i \geq 0 \\ 0 & x_i < 0 \end{cases}$$
  * This reduces vector memory usage by **97%**. Similarity is calculated using fast bitwise XOR operations (Hamming Distance), accelerating query speeds by up to **10x** with minimal loss in recall.

---

## Interview Questions

### 1. Why does Mean Pooling require the use of an attention mask vector when processing batched inputs?
**Answer**: In a batched input, sentences have varying lengths, and shorter sequences are padded with zero tokens (`[PAD]`) to match the longest sequence. During mean pooling, if we simply averaged all token states across the sequence, the zero padding states would pull down the average, corrupting the embedding. Multiplying by the binary attention mask (where pad positions are 0 and valid tokens are 1) ensures only real tokens contribute to the semantic average.

### 2. Explain how a Cross-Encoder acts as a reranker in a search pipeline.
**Answer**: Because Cross-encoders process query-document pairs together, they compute self-attention across all tokens in both inputs. This allows them to capture fine-grained semantic relationships, making them highly accurate. However, this $O((N_q + N_d)^2)$ complexity is too slow to run across millions of documents. In a search pipeline, we use a fast Bi-Encoder or BM25 to retrieve the top 100 candidate documents, and then use a Cross-Encoder to rerank only those 100 candidates.

---

## References

* [Kusner et al. (2015): From Word Embeddings To Document Distances](https://arxiv.org/abs/1502.06216)
* [Matryoshka Representation Learning (MRL) Research Paper](https://arxiv.org/abs/2205.13147)
* [Hugging Face: Rerankers and Cross-Encoders Guide](https://huggingface.co/blog/cross-encoder-reranking)
