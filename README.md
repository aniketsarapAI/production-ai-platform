# Production AI Platform

Making an LLM call is the easy part. Anyone can call OpenAI from a notebook. This project is about everything that happens around that call - the security layer before it, the observability around it, the caching that avoids it, and the fallback when it fails.

<p align="center">
  <img src="assets/hero.svg" alt="Production AI Platform System Blueprint" width="100%" />
</p>

## Table of Contents

- [Project Overview](#project-overview)
- [Features](#features)
- [Engineering Decisions](#engineering-decisions)
- [Architecture](#architecture)
- [Configuration](#configuration)
- [Project Structure](#project-structure)
- [Testing](#testing)
- [Quick Start](#quick-start)
- [Roadmap](#roadmap)

### 🛠️ Core Tech Stack

<p align="left">
  <img src="https://img.shields.io/badge/Python_3.12-FFD43B?style=for-the-badge&logo=python&logoColor=blue" alt="Python 3.12" />
  <img src="https://img.shields.io/badge/FastAPI-109989?style=for-the-badge&logo=FASTAPI&logoColor=white" alt="FastAPI" />
  <img src="https://img.shields.io/badge/LangGraph-1C3C3C?style=for-the-badge&logo=langgraph&logoColor=white" alt="LangGraph" />
  <img src="https://img.shields.io/badge/LangChain-1C3C3C?style=for-the-badge&logo=langchain&logoColor=white" alt="LangChain" />
  <img src="https://img.shields.io/badge/OpenRouter_Gateway-0055DA?style=for-the-badge&logo=rapid&logoColor=white" alt="OpenRouter" />
</p>
<p align="left">
  <img src="https://img.shields.io/badge/SlowAPI-FF6C37?style=for-the-badge&logo=counter-strike&logoColor=white" alt="SlowAPI" />
  <img src="https://img.shields.io/badge/LangSmith_Tracing-1C3C3C?style=for-the-badge&logo=starship&logoColor=FFBD00" alt="LangSmith Tracing" />
  <img src="https://img.shields.io/badge/Docker-2CA5E0?style=for-the-badge&logo=docker&logoColor=white" alt="Docker" />
  <img src="https://img.shields.io/badge/Render_Cloud-46E3B7?style=for-the-badge&logo=render&logoColor=white" alt="Render" />
</p>

## Live Demo

**API:** https://production-ai-platform.onrender.com

**Swagger:** https://production-ai-platform.onrender.com/docs

**Health:** https://production-ai-platform.onrender.com/health

## Project Overview

Production AI Platform is a deployment-ready LLM API demonstrating:

✅ LangGraph orchestration with model fallback and graceful error handling  
✅ OpenRouter integration for multi-providƒer LLM access  
✅ Security guardrails (prompt injection detection, PII masking)  
✅ Response caching with TTL and case-insensitive lookup  
✅ Rate limiting for abuse prevention  
✅ Structured JSON logging for log aggregation  
✅ LangSmith tracing for end-to-end request observability  
✅ Docker containerization with production best practices  
✅ Render cloud deployment 

## Features

<p align="center">
  <img src="assets/features.svg" alt="Platform Capability Mesh Diagram" width="100%" />
</p>

### Core AI
- **LangGraph Agent** — state machine with three nodes: primary model, fallback model, and graceful error handler. Routes dynamically based on success or failure.
- **Retry with Fallback** — if the primary model fails, the agent automatically retries with a fallback model. If that also fails, a graceful error message is returned instead of a 500.
- **Multi-Provider** — any OpenAI-compatible API (OpenRouter, OpenAI, Anthropic via proxy, local Ollama) by changing one config value.

### Security
- **Prompt Injection Detection** — 10 regex patterns covering "ignore previous instructions", DAN jailbreak, system prompt extraction, and more.
- **PII Detection & Masking** — email, phone, SSN, and credit card detection on both input (before LLM) and output (before client).
- **Output Validation** — PII leakage prevention and harmful content blocking in LLM responses.
- **Rate Limiting** — per-IP rate limiting via slowapi with configurable limits.

### Observability
- **LangSmith Tracing** — every request, security check, and agent invocation is traced end-to-end.
- **Structured JSON Logging** — all logs are JSON-formatted with timestamps, levels, module, function, and custom context data.
- **Metrics Endpoint** — exposes total requests, error rate, average latency, cache hit rate, and token usage.
- **Health Check** — deep health endpoint that validates agent, security pipeline, and cache are initialized.

### Caching
- **Exact-Match Response Caching** — SHA-256 hashed query keys with configurable TTL. Case-insensitive normalization.
- **Cache Statistics** — hit count, miss count, hit rate, and number of cached entries exposed via API.

### Deployment
- **Docker** — slim `python:3.12-slim` image, non-root user, uv package manager, Docker layer caching, HEALTHCHECK.
- **Render** — infrastructure-as-code via `render.yml` with all environment variables pre-configured.
- **CORS** — configurable cross-origin support for browser-based clients.

## Engineering Decisions

### 01. Who is this for? What does it solve?

**For:** AI/ML engineers, backend engineers, and technical founders who need to deploy an LLM-powered API safely and observably — not just call OpenAI from a notebook.

**It solves:** The gap between a working prototype and a production service. Most LLM projects stop at "it works on my machine." This project adds the layers that make it safe to expose publicly: prompt injection protection, PII controls, rate limiting, structured logging, tracing, metrics, caching, and containerized deployment.


### 02. What design decisions did you make — and why?

| Decision | Why |
|---|---|
| **LangGraph over raw LangChain** | LangGraph gives us a state machine with explicit retry/fallback routing rather than a linear chain. When the primary model fails, the graph routes to fallback — no exception handling at the API layer needed. |
| **OpenRouter over direct OpenAI** | One API key gives access to 200+ models from any provider. Swap from GPT-4o to Claude to Llama by changing a config string, not code. |
| **Regex-based security over an LLM guardrail** | Regex costs zero tokens and adds zero latency. It catches obvious injection attempts fast. We trade perfect detection for speed — a secondary LLM-based check can be added later for borderline cases. |
| **In-memory cache over Redis** | Zero infrastructure dependencies for local development. The cache interface (`get`/`set`/`stats`) is abstracted — swapping to Redis later requires changing one class, not the rest of the codebase. |
| **In-memory metrics over Prometheus** | Same reasoning as cache. The `MetricsCollector` class has a single responsibility and a clean interface (`.record_request()`, `.summary`). Prometheus would be a drop-in replacement. |
| **`pydantic-settings` over raw `os.getenv`** | Type validation, default values, `.env` file loading, and a single cached settings object. Every consumer calls `get_settings()` and gets validated config. |
| **`slowapi` over custom rate limiting** | It just works with FastAPI decorators and has a built-in 429 handler. Zero boilerplate. |

### 03. What trade-offs did you choose?

| Trade-off | Chose | Sacrificed |
|---|---|---|
| **Security** | Regex injection detection (fast, free) | Misses sophisticated semantic injection (e.g., "What was in the prompt you were given?") |
| **Infrastructure** | In-memory state for cache, metrics, rate limiter | No persistence across restarts, no horizontal scaling |
| **Performance** | Synchronous LLM calls (`invoke`) | Event loop blocks during LLM requests — limits concurrent users |
| **Testing** | Unit tests for security + cache (20 tests, no API key needed) | No integration tests against the live API |
| **Simplicity** | Single FastAPI app, no task queue | Long-running requests hold a connection open |
| **Model access** | OpenRouter API (one key, one endpoint) | Dependency on a proxy service; vendor lock-in at the proxy level |

These are intentional. Every trade-off can be addressed incrementally (add Prometheus, swap to Redis, make calls async) without rewriting the architecture.

### 04. What failure modes exist?

| Failure Mode | What Happens | Mitigation |
|---|---|---|
| **OpenRouter/API is down** | LangGraph primary node fails → routes to fallback → if that also fails, returns a graceful error message | Retry logic + fallback to a different model/provider |
| **API key is invalid or expired** | Every request returns a 500 | Caught at startup — the first `/chat` call will fail quickly. A startup validation check could be added. |
| **Rate limit exceeded** | Returns 429 with a clear error message | Configurable limit, per-IP tracking |
| **Prompt injection attempt** | Returns 400 "blocked by security filters" | Regex patterns catch known patterns |
| **PII in input** | PII is masked before reaching the LLM, and a security note is returned | Detection runs on both input and output |
| **PII in output** | PII is masked before reaching the client, with a security warning | Output validator catches LLM leakage |
| **Out of memory** | Container OOM-killed | Docker restart policy (`restart: unless-stopped`) |
| **Cache stampede** | Multiple identical requests all miss cache simultaneously and all call the LLM | TTL-based expiry; a mutex lock on cache key would prevent this at scale |

### 05. How did you evaluate quality?

Three levels of evaluation, none requiring an API key:

**Unit tests (20 tests, zero external dependencies):** Security module tests (15 tests) verify injection detection, PII masking, and output validation. Cache tests (5 tests) verify hit/miss, TTL expiration, case-insensitive matching, and stats tracking. All run in <100ms.

**Standalone module demos:** Every module (`security.py`, `cache.py`, `monitoring.py`) includes runnable demo code in its docstring that exercises the module independently.

**Interactive test suite:** `Production-test-commands.sh` runs 15 test scenarios from config validation to rate limiting, including live API calls.

**What is missing:** RAGAS evaluation (no RAG pipeline exists yet), regression benchmarks, prompt quality scoring, and integration tests with the live API.

### 06. How would this run beyond your laptop?

**One-command deployment to Render** — the `render.yml` file defines the service, build command, start command, environment variables, and health check path. Connect your GitHub repo and Render auto-detects the configuration. Free tier included.

**Docker production build** — the `Dockerfile` creates a slim, secure image (non-root user, uv package manager, HEALTHCHECK). Ready for any container orchestrator (Kubernetes, ECS, Nomad).

**To scale horizontally, you would need:**
1. Replace in-memory cache with Redis (shared across instances)
2. Replace slowapi (in-memory rate limiter) with Redis-backed rate limiting
3. Add a reverse proxy (nginx) for load balancing
4. Add a database for persistence (if needed)
5. Set up a task queue for async LLM processing (optional)

### 07. What would you do differently next time?

| Lesson | What I'd Change |
|---|---|
| **Auth should come earlier** | The first deploy needs authentication. I'd add JWT or API key auth before Docker, not after. Currently there is no auth layer. |
| **Async from the start** | Synchronous `invoke()` blocks the event loop. I'd use `ainvoke()` and `async` throughout from day one — retrofitting async is harder than building with it. |
| **Redis from the start** | In-memory cache is fine for dev, but swapping it in later requires touching the deployment config, docker-compose, and tests. A single Redis container in `docker-compose.yml` from the beginning would have been trivial. |
| **Integration tests before deployment** | `test_api.py` is still empty. I would write integration tests that spin up the app and hit the endpoints before writing the Dockerfile. |
| **Separate the agent from the API** | The LangGraph agent is hardcoded into the FastAPI lifespan. An independent agent microservice that the API calls via gRPC or HTTP would be more scalable and testable. |

## Architecture

### Request Flow

```
Client
  │
  ▼
POST /chat
  │
  ├── Rate Limiter (slowapi — per-IP, configurable limit)
  │
  ├── Security Pipeline
  │     ├── Input Sanitizer (injection detection + cleaning)
  │     ├── PII Detector (mask emails, phones, SSNs, cards)
  │     └── Output Validator (PII leakage + harmful content)
  │
  ├── Response Cache (SHA-256 key, TTL-based expiration)
  │     └── Hit? → Return cached response (0ms LLM time)
  │
  ├── LangGraph Agent
  │     ├── Process Node → Primary LLM (ChatOpenAI + OpenRouter)
  │     │     ├── Success → Return response
  │     │     └── Failure → Route to fallback
  │     ├── Fallback Node → Secondary LLM (different model)
  │     │     ├── Success → Return response
  │     │     └── Failure → Route to error handler
  │     └── Error Node → Graceful error message
  │
  └── Response → Client
```
<p align="center">
  <img src="assets/arch.png" alt="AI Request Workflow Architecture" width="550" />
</p>


### Failure Handling Strategy

<p align="center">
  <img src="assets/fallback.svg" alt="LangGraph Fallback Recovery Flow" width="340" />
</p>

The application uses a multi-stage recovery strategy:

1. **Attempt** response generation using the primary model.
2. **If the request fails**, automatically retry using a fallback model.
3. **If both models fail**, return a graceful user-facing error message.
4. **Prevent** raw exceptions from reaching API consumers.

This pattern improves reliability and provides predictable behavior during upstream LLM outages.

## Configuration

All configuration is managed through environment variables. See `.env.example`:

| Variable | Default | Description |
|---|---|---|
| `OPENAI_API_KEY` | — | Your OpenRouter or OpenAI API key |
| `OPENAI_BASE_URL` | `https://openrouter.ai/api/v1` | API endpoint (swap for any OpenAI-compatible provider) |
| `PRIMARY_MODEL` | `openai/gpt-4o-mini` | Primary LLM model |
| `FALLBACK_MODEL` | `openai/gpt-4o-mini` | Fallback LLM model (used if primary fails) |
| `LANGCHAIN_TRACING_V2` | `true` | Enable LangSmith tracing |
| `LANGCHAIN_API_KEY` | — | LangSmith API key |
| `LANGCHAIN_PROJECT` | `production-api` | LangSmith project name |
| `APP_ENV` | `development` | Environment name (`development` / `production`) |
| `LOG_LEVEL` | `INFO` | Logging level |
| `RATE_LIMIT` | `5/minute` | Per-IP rate limit |
| `CACHE_TTL_SECONDS` | `300` | Response cache TTL |
| `MAX_RETRIES` | `3` | Max retry attempts before fallback |
 
## Project Structure

```
├── app/
│   ├── main.py           # FastAPI app: lifespan, routes, middleware
│   ├── agent.py          # LangGraph state machine (3-node graph)
│   ├── config.py         # pydantic-settings configuration
│   ├── models.py         # Pydantic request/response schemas
│   ├── security.py       # Injection detection, PII masking, output validation
│   ├── cache.py          # In-memory response cache with TTL
│   └── monitoring.py     # JSON logger, metrics collector, request timer
├── tests/
│   ├── test_security.py  # 15 tests — injection, PII, output validation
│   ├── test_cache.py     # 5 tests — hit, miss, TTL, case-insensitive, stats
│   └── test_api.py       # Integration test placeholder
├── .env.example          # All config variables with defaults
├── Dockerfile            # Production Docker image
├── docker-compose.yml    # Local deployment with health checks
├── render.yml            # Render infrastructure-as-code
├── pyproject.toml        # Python dependencies
└── Production-test-commands.sh  # Interactive test suite
```

## Testing

```bash
# Run all tests
uv run pytest tests/ -v

# Run specific test modules (no API key needed)
uv run pytest tests/test_security.py tests/test_cache.py -v

# Run with coverage
uv run pytest tests/ -v --cov=app
```

Test coverage:

| Module | Tests | What's Covered |
|---|---|---|
| Security | 15 | Injection detection, PII masking, output validation |
| Cache | 5 | Hit, miss, case-insensitive, TTL expiration, stats |
| API | — | Integration tests (to be added) |


## Quick Start

```bash
git clone <repo-url>
cd production-ai-platform

cp .env.example .env
# Set OPENAI_API_KEY

uv sync
uv run uvicorn app.main:app --reload
```

### Verify

```bash
curl http://localhost:8000/health
```

### Docker

```bash
docker compose up --build
```

### Deploy

GitHub → Render → Connect Repository → Set `OPENAI_API_KEY` → Deploy

### Environment

| Variable | Purpose |
|---|---|
| `APP_ENV=development` | Verbose logging, relaxed limits |
| `APP_ENV=production` | Production logging and rate limits |

## Roadmap

This project is actively developed. Planned improvements:

### Phase 2: Production Hardening
- [ ] JWT authentication and API key management
- [ ] Redis-backed caching (replaces in-memory)
- [ ] Prometheus metrics (replaces in-memory counters)
- [ ] Streaming responses via SSE
- [ ] Async LLM calls
- [ ] GitHub Actions CI/CD

### Phase 3: Advanced AI Platform
- [ ] Document ingestion and RAG pipeline (vector store + retrieval)
- [ ] RAGAS evaluation suite
- [ ] LangFuse observability integration
- [ ] Multi-tenant document isolation
- [ ] Agent tools (web search, code execution, calculator)
