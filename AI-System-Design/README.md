<div align="center">

# AI System Design

<img src="./assets/banner.png" alt="AI System Design Banner" width="100%" />

### *The Ultimate Guide to Learning, Designing, and Building Production-Scale AI Systems*

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=flat-square)](LICENSE)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)](CONTRIBUTING.md)
[![Contributions Welcome](https://img.shields.io/badge/Contributions-Welcome-blue.svg?style=flat-square)](CONTRIBUTING.md)
[![Awesome](https://awesome.re/badge.svg)](https://awesome.re)

---

**AI System Design** is a comprehensive, production-grade guide and repository curated for developers, machine learning engineers, and architects looking to design, deploy, and scale AI-native applications. From model fundamentals to complex distributed agent architectures and production-level MLOps, this space has everything you need.

</div>

## 🛠️ Core Technology Stack & Libraries

To build state-of-the-art AI systems, you need a robust ecosystem. This repository covers design patterns and implementations featuring:

| Category | Technologies & Libraries |
| :--- | :--- |
| **LLMs & Foundations** | ![Python](https://img.shields.io/badge/python-3670A0?style=for-the-badge&logo=python&logoColor=ffdd54) ![OpenAI](https://img.shields.io/badge/OpenAI-412991?style=for-the-badge&logo=openai&logoColor=white) ![PyTorch](https://img.shields.io/badge/PyTorch-%23EE4C2C.svg?style=for-the-badge&logo=PyTorch&logoColor=white) ![Hugging Face](https://img.shields.io/badge/%F0%9F%A4%97%20Hugging%20Face-F9D342?style=for-the-badge&logoColor=black) |
| **Vector Databases** | ![Pinecone](https://img.shields.io/badge/Pinecone-000000?style=for-the-badge) ![Milvus](https://img.shields.io/badge/Milvus-00b4db?style=for-the-badge) ![Chroma](https://img.shields.io/badge/Chroma-orange?style=for-the-badge) |
| **Model Serving & Inference** | ![vLLM](https://img.shields.io/badge/vLLM-purple?style=for-the-badge) ![Nvidia Triton](https://img.shields.io/badge/Nvidia_Triton-76B900?style=for-the-badge&logo=nvidia&logoColor=white) ![TensorRT](https://img.shields.io/badge/TensorRT-green?style=for-the-badge) |
| **MLOps & Infrastructure** | ![Docker](https://img.shields.io/badge/docker-%230db7ed.svg?style=for-the-badge&logo=docker&logoColor=white) ![Kubernetes](https://img.shields.io/badge/kubernetes-%23326ce5.svg?style=for-the-badge&logo=kubernetes&logoColor=white) ![MLflow](https://img.shields.io/badge/MLflow-blue?style=for-the-badge&logo=mlflow) ![Kubeflow](https://img.shields.io/badge/Kubeflow-lightblue?style=for-the-badge&logo=kubeflow) |
| **Cloud & Deployment** | ![AWS](https://img.shields.io/badge/AWS-%23FF9900.svg?style=for-the-badge&logo=amazon-aws&logoColor=white) ![GCP](https://img.shields.io/badge/Google_Cloud-%234285F4.svg?style=for-the-badge&logo=google-cloud&logoColor=white) ![Terraform](https://img.shields.io/badge/terraform-%235835CC.svg?style=for-the-badge&logo=terraform&logoColor=white) |

---

## 🗺️ Learning Roadmap

<img src="./diagrams/roadmap_overall.png" alt="AI System Design Overall Learning Roadmap" width="500" />

---

## 📂 Repository Structure

Explore each module to deep-dive into source code, design mockups, and step-by-step walkthroughs:

```directory
AI-System-Design/
├── 01-Fundamentals/              # Core ML, Transformers, Embeddings, Tokenization
├── 02-LLM-System-Design/         # ChatGPT Architecture, RAG Systems, Agents, Context Management
├── 03-Architecture/              # Microservices, API Gateways, Event-Driven Patterns, Caching
├── 04-Data/                      # SQL vs NoSQL, Streaming, Data Lakes, ETL Pipelines
├── 05-Vector-Search/             # FAISS, Pinecone, Weaviate, Milvus, Chroma
├── 06-Model-Serving/             # vLLM, Triton Server, TensorRT, Quantization, GPU optimization
├── 07-MLOps/                     # Kubeflow, MLflow, CI/CD for ML, Kubernetes orchestration
├── 08-Cloud/                     # Infrastructure as Code (Terraform), AWS, GCP, Azure setup
├── 09-Case-Studies/              # Real-world architectures (Netflix, Uber, GitHub Copilot)
└── 10-Interview-Questions/       # FAANG interview guides from Beginner to Principal level
```

---

## 📘 Modules Breakdown

### 🧠 [01. Fundamentals](./01-Fundamentals/README.md)
*Core neural network paradigms and architectural foundations.*
* [AI vs ML vs DL](./01-Fundamentals/AI_vs_ML_vs_DL.md) — Artificial Intelligence, Machine Learning, and Deep Learning differences.
* [LLM Basics](./01-Fundamentals/LLM_Basics.md) — Under the hood of Large Language Models.
* [Transformers](./01-Fundamentals/Transformers.md) — Attention mechanism, encoders, decoders, and architecture.
* [Embeddings](./01-Fundamentals/Embeddings.md) — How dense representations capture semantic meaning.
* [Vector Databases](./01-Fundamentals/Vector_Databases.md) — High-dimensional indexing and search.
* [Tokenization](./01-Fundamentals/Tokenization.md) — Byte-Pair Encoding (BPE), WordPiece, and text preparation.

### 🤖 [02. LLM System Design](./02-LLM-System-Design/README.md)
*Designing interactive, context-aware AI systems.*
* [ChatGPT Architecture](./02-LLM-System-Design/ChatGPT_Architecture.md) — RLHF, instruction tuning, and scaling.
* [RAG System](./02-LLM-System-Design/RAG_System.md) — Retrieval-Augmented Generation workflows.
* [AI Search](./02-LLM-System-Design/AI_Search.md) — Combining semantic search with keyword retrieval.
* [AI Agent](./02-LLM-System-Design/AI_Agent.md) — ReAct, tool use, and loop control structures.
* [Multi-Agent](./02-LLM-System-Design/Multi_Agent.md) — Orchestration, routing, and consensus protocols.
* [Prompt Engineering](./02-LLM-System-Design/Prompt_Engineering.md) — Few-shot, chain-of-thought, skeleton prompts.
* [Context Management](./02-LLM-System-Design/Context_Management.md) — Sliding windows, attention compression, long contexts.
* [Memory System](./02-LLM-System-Design/Memory_System.md) — Short-term vs long-term memory for chatbots.

### 🏗️ [03. Architecture](./03-Architecture/README.md)
*System design fundamentals for high-availability systems.*
* [High-Level Architecture](./03-Architecture/High_Level_Architecture.md) — Global scale request routing.
* [Microservices](./03-Architecture/Microservices.md) — Monolith decomposition and service separation.
* [Event-Driven](./03-Architecture/Event_Driven.md) — Event sourcing, pub-sub architectures.
* [Message Queues](./03-Architecture/Message_Queues.md) — RabbitMQ, Kafka, and background processing.
* [API Gateway](./03-Architecture/API_Gateway.md) — Routing, rate limiting, and SSL termination.
* [Authentication](./03-Architecture/Authentication.md) — OAuth2, JWT, and session management.
* [Rate Limiting](./03-Architecture/Rate_Limiting.md) — Token bucket, leaking bucket, and sliding window logs.
* [Caching](./03-Architecture/Caching.md) — Redis, Memcached, eviction policies (LRU/LFU).

### 📊 [04. Data](./04-Data/README.md)
*Handling multi-modal, large-scale, streaming datasets.*
* [SQL](./04-Data/SQL.md) & [NoSQL](./04-Data/NoSQL.md) — Relational databases vs document, wide-column, key-value stores.
* [Data Pipelines](./04-Data/Data_Pipelines.md) & [ETL](./04-Data/ETL.md) — Batch and real-time data ingestion.
* [Streaming](./04-Data/Streaming.md) — Apache Flink, Spark Streaming.
* [Data Lake](./04-Data/Data_Lake.md) & [Data Warehouse](./04-Data/Data_Warehouse.md) — Delta Lake, Iceberg, Snowflake, BigQuery.

### 🔍 [05. Vector Search](./05-Vector-Search/README.md)
*Deep dive into spatial index architectures and matching algorithms.*
* [FAISS](./05-Vector-Search/FAISS.md) — Facebook AI Similarity Search algorithms (IVF-PQ).
* [Pinecone](./05-Vector-Search/Pinecone.md) — Managed vector database operations.
* [Weaviate](./05-Vector-Search/Weaviate.md) — Open-source hybrid vector database.
* [Chroma](./05-Vector-Search/Chroma.md) — Lightweight embedding database for local development.
* [Milvus](./05-Vector-Search/Milvus.md) — Enterprise-grade cloud-native vector search.

### ⚡ [06. Model Serving](./06-Model-Serving/README.md)
*Scaling low-latency, high-throughput model inference.*
* [vLLM](./06-Model-Serving/vLLM.md) — PagedAttention and high-concurrency serving.
* [Triton Server](./06-Model-Serving/Triton.md) — Multi-framework serving (PyTorch, ONNX, TensorRT).
* [TensorRT](./06-Model-Serving/TensorRT.md) — Deep learning inference compilation.
* [Quantization](./06-Model-Serving/Quantization.md) — Post-training quantization (FP16, INT8, INT4, AWQ, GPTQ).
* [GPU Inference](./06-Model-Serving/GPU_Inference.md) — VRAM budgeting, speculative decoding, KV caching optimization.

### 🔄 [07. MLOps](./07-MLOps/README.md)
*Managing lifecycle, automation, and system reliability.*
* [MLflow](./07-MLOps/MLflow.md) — Model registry and experiment tracking.
* [Kubeflow](./07-MLOps/Kubeflow.md) — Pipeline orchestration on Kubernetes.
* [Docker](./07-MLOps/Docker.md) & [Kubernetes](./07-MLOps/Kubernetes.md) — Containerization and scaling clusters.
* [CI/CD](./07-MLOps/CI_CD.md) — Automated testing and model deployment.
* [Monitoring](./07-MLOps/Monitoring.md) — Data drift, model decay, Prometheus & Grafana alerting.

### ☁️ [08. Cloud](./08-Cloud/README.md)
*Infrastructure deployment setups.*
* [AWS](./08-Cloud/AWS.md), [Azure](./08-Cloud/Azure.md), [GCP](./08-Cloud/GCP.md) — Cloud architectures for AI workloads.
* [Terraform](./08-Cloud/Terraform.md) — Multi-cloud resource provision scripts.
* [Infrastructure](./08-Cloud/Infrastructure.md) — GPU instance configuration, Infiniband networking.

### 🏢 [09. Case Studies](./09-Case-Studies/README.md)
*System designs of production systems from industry leaders.*
* [ChatGPT](./09-Case-Studies/ChatGPT.md), [Netflix](./09-Case-Studies/Netflix.md), [Uber](./09-Case-Studies/Uber.md), [Airbnb](./09-Case-Studies/Airbnb.md), [LinkedIn](./09-Case-Studies/LinkedIn.md)
* [GitHub Copilot](./09-Case-Studies/GitHub_Copilot.md), [Cursor](./09-Case-Studies/Cursor.md), [Perplexity AI](./09-Case-Studies/Perplexity.md)

### 🎯 [10. Interview Questions](./10-Interview-Questions/README.md)
*Preparation checklists for FAANG-level positions.*
* [Beginner](./10-Interview-Questions/Beginner.md), [Intermediate](./10-Interview-Questions/Intermediate.md), [Advanced](./10-Interview-Questions/Advanced.md), [FAANG](./10-Interview-Questions/FAANG.md)

---

## 🤝 Contributing

We welcome contributions to make this guide even more robust!
1. Fork the repo.
2. Create a branch (`git checkout -b feature/AmazingFeature`).
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`).
4. Push to the branch (`git push origin feature/AmazingFeature`).
5. Open a Pull Request.

---

## 📄 License

Distributed under the MIT License. See [LICENSE](LICENSE) for more information.
