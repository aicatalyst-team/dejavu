# PoC Plan: dejavu

## Project Classification
- **Type:** llm-app
- **Key Technologies:** Python, FastAPI, Uvicorn, SQLAlchemy (SQLite), OpenAI SDK (Venice-compatible), Qdrant Client, Typer CLI, Pydantic
- **ODH Relevance:** Deja Vu is a local-first AI memory layer that can serve as a persistent memory backend for AI agents and assistants. In an ODH/OpenShift AI context, it demonstrates how a memory service can be containerized, deployed as a microservice, and integrated with LLM-powered workloads running on the platform. The project's support for multiple vector stores (Qdrant, ChromaDB, FAISS, Milvus, PGVector, etc.) and LLM backends aligns with ODH's model serving infrastructure.

## PoC Objectives
What we want to prove:
1. The Deja Vu Python core can be containerized and deployed as a REST API server on OpenShift
2. The `/health` endpoint confirms the server starts correctly and is ready to accept requests
3. The memory addition endpoint (`/v1/memories/`) accepts text input and persists memories to the local SQLite store
4. The memory search endpoint (`/v1/memories/search/`) performs semantic search and returns relevant results
5. The memory retrieval endpoint (`/v1/memories/`) lists stored memories, confirming persistence across requests

## Infrastructure Requirements
- **Inference Server:** none (the project includes its own FastAPI-based REST server via `dejavu serve`)
- **Vector Database:** none (uses SQLite locally by default; Qdrant client is a dependency but the default config uses local/in-memory storage)
- **Embedding Model:** none bundled — embeddings are handled via the LLM API (Venice/OpenAI-compatible endpoint)
- **GPU Required:** no
- **Persistent Storage:** 1Gi PVC mounted at `/root/.dejavu` (or `$HOME/.dejavu`) to persist the SQLite database and config
- **Resource Profile:** small (256Mi RAM, 250m CPU) — the server is lightweight; LLM computation is offloaded to the external API
- **Sidecar Containers:** none

## Test Scenarios

### Scenario 1: Health Check
- **Description:** Verify the Deja Vu REST API server is running and healthy
- **Type:** http (GET)
- **Endpoint:** `/health`
- **Input:** None
- **Expected:** Returns HTTP 200 with a JSON health status object
- **Timeout:** 60 seconds

### Scenario 2: Add Memory
- **Description:** Add a memory entry via the REST API and verify it is accepted and processed by the LLM-backed memory extraction engine
- **Type:** http (POST)
- **Endpoint:** `/v1/memories/`
- **Input:** `{"messages": [{"role": "user", "content": "I prefer concise technical explanations over verbose ones."}], "user_id": "test-user"}`
- **Expected:** Returns HTTP 200 with confirmation that the memory was stored, including memory IDs in the response
- **Timeout:** 60 seconds (LLM extraction may take a few seconds)

### Scenario 3: Search Memory
- **Description:** Search for the previously added memory to verify semantic retrieval works end-to-end
- **Type:** http (POST)
- **Endpoint:** `/v1/memories/search/`
- **Input:** `{"query": "How should responses be written for me?", "user_id": "test-user"}`
- **Expected:** Returns HTTP 200 with search results containing the memory about concise technical explanations
- **Timeout:** 60 seconds

### Scenario 4: Get All Memories
- **Description:** Retrieve all memories for a user to verify persistence across requests
- **Type:** http (GET)
- **Endpoint:** `/v1/memories/?user_id=test-user`
- **Input:** None
- **Expected:** Returns HTTP 200 with a list of stored memories including the one previously added
- **Timeout:** 30 seconds

## Dockerfile Considerations

This is a FastAPI-based REST API server. The Dockerfile should:

- Use a Python 3.11 or 3.12 base image
- Install the `dejavu-memory` package from the repo root using `pip install .` (the `pyproject.toml` is in the repo root and uses hatchling as the build backend)
- The core dependencies are sufficient — no need for optional extras (`vector_stores`, `llms`, `extras`) for the basic PoC since the default configuration uses SQLite and OpenAI-compatible API calls (which Venice and our LLM proxy both support)
- Create the `~/.dejavu` directory in the container for SQLite storage
- **ENTRYPOINT** should be `["dejavu"]` and **CMD** should be `["serve", "--host", "0.0.0.0", "--port", "8765"]`
- **Add `EXPOSE 8765`** — the server listens on port 8765
- The project expects `OPENAI_API_KEY` and optionally `OPENAI_BASE_URL` environment variables at runtime (Venice API is OpenAI-compatible). These will be injected via the deployment manifest, not baked into the image.
- Note: The `dejavu` CLI entry point is defined in `pyproject.toml` under `[project.scripts]` as `dejavu = "dejavu.cli:app"`. The `serve` subcommand starts the FastAPI/Uvicorn server.

## Deployment Considerations

- **Deployment model:** Deploy as a Kubernetes **Deployment** with 1 replica. This is a long-running HTTP server.
- **Service:** Create a **Service** on port 8765. The server binds to `0.0.0.0:8765`.
- **Persistent Volume:** Mount a 1Gi PVC at `/root/.dejavu` (the default home directory in most container base images). This stores the SQLite database (`memories.db`) and config (`config.json`). Without this, memories are lost on pod restart.
- **Environment Variables:**
  - `OPENAI_API_KEY` — Required. This is the LLM API key. In the PoC, this should be pointed at the internal LLM proxy or a Venice API key.
  - `OPENAI_BASE_URL` — Required for PoC. Set this to the internal LLM proxy URL so the project routes LLM calls through the proxy instead of directly to Venice/OpenAI.
- **LLM API dependency:** The project uses the OpenAI SDK (`openai` package) to make LLM calls. By default it targets Venice's API, but since it uses `OPENAI_BASE_URL`, it can be redirected to any OpenAI-compatible endpoint including our internal LLM proxy. The deploy agent should inject the LLM proxy endpoint.
- **Testing:** Test via HTTP requests to the deployed Service. Run the 4 test scenarios in order (health → add → search → get) since scenarios 3 and 4 depend on scenario 2 having added data.
- **Init:** The server may need an initial `dejavu init` call or the config may need to be pre-seeded. However, the REST server should auto-initialize on first use. If not, a Kubernetes init container running `dejavu init` can be added.
- **No GPU needed.** All ML computation (embeddings, LLM reasoning) is offloaded to the external LLM API.