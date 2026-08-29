# Kestra Flow Files Explanation Index

This folder contains copies of Kestra's YAML flow definitions annotated with line-by-line comments to explain their architecture and options. Below is an overview of the progression of flows, from basic LLM prompts to autonomous multi-agent orchestration.

---

## Flow Overview Matrix

| File Name | Flow ID | Key Concept | Core Plugins Used |
| :--- | :--- | :--- | :--- |
| [`1_chat_without_rag.yaml`](./1_chat_without_rag.yaml) | `1_chat_without_rag` | LLM training data limitations | `io.kestra.plugin.ai.completion.ChatCompletion` |
| [`2_chat_with_rag.yaml`](./2_chat_with_rag.yaml) | `2_chat_with_rag` | Static Document RAG & Vector Embeddings | `IngestDocument`, `KestraKVStore`, `ChatCompletion` |
| [`3_rag_with_websearch.yaml`](./3_rag_with_websearch.yaml) | `3_rag_with_websearch` | Dynamic Search RAG | `TavilyWebSearch` content retriever |
| [`4_simple_agent.yaml`](./4_simple_agent.yaml) | `4_simple_agent` | Chained Agents & Token Logging | `AIAgent`, `pluginDefaults` |
| [`5_web_research_agent.yaml`](./5_web_research_agent.yaml) | `5_web_research_agent` | Autonomous Tool Use (MCP & Filesystem) | `AIAgent`, `DockerMcpClient` (filesystem tool) |
| [`6_multi_agent_research.yaml`](./6_multi_agent_research.yaml) | `6_multi_agent_research` | Multi-Agent delegation & JSON parsing | Nested `AIAgent` tools, Pebble JSON extraction |

---

## Detailed Architectural Explanations

### 1. [1_chat_without_rag.yaml](./1_chat_without_rag.yaml)
*   **Purpose**: Acts as a baseline demonstration. When you run this flow, the model (`gemini-3.5-flash`) tries to answer what features were released in Kestra 1.1 based purely on its training cutoff weights. 
*   **Outcome**: The response is generic, outdated, or hallucinated because it lacks context.

### 2. [2_chat_with_rag.yaml](./2_chat_with_rag.yaml)
*   **Purpose**: Demonstrates static Retrieval-Augmented Generation (RAG).
*   **Mechanism**:
    1.  **Ingestion**: `IngestDocument` reads the Kestra 1.1 release blog from GitHub.
    2.  **Vectorization**: Uses `gemini-embedding-001` to turn text blocks into vector coordinates.
    3.  **Storage**: Persists vectors in Kestra's local Key-Value store (`KestraKVStore`).
    4.  **Querying**: `ChatCompletion` queries the vector store for semantic matches and injects them as prompt context.

### 3. [3_rag_with_websearch.yaml](./3_rag_with_websearch.yaml)
*   **Purpose**: Demonstrates real-time, dynamic RAG.
*   **Mechanism**: Instead of database embeddings, it runs a live search queries using the `TavilyWebSearch` content retriever. The search results are parsed and automatically fed into the model prompt context before execution.

### 4. [4_simple_agent.yaml](./4_simple_agent.yaml)
*   **Purpose**: Introduces basic autonomous agent behaviors.
*   **Key Concepts**:
    *   **Prompt Chaining**: The output text of `multilingual_agent` is directly mapped into the input parameter of `english_brevity`.
    *   **Token Metrics**: Tracks token usage values (`outputs.agent_id.tokenUsage.totalTokenCount`) to assist with cost estimation.
    *   **pluginDefaults**: Eliminates redundant YAML declarations by defining the AI provider details globally at the bottom of the flow.

### 5. [5_web_research_agent.yaml](./5_web_research_agent.yaml)
*   **Purpose**: Demonstrates autonomous tool use.
*   **Key Concepts**:
    *   **Goal-Oriented Execution**: You define the final goal in the prompt, and the agent decides how many search queries to perform, structures the report, and writes to a file.
    *   **Docker MCP (Model Context Protocol) Client**: Mounts a Docker container running the `mcp/filesystem` tool inside the Kestra environment. This gives the agent programmatic permissions to create, write, and read files in Kestra's temporary execution directory.
    *   **outputFiles**: Directs Kestra to track and retrieve `research_report.md` as an output artifact.

### 6. [6_multi_agent_research.yaml](./6_multi_agent_research.yaml)
*   **Purpose**: Demonstrates multi-agent systems using delegation.
*   **Mechanism**:
    *   **Orchestration**: The `analysis` agent acts as the coordinator (manager).
    *   **Delegation**: The coordinator has a sub-agent nested under its `tools` block. When it needs updated information about a company, it executes the nested sub-agent tool to run Tavily searches and return summaries.
    *   **Output Parsing**: The coordinator is instructed to generate raw JSON text. In Task 2 (`parse_results`), Pebble template loops parse the text string back into structured objects (e.g. `{{ json(outputs.analysis.textOutput).company }}`) for logging.
