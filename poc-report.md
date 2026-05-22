# PoC Report: dejavu — Local-First AI Memory Layer

## 1. Executive Summary

The **Deja Vu** project — a local-first AI memory layer for agents and assistants — was evaluated for containerized deployment on OpenShift. The PoC objective was to prove that the core Python REST API server could be containerized, deployed on Kubernetes, and exercised end-to-end (health check, memory addition, semantic search, and memory listing). **The PoC partially succeeded:** the server started and passed the health check, but 3 of 4 functional test scenarios failed due to HTTP redirect issues (307), missing routes (404), and internal server errors (500). These failures indicate that while the containerization and deployment pipeline works, the API contract requires closer alignment with the actual server's routing conventions and likely needs a valid LLM API key for memory extraction to function.

---

## 2. Project Analysis

- **Repository URL:** `https://github.com/JSingletonAI/dejavu`
- **Local Path:** `/workspace/dejavu`
- **Project Name:** dejavu

### Repository Summary

Deja Vu is a local-first AI memory layer for agents and assistants. It provides a Python SDK, CLI, REST API, and MCP server interface backed by SQLite storage and the Venice LLM API for memory extraction and search. The repository is a monorepo containing the core Python library, a Node.js CLI implementation, a Python CLI implementation, and a separate `dejavu-openclaw` TypeScript plugin/tool package.

### Components Detected

| Component | Language | Build System | ML Workload | Port |
|-----------|----------|-------------|-------------|------|
| dejavu-core | Python | pip | Yes | 8765 |

### Project Classification

- **Type:** LLM Application (`llm-app`)
- **Technologies & Frameworks:**
  - Python 3.x
  - FastAPI / Uvicorn (REST API server)
  - SQLAlchemy with SQLite (persistence)
  - OpenAI SDK (Venice-compatible LLM backend)
  - Qdrant Client (optional vector store)
  - Typer (CLI)
  - Pydantic (data validation)

---

## 3. PoC Objectives

### What We Set Out to Prove

1. The Deja Vu Python core can be **containerized** and deployed as a REST API server on OpenShift.
2. The `/health` endpoint confirms the server starts correctly and is ready to accept requests.
3. The memory addition endpoint (`/v1/memories/`) accepts text input and persists memories to the local SQLite store.
4. The memory search endpoint (`/v1/memories/search/`) performs semantic search and returns relevant results.
5. The memory retrieval endpoint (`/v1/memories/`) lists stored memories, confirming persistence across requests.

### Relevance to Open Data Hub / OpenShift AI

Deja Vu demonstrates how a **memory service** can be containerized, deployed as a microservice, and integrated with LLM-powered workloads running on the platform. Its support for multiple vector stores (Qdrant, ChromaDB, FAISS, Milvus, PGVector, etc.) and LLM backends aligns with ODH's model serving and data pipeline infrastructure. As a building block for agentic AI systems, it represents a class of applications increasingly common in enterprise AI deployments.

### Infrastructure Requirements Identified

| Requirement | Value |
|-------------|-------|
| Inference Server | None (built-in FastAPI server) |
| Vector Database | None (SQLite default) |
| Embedding Model | None bundled (offloaded to LLM API) |
| GPU Required | No |
| Persistent Storage | 1Gi PVC at `$HOME/.dejavu` |
| Resource Profile | Small (256Mi RAM, 250m CPU) |
| Sidecar Containers | None |
| Extra Environment Variables | `OPENAI_API_KEY` (required), `OPENAI_BASE_URL` (required) |

---

## 4. Pipeline Execution

### Intake Phase

The intake phase analyzed the repository and identified a single deployable component: `dejavu-core`, a Python application using `pip` as its build system. The project was classified as an LLM application (`llm-app`) with a FastAPI-based REST server that listens on port **8765**. The suggested entrypoint was `dejavu serve --host 0.0.0.0 --port 8765`.

### PoC Plan Phase

The plan defined 4 HTTP-based test scenarios targeting the REST API:

1. **Health Check** — `GET /health`
2. **Add Memory** — `POST /v1/memories/`
3. **Search Memory** — `POST /v1/memories/search/`
4. **Get All Memories** — `GET /v1/memories/`

Infrastructure was kept minimal: a single Deployment, a Service, a PVC for SQLite persistence, and a Secret for the required API keys.

### Fork Phase

The repository was forked to GitLab for artifact storage. Build artifacts and test outputs are stored on the `autopoc-artifacts` branch.

### Containerize Phase

A Dockerfile was generated for the `dejavu-core` component, packaging the Python application with all dependencies and configuring the entrypoint as `dejavu serve --host 0.0.0.0 --port 8765`.

**Dockerfiles generated:**
- `Dockerfile.dejavu-core`

### Build Phase

The container image was built and pushed to the registry.

**Images built and pushed:**
- `quay.io/aicatalyst/dejavu-dejavu-core:latest`

**Build retries:** 1 (indicating a minor issue was resolved automatically during the build)

### Deploy Phase

All Kubernetes resources were deployed successfully on the first attempt.

**Resources deployed:**
| Resource | Name |
|----------|------|
| Namespace | `dejavu` |
| Secret | `dejavu-core-secrets` |
| PersistentVolumeClaim | `dejavu-core-data` (1Gi) |
| Deployment | `dejavu-core` |
| Service | `dejavu-core` |

**Service URL:** `http://172.30.11.64:8765`

**Deploy retries:** 0

### PoC Execute Phase

A test script (`poc_test.py`) was generated and executed against the deployed service. The script ran 4 HTTP-based test scenarios with retry logic (5 attempts per scenario).

---

## 5. Test Results

| Scenario | Status | Duration | Details |
|----------|--------|----------|---------|
| health-check | ✅ PASS | 0.0s | `{"status":"ok","name":"dejavu"}` |
| add-memory | ❌ FAIL | 40.0s | HTTP Error 307 after 5 attempts |
| search-memory | ❌ FAIL | 40.0s | HTTP Error 404 after 5 attempts: `{"detail":"Not Found"}` |
| get-all-memories | ❌ FAIL | 40.9s | HTTP Error 500 after 5 attempts: Internal Server Error |

**Overall: 1/4 passed, 3/4 failed**

### Failure Analysis

#### Scenario: add-memory (HTTP 307 Temporary Redirect)

**What went wrong:** The `POST /v1/memories/` endpoint returned a **307 Temporary Redirect**. This is a common FastAPI behavior when a request is made to a URL with a trailing slash and the route is defined without one (or vice versa). The test client likely did not follow the redirect automatically, or the redirect target was unreachable.

**Root Cause:** FastAPI's `redirect_slashes` behavior. The actual route may be `/v1/memories` (without trailing slash), or the test client configuration did not follow redirects.

**Suggested Fix:**
- Inspect the actual FastAPI route definitions in the Deja Vu source to determine the canonical URL (with or without trailing slash).
- Update the test script to either use the correct URL or enable automatic redirect following (`allow_redirects=True` in requests).
- Alternatively, configure FastAPI with `redirect_slashes=False` in the containerized deployment.

#### Scenario: search-memory (HTTP 404 Not Found)

**What went wrong:** The endpoint `POST /v1/memories/search/` returned a **404 Not Found**, indicating the route does not exist on the running server.

**Root Cause:** The API route for search may differ from what was assumed in the PoC plan. The actual endpoint could be `/v1/memories/search` (no trailing slash), `/v1/search/`, or use a query parameter on the memories endpoint.

**Suggested Fix:**
- Review the Deja Vu source code or hit the `/docs` (Swagger UI) or `/openapi.json` endpoint to discover the actual API schema.
- Update test scenarios to match the real API contract.

#### Scenario: get-all-memories (HTTP 500 Internal Server Error)

**What went wrong:** The `GET /v1/memories/` endpoint returned a **500 Internal Server Error**, suggesting an unhandled exception on the server side.

**Root Cause:** Most likely, the server attempted to use the `OPENAI_API_KEY` and/or `OPENAI_BASE_URL` environment variables to make an LLM API call during memory retrieval (e.g., for embedding generation), and the values provided were invalid, missing, or pointed to an unreachable endpoint. The SQLite database or configuration may also not have been fully initialized.

**Suggested Fix:**
- Check the pod logs (`kubectl logs -n dejavu deployment/dejavu-core`) for the full stack trace.
- Provide valid `OPENAI_API_KEY` and `OPENAI_BASE_URL` values pointing to a real or mock LLM endpoint.
- Consider deploying a mock LLM API server as a sidecar to isolate the test from external dependencies.

---

## 6. Infrastructure Deployed

### Kubernetes Namespace

```
dejavu
```

### Container Images

| Image | Tag |
|-------|-----|
| `quay.io/aicatalyst/dejavu-dejavu-core` | `latest` |

### Kubernetes Resources

| Kind | Name | Details |
|------|------|---------|
| Namespace | `dejavu` | Dedicated namespace for the PoC |
| Secret | `dejavu-core-secrets` | Contains `OPENAI_API_KEY`, `OPENAI_BASE_URL` |
| PersistentVolumeClaim | `dejavu-core-data` | 1Gi, mounted at `$HOME/.dejavu` |
| Deployment | `dejavu-core` | 1 replica, `dejavu serve --host 0.0.0.0 --port 8765` |
| Service | `dejavu-core` | ClusterIP, port 8765 |

### Service URLs / Routes

| Service | URL |
|---------|-----|
| dejavu-core | `http://172.30.11.64:8765` |

### Resource Allocations

| Resource | Request | Limit |
|----------|---------|-------|
| CPU | 250m | — |
| Memory | 256Mi | — |

### Persistent Volumes

| PVC | Size | Mount Path | Purpose |
|-----|------|------------|---------|
| `dejavu-core-data` | 1Gi | `$HOME/.dejavu` | SQLite database and configuration |

---

## 7. Recommendations

### Production Readiness

**Not production-ready.** The PoC revealed that 3 of 4 API scenarios failed. While the server starts and responds to health checks, the core memory functionality (add, search, list) could not be validated. Key gaps:

- **API contract mismatch:** Route definitions need to be verified against the actual OpenAPI schema.
- **External dependency:** The server requires a valid LLM API key to function — this must be provisioned and managed securely.
- **Error handling:** The 500 error on `GET /v1/memories/` indicates inadequate error handling when the LLM backend is unavailable.

### Performance

- The health check responded in under 1 second, indicating a lightweight server startup.
- The 40-second timeout on failed scenarios was consumed by retries against broken endpoints, not by actual processing latency.
- In production, LLM API call latency will dominate response times. Consider implementing async processing or caching for frequently accessed memories.

### Security

- **API Keys:** `OPENAI_API_KEY` must be stored in Kubernetes Secrets (already done) and rotated regularly.
- **Network Policy:** The service is currently accessible within the cluster via ClusterIP. For external access, implement proper authentication (API keys, OAuth2) and TLS termination.
- **SQLite:** Not suitable for multi-replica deployments. Consider PostgreSQL with PGVector for production.
- **Container Security:** Run as non-root user; apply read-only root filesystem where possible.

### Scalability

- **Single-replica limitation:** SQLite is a single-writer database. Scaling to multiple replicas requires migrating to PostgreSQL/PGVector or another networked database.
- **Horizontal scaling:** Once backed by a shared database, the stateless API server can scale horizontally behind a load balancer.
- **Vector search:** For large-scale deployments, migrate from local/SQLite-based vector search to Qdrant, Milvus, or PGVector running as a dedicated service.

### Next Steps

1. **Fix API contract:** Hit the `/docs` or `/openapi.json` endpoint on the running server to discover actual routes, then update test scenarios accordingly.
2. **Provide valid LLM credentials:** Configure a real or mock OpenAI-compatible endpoint (e.g., vLLM, Ollama, or a Venice API key) and re-run the PoC.
3. **Investigate 500 error:** Check pod logs for the stack trace and address the root cause.
4. **Re-run PoC:** After fixing the above, re-execute the test suite to validate all 4 scenarios.
5. **Add OpenAPI-driven tests:** Auto-generate test cases from the server's OpenAPI spec to ensure full coverage.
6. **Production database migration:** Evaluate PGVector as a drop-in replacement for SQLite to enable multi-replica deployments.

---

## 8. Open Data Hub / OpenShift AI Considerations

### Relevant ODH Components

| ODH Component | Relevance | Notes |
|---------------|-----------|-------|
| **Model Serving (vLLM/KServe)** | High | Deploy an LLM endpoint that Deja Vu can call for memory extraction and embedding, replacing the external Venice API dependency |
| **Data Science Pipelines** | Medium | Orchestrate batch memory ingestion workflows — e.g., bulk-loading conversation histories into the memory store |
| **Model Registry** | Low | Track embedding model versions used by the memory layer |
| **Workbenches** | Medium | Develop and test Deja Vu integrations with custom agents in Jupyter environments |
| **TrustyAI** | Low | Monitor memory extraction quality and detect drift in LLM-generated memory summaries |

### Migration Path: Vanilla K8s → ODH-Managed

1. **Phase 1 — Current state:** Vanilla Deployment + Service on OpenShift. External LLM API dependency.
2. **Phase 2 — Internalize LLM:** Deploy an LLM model via **KServe/vLLM** on ODH. Point Deja Vu's `OPENAI_BASE_URL` to the internal KServe endpoint. This eliminates the external API dependency and keeps all traffic in-cluster.
3. **Phase 3 — Vector DB:** Deploy **Qdrant** or **Milvus** via ODH operators. Configure Deja Vu to use the in-cluster vector store for production-grade semantic search.
4. **Phase 4 — Pipeline integration:** Use **Data Science Pipelines** to automate memory ingestion from conversation logs, CRM systems, or document stores.
5. **Phase 5 — Monitoring:** Integrate **TrustyAI** to monitor memory extraction quality and alert on degradation.

### ODH-Specific Features to Leverage

- **KServe InferenceService:** Host the embedding model and LLM for memory extraction as managed inference endpoints with autoscaling and canary deployments.
- **PVC-backed workbenches:** Use Jupyter workbenches for interactive development of custom memory extraction prompts and evaluation.
- **S3-compatible storage:** Store memory export snapshots in ODH-managed MinIO for backup and disaster recovery.

---

## 9. Appendix

### Artifact Links

| Artifact | Location |
|----------|----------|
| PoC Plan | `poc-plan.md` |
| Test Script | `/workspace/dejavu/poc_test.py` |
| Dockerfile | `Dockerfile.dejavu-core` |
| K8s Manifests | Generated during deploy phase |
| Raw Test Output | `poc-test-output/` on `autopoc-artifacts` branch |

### Build Issues

- **Build retries:** 1 retry was required during the image build phase. The initial build likely failed due to a dependency resolution issue or missing system package, which was automatically corrected on retry.

### Deploy Issues

- **Deploy retries:** 0 — deployment succeeded on the first attempt.

### Test Execution Errors

| Scenario | Error Type | HTTP Code | Response Body |
|----------|-----------|-----------|---------------|
| add-memory | Redirect | 307 | (empty) |
| search-memory | Not Found | 404 | `{"detail":"Not Found"}` |
| get-all-memories | Server Error | 500 | `Internal Server Error` |

### Environment Variables Required

```bash
OPENAI_API_KEY=<valid-api-key>        # Required for LLM-backed memory extraction
OPENAI_BASE_URL=<api-base-url>       # e.g., https://api.venice.ai/api/v1 or internal KServe URL
```

### Useful Debugging Commands

```bash
# Check pod status
kubectl get pods -n dejavu

# View server logs
kubectl logs -n dejavu deployment/dejavu-core

# Access the OpenAPI documentation
kubectl port-forward -n dejavu svc/dejavu-core 8765:8765
# Then open: http://localhost:8765/docs

# Check the actual API routes
curl http://172.30.11.64:8765/openapi.json | jq '.paths | keys'
```
