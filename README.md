# Overview
- MCP RAG Server `mcp-rag-server` exposes document ingestion and knowledge retrieval capabilities using the **Model Context Protocol (MCP)**. It is a reusable knowledge service, not an LLM chat application. 
- A Streamlit-based chat application `mcp-rag-chat-demo` that answers questions using documents indexed by a separate **MCP RAG Server**. The application communicates with the server over the **Model Context Protocol (MCP), Streamable HTTP transport**, and uses an LLM to formulate responses from retrieved context.
- `ai-projects` with a compose.yml file to orchestrate `mcp-rag-server` and `mcp-rag-chat-demo` to make standard, robust, and effective pattern for local development, integration testing, and simple single-host deployments.

## Simple Architectural Flow

```text
Browser
   |
   v
Streamlit UI (mcp-rag-chat-demo)
   |
   +---- LLM provider (answer generation)
   |
   +---- MCP Client -- Streamable HTTP --> MCP RAG Server
                                           |
                                           +-- search_documents tool
                                           +-- document embeddings / retrieval
                                           +-- LanceDB persistent store
```

## Run with Docker Compose

The sibling MCP server and chat demo are orchestrated from the parent `ai-projects` directory.

```text
ai-projects/
├── compose.yaml
├── mcp-rag-server/
└── mcp-rag-chat-demo/
```

- `mcp-rag-server` repository: https://github.com/jam1179/mcp-rag-server

- `mcp-rag-chat-demo` repository: https://github.com/jam1179/mcp-rag-chat-demo

From that parent directory:

```powershell
docker compose up -d
docker compose ps
```

Open:

- **Chat UI:** <http://localhost:8501>
- **MCP endpoint (host):** <http://localhost:8000/mcp>

Inspect logs:

```powershell
docker compose logs --tail 100 mcp-rag-chat-demo
docker compose logs --tail 100 mcp-rag-server
```

Stop the stack while **retaining indexed documents**:

```powershell
docker compose down
```

**Do not run `docker compose down -v`** unless you intentionally want to remove the named LanceDB volume and its indexed data.

## Example workflow

1. Start the MCP server and the Streamlit application.
2. Confirm the MCP server already contains indexed documents (for example, `Helm.pdf`).
3. Open the Streamlit UI.
4. Ask: **“<your queries>”**
5. Review the response against the document.

Documents are ingested by the MCP server, not by the chat application's local filesystem.