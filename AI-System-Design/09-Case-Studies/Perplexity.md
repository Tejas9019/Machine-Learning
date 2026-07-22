# Case Study: Perplexity AI (Conversational Answer Engine)

## Overview

**Perplexity AI** is a conversational answer engine designed to provide direct, natural language answers backed by real-time citations. Unlike standard LLMs which rely on static pre-training data, Perplexity routes queries through an active search orchestration layer. It formulates search terms, queries search brokers (Bing/Google) in parallel, scrapes pages asynchronously, ranks relevant text snippets, and synthesizes answers with verifiable citations.

---

## Problem Statement

Building a real-time web-backed answer engine introduces several architectural challenges:
1. **Search Query Mismatch**: Conversational queries (e.g., "tell me how the stock of Apple did today compared to Microsoft after their earnings report") are too verbose for search engines, which optimize for short keywords.
2. **Real-time Scraping & Parsing Latency**: Fetching raw HTML pages from the web, cleaning text, and chunking data on-the-fly during the user's request path can add significant latencies.
3. **Hallucination Prevention**: Synthesizing answers must be grounded strictly in scraped content. The system must map claims back to specific URLs via verifiable citations.

---

## High-Level System Architecture

Perplexity orchestrates real-time web retrieval in parallel before executing model generation:

```
                          [User Search Query]
                                   │
                                   ▼
                         [Query Formulation LLM]
                        (Generates Search Keywords)
                                   │
         ┌─────────────────────────┴─────────────────────────┐
         ▼                                                   ▼
  [Search Broker API]                                 [Web Search Cache]
  (Bing / Google Search APIs)                         (Hot metadata hit)
         │                                                   │
         └─────────────────────────┬─────────────────────────┘
                                   ▼
                        [Scraping Worker Nodes]
                        (Async Headless Chrome/Fetch)
                                   │
                                   ▼
                       [Rerank & Similarity Pool]
                       (Dense Embeddings + BM25 -> RRF)
                                   │
                                   ▼
                        [Synthesizer LLM Engine]
                        (Generates answer with inline citations)
                                   │
                                   ▼
                           [Client UI (SSE)]
```

### 1. Query Formulation
- The user's conversational prompt is processed by a lightweight LLM (Query Formulator).
- It generates $2$-$5$ distinct search keyword queries suited for search engines, along with intent labels (e.g., "news", "code", "weather").

---

### 2. Parallel Search & Asynchronous Scraping
- The generated keywords are executed in parallel against search index APIs (Bing, Google, and Perplexity's internal web index).
- The top $10$-$15$ target page URLs are returned.
- **Scraping Workers**: An asynchronous worker pool fetches the page contents.
  * To bypass scraping blocks and execution delays, workers use headless scrapers, reading cached versions of pages or raw markdown extractions.

---

### 3. Chunking & Hybrid Reranking
- Scraped text is split into small passages ($\sim 200$ tokens).
- A **Bi-Encoder model** computes vector embeddings for the passages.
- **Similarity Search**: Calculates cosine similarity against the query embedding.
- **Hybrid Reranking**: Combines semantic vector similarity with keyword BM25 scores using **Reciprocal Rank Fusion (RRF)** to select the top $5$-$8$ most relevant snippets.

---

### 4. Synthesizing Answers with Citations
- The selected snippets are packed into the context window of the Synthesizer LLM.
- **System Prompt Instructions**:
  * "Answer the user query *only* based on the provided snippets."
  * "Every fact must be followed by an inline citation number matching the snippet index: `[1]`, `[2]`."
- The model generates the response, streaming tokens via SSE. The frontend client matches the inline citation indices with the source URL metadata, displaying clickable citation cards.

---

## Design Decisions & Trade-offs

### Parallel Search & Dynamic Scraping vs. Massive Local Indexing

- **Parallel Search + Dynamic Scraping (Perplexity)**:
  * *Pros*: Access to the real-time internet (news, stock prices). Lower storage costs (relying on search engine indexes).
  * *Cons*: Query latency depends on third-party APIs and network scraping speeds.
- **Massive Local Web Crawling (Google/Bing)**:
  * *Pros*: Extremely fast query times (retrieving pre-scraped, pre-indexed web data in milliseconds).
  * *Cons*: Requires massive infrastructure to crawl and index billions of pages daily, costing millions of dollars in storage and network bandwidth.

---

## Interview Questions

### Q1: Walk through the pipeline of how Perplexity answers a query with real-time web citations.
**Answer**:
1. **Query Formulation**: A lightweight LLM refines the conversational user query into search engine keywords.
2. **Retrieval**: Keywords are sent to search engines (Bing/Google) in parallel to retrieve the top URL results.
3. **Scraping**: Asynchronous worker threads fetch and parse raw text from the top URLs.
4. **Ranking**: The parsed text is chunked, embedded, and reranked using a hybrid vector-BM25 model to select the top 5 most relevant snippets.
5. **Prompt Construction**: The query and the top snippets (labeled with IDs like `[Doc 1]`, `[Doc 2]`) are sent to the generator LLM.
6. **Generation & SSE**: The LLM generates the response, formatting inline citations. The token stream is flushed to the client via Server-Sent Events (SSE).

### Q2: How does Perplexity handle scraping latencies when fetching real-time web pages?
**Answer**:
1. **Parallel Execution**: Search calls and page fetching run concurrently.
2. **Timeout Caps**: Scraping workers have strict timeout limits (e.g., $1.5\text{ seconds}$). If a website fails to respond within the limit, the worker discards it and falls back to other retrieved URLs.
3. **Metadata Snippets**: While waiting for full page scraping, Perplexity uses the meta-descriptions and snippet texts returned directly by the search index APIs (Bing/Google) as fallback contexts.
4. **Caching Layer**: Frequently visited pages (e.g., Wikipedia, major news portals) are cached locally to bypass external network requests.

---

## References

1. **RAG Engines**: Lewis, P., et al. (2020). *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks*. NeurIPS.
2. **Reciprocal Rank Fusion**: Cormack, G. V., et al. (2009). *Reciprocal Rank Fusion in a Link-Based Multi-Retrieval Environment*. ACM SIGIR.
3. **Search Engine Architecture**: Brin, S., & Page, L. (1998). *The Anatomy of a Large-Scale Hypertextual Web Search Engine*. Computer Networks.
