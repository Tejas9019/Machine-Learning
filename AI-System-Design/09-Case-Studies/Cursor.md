# Case Study: Cursor (AI-First Code Editor)

## Overview

**Cursor** is an AI-first code editor built as a fork of VS Code. It integrates LLM intelligence directly into the developer workflow. Cursor's architecture relies on local AST-based codebase parsing, incremental indexing, hybrid semantic search, and a specialized inline Diff Execution Engine that stream edits directly into active files.

---

## Problem Statement

Designing an AI-first coding assistant introduces specific client-side and system-level challenges:
1. **Whole-Codebase Context**: Standard code completion engines operate on single files. Answering codebase-wide questions (e.g., "how is this API utility used elsewhere?") requires indexing and searching thousands of files across different folder layers.
2. **Resource Constraints**: Parsing and computing embeddings for a large codebase on a developer's local laptop can exhaust CPU and memory resources, degrading editor performance.
3. **Structured Edit Application**: Standard LLMs output code as raw markdown text. Parsing this text and applying it as accurate inline insertions or deletions (diffs) without breaking syntax requires specialized parser tools.

---

## High-Level System Architecture

Cursor's architecture splits operations between a local client editor interface and a secure cloud inference server:

```
  [Developer Laptop (Local Editor)]
  ├─ Code Editor UI (VS Code Fork)
  ├─ AST Parser (Tree-sitter)
  ├─ Local Vector Store (HNSWlib) ──> Persists local embeddings
  │
  │ (Encrypted HTTPS REST/SSE)
  ▼
  [Cursor Cloud Gateway]
  │
  ├───────────────────────────┴───────────────────────────┐
  ▼                                                       ▼
[Context aggregation pipeline]                   [Diff Parser / Model Pool]
(Merges local AST + active tabs + queries)       (Translates streaming tokens
  │                                               into editor instructions)
  ▼                                                       │
[Inference Providers (OpenAI/Anthropic/Custom)] <────────┘
```

### 1. Local Codebase Indexing & AST Parsing
- **AST Parsing (Tree-sitter)**: Instead of chunking code files by arbitrary line counts (which cuts functions in half), Cursor uses AST parsers to understand code structure. Code is split logically along class, function, and method boundaries.
- **Incremental Indexing**: When a developer opens a project, Cursor parses the files, computes embeddings (locally or via API), and saves them to a local vector index (HNSWlib). When files change, Cursor uses file-watchers to update *only* the modified files, preventing CPU starvation.

---

### 2. Context Construction Pipeline
To answer queries, Cursor gathers context dynamically:
- **Active Document State**: Aggregates the cursor line location, active file tabs, and imported dependency files.
- **Semantic Retrieval**: Queries the local vector store using the developer's prompt to pull the top $K$ relevant code snippets from other files.
- **Code Graph Traversal**: Traverses references and definitions of code symbols under the cursor, building a structured context block containing the relevant declarations.

---

### 3. Inline Diff Execution Engine
- When using inline edits (Ctrl/Cmd + K), the model stream completes replacement code.
- **Diff Parser**: Instead of waiting for the full model response, a local parser parses the streaming tokens. It identifies formatting blocks (e.g., Unified Diff format `@@ -1,4 +1,4 @@`) and applies edits in the editor UI as real-time green/red insertions and deletions.

---

## Design Decisions & Trade-offs

### Local Vector Store vs. Cloud Codebase Indexing

- **Local Vector Store (HNSWlib on Laptop)**:
  * *Pros*: High privacy. Codebase embeddings remain on the developer's machine. Zero cloud hosting costs for the database. Works offline.
  * *Cons*: Heavy CPU/RAM usage on the host laptop. Large project codebases can exhaust local storage.
- **Cloud Codebase Indexing**:
  * *Pros*: Offloads CPU/RAM to cloud servers. Supports advanced background compilation and larger search indexes.
  * *Cons*: Requires uploading the developer's entire private codebase to cloud servers (privacy risk). High cloud database hosting costs.

---

## Interview Questions

### Q1: How does Cursor build context for a codebase query like "Where do we instantiate the DB connection?"
**Answer**:
1. **User Query**: The developer submits the prompt.
2. **Semantic Search**: The query is embedded, and Cursor performs an ANN search on the local HNSWlib index to find the top $N$ code snippets with high semantic similarity.
3. **AST Symbol Analysis**: Cursor parses the query for code symbols (e.g., class names like `DatabaseConnection`). If found, it uses the editor's Language Server Protocol (LSP) definitions to locate the file and line number where that symbol is declared.
4. **Active State Context**: Merges active files, recent cursor locations, and open editor tabs.
5. **Prompt Synthesis**: Assembles these components into a structured XML/Markdown prompt layout, prioritizing active files first, followed by LSP symbol definitions, and finally semantic search snippets, before sending the request to the LLM.

### Q2: How does Cursor apply real-time inline diffs to a file during streaming?
**Answer**:
1. The client sends the code block along with the instruction to the server.
2. The server model is prompted to return output in a structured search-and-replace block format (specifying target lines to delete and new lines to insert).
3. As tokens stream back to the editor, a **local parser** reads the tokens in real time, detecting block start markers (e.g., `<<<<<<< SEARCH` and `=======`).
4. As soon as the search block matches a contiguous segment in the active file, Cursor displays the red "deletion" diff. As the replace tokens arrive, it renders the green "insertion" diff, applying edits incrementally instead of waiting for the entire generation to finish.

---

## References

1. **Tree-sitter AST Parser**: *Tree-sitter: An incremental parsing system for programming tools*. (Tree-sitter Project).
2. **VS Code Architecture**: *VS Code Extension API: Language Server Extension Guide*. (VS Code Documentation).
3. **Code Semantic Search**: *CodeSearchNet Challenge: Evaluating Semantic Code Search*. (GitHub Engineering).
