# Ferro Labs AI Gateway - Quickstart Guide

> **Self-host | REST API | Admin API**  
> Open-source AI gateway in Go | Apache 2.0  
> Route requests across 30 providers and 2,500+ models through a single OpenAI-compatible API.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [Self-Hosting](#3-self-hosting)
   - 3.1 [Docker (Quickest)](#31-docker-quickest)
   - 3.2 [Deploy to Railway](#32-deploy-to-railway)
   - 3.3 [Build from Source](#33-build-from-source)
4. [REST API Reference](#4-rest-api-reference)
   - 4.1 [Authentication](#41-authentication)
   - 4.2 [Base URL](#42-base-url)
   - 4.3 [OpenAI-compatible Endpoints](#43-openai-compatible-endpoints)
   - 4.4 [Core Endpoints](#44-core-endpoints)
   - 4.5 [Proxy Pass-through](#45-proxy-pass-through)
   - 4.6 [Provider Selection](#46-provider-selection)
   - 4.7 [Ferro Request Extensions](#47-ferro-request-extensions)
   - 4.8 [Ferro Response Fields](#48-ferro-response-fields)
5. [Admin API](#5-admin-api)
   - 5.1 [Scopes](#51-scopes)
   - 5.2 [Read Endpoints](#52-read-endpoints)
   - 5.3 [Write Endpoints](#53-write-endpoints-admin-scope)
   - 5.4 [Query Parameters](#54-query-parameters)
   - 5.5 [Routing Strategies](#55-routing-strategies)
   - 5.6 [Bootstrap Keys](#56-bootstrap-keys)
6. [Python SDK](#6-python-sdk)
7. [Observability](#7-observability)
8. [Contributing & Development](#8-contributing--development)
9. [Useful Links](#9-useful-links)

---

## 1. Overview

The Ferro Labs AI Gateway is a high-performance, open-source control plane for AI applications, written in Go. It acts as an intelligent proxy between your application and upstream LLM providers, exposing a single unified OpenAI-compatible API regardless of which model or provider you target.

**Key capabilities:**

- **One API for 30 providers.** OpenAI, Anthropic, Google Gemini, Groq, Together AI, Mistral, Cohere, AWS Bedrock, Azure OpenAI, Vertex AI, Cerebras, DeepSeek, xAI, and more — all via a single endpoint and a single key.
- **Drop-in OpenAI replacement.** Any client, SDK, or framework that speaks the OpenAI API works without code changes — just point `base_url` at the gateway.
- **8 routing strategies.** Single, fallback, weighted load balancing, conditional, least-latency, cost-optimized, content-based, and A/B testing.
- **Built-in observability.** Every response carries `provider`, `cost_usd`, `latency_ms`, and `trace_id`. Prometheus `/metrics` and per-provider `/health` endpoints included.
- **6 built-in plugins.** Word filter, rate limiter, budget cap, response cache, request logger, max-token limiter.
- **No vendor lock-in.** Apache 2.0 licensed with no community vs. enterprise edition split.

### Supported Providers

The gateway supports **30 providers** out of the box:

| Provider | Provider | Provider |
|----------|----------|----------|
| AI21 | Anthropic | Azure AI Foundry |
| Azure OpenAI | AWS Bedrock | Cerebras |
| Cloudflare Workers AI | Cohere | Databricks |
| DeepInfra | DeepSeek | Fireworks AI |
| Google Gemini | Groq | Hugging Face |
| Mistral AI | Moonshot | Novita |
| NVIDIA NIM | Ollama (local) | Ollama Cloud |
| OpenAI | OpenRouter | Perplexity |
| Qwen | Replicate | SambaNova |
| Together AI | Vertex AI | xAI (Grok) |

---

## 2. Architecture

```
Your Application / any OpenAI-compatible client
            │
            ▼
┌────────────────────────────────────────────────────────┐
│                  Ferro AI Gateway :8080                │
│                                                        │
│  POST /v1/chat/completions                             │
│  POST /v1/embeddings                                   │
│  POST /v1/images/generations                           │
│  GET  /metrics      GET  /health                       │
│  /admin/*  (key management, config, logs)              │
│                                                        │
│  ┌──────────────────────────────────────────────────┐  │
│  │             Smart Routing Engine                 │  │
│  │  Single · Fallback · Weighted · Conditional ·    │  │
│  │  Least-latency · Cost-optimized · Content-based  │  │
│  │  · A/B Test                                      │  │
│  └──────────────────────┬───────────────────────────┘  │
│                         │                              │
│  ┌──────────────────────▼───────────────────────────┐  │
│  │               Plugin Pipeline                    │  │
│  │  Cache · Logger · Word Filter · Rate Limiter     │  │
│  │  Max Token Limiter · Budget Cap                  │  │
│  └──────────────────────┬───────────────────────────┘  │
│                         │                              │
│  ┌──────────────────────▼───────────────────────────┐  │
│  │           Provider Adapters (30+)                │  │
│  │  OpenAI · Anthropic · Gemini · Bedrock           │  │
│  │  Azure · Groq · Mistral · DeepSeek ...           │  │
│  └──────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

The gateway is a single Go binary with zero runtime dependencies. It speaks the standard OpenAI HTTP API spec on its ingress side and translates each request to the target provider's native format on egress. The upstream provider is selected automatically from the `model` field in the request — no per-request routing configuration needed.

---

## 3. Self-Hosting

### 3.1 Docker (Quickest)

Pull the official image from GitHub Container Registry and start the gateway in one command:

```bash
docker run --rm -p 8080:8080 \
  -e OPENAI_API_KEY=sk-openai-your-key \
  ghcr.io/ferro-labs/ai-gateway:latest
```

The gateway listens on `http://localhost:8080`. Use the `:edge` tag for the latest unreleased code from `main`.

To enable multiple providers, pass each provider's key as an environment variable. The gateway auto-discovers credentials — no per-provider setup code required:

```bash
docker run --rm -p 8080:8080 \
  -e OPENAI_API_KEY=sk-openai-... \
  -e ANTHROPIC_API_KEY=sk-ant-... \
  -e GOOGLE_API_KEY=... \
  -e GROQ_API_KEY=... \
  ghcr.io/ferro-labs/ai-gateway:latest
```

### 3.2 Deploy to Railway

Railway turns a Docker image into a running service with zero infrastructure config. Choose between SQLite (simplest) or PostgreSQL (production-ready) storage.

#### One-click deploy

| Template | Storage | Best for |
|----------|---------|----------|
| **[Deploy with SQLite](https://railway.com/deploy/ferro-labs-ai-sqlite-storage?referralCode=KblxKX)** | File-based on Railway Volume | Development, single-instance, low traffic |
| **[Deploy with PostgreSQL](https://railway.com/deploy/ferro-labs-ai-postgresql-storage?referralCode=KblxKX)** | Managed Railway Postgres | Production, multiple instances, durability |

Both templates prompt you for provider API keys and configure storage automatically.

#### SQLite template

The SQLite template attaches a Railway Volume at `/data` for persistent file storage. Environment variables:

```bash
MASTER_KEY=fgw_your-master-key          # auto-generated
OPENAI_API_KEY=sk-your-key              # you provide
PORT=8080

# SQLite storage — persisted to Railway Volume
API_KEY_STORE_BACKEND=sqlite
API_KEY_STORE_DSN=/data/keys.db
CONFIG_STORE_BACKEND=sqlite
CONFIG_STORE_DSN=/data/config.db
REQUEST_LOG_STORE_BACKEND=sqlite
REQUEST_LOG_STORE_DSN=/data/logs.db
RAILWAY_RUN_UID=0
```

> **When to use SQLite:** SQLite is the fastest path to a running gateway. It works well for development, demos, and low-traffic production workloads. For high availability or multi-replica deployments, use PostgreSQL.

#### PostgreSQL template

The PostgreSQL template provisions a managed Postgres instance and wires all three store backends automatically:

```bash
MASTER_KEY=fgw_your-master-key
OPENAI_API_KEY=sk-your-key
PORT=8080

# PostgreSQL storage — auto-wired to Railway Postgres
API_KEY_STORE_BACKEND=postgres
API_KEY_STORE_DSN=${{Postgres.DATABASE_URL}}
CONFIG_STORE_BACKEND=postgres
CONFIG_STORE_DSN=${{Postgres.DATABASE_URL}}
REQUEST_LOG_STORE_BACKEND=postgres
REQUEST_LOG_STORE_DSN=${{Postgres.DATABASE_URL}}
```

#### Manual Railway setup

**Step 1 — Install the Railway CLI:**

```bash
npm install -g @railway/cli
railway login
```

**Step 2 — Create the project:**

```bash
railway init
```

**Step 3 — Deploy the gateway image:**

```bash
railway up --image ghcr.io/ferro-labs/ai-gateway:latest
```

**Step 4 — Set environment variables:**

```bash
railway variables set MASTER_KEY=fgw_your-master-key
railway variables set OPENAI_API_KEY=sk-your-key
railway variables set PORT=8080
railway variables set API_KEY_STORE_BACKEND=sqlite
railway variables set API_KEY_STORE_DSN=/data/keys.db
railway variables set CONFIG_STORE_BACKEND=sqlite
railway variables set CONFIG_STORE_DSN=/data/config.db
railway variables set REQUEST_LOG_STORE_BACKEND=sqlite
railway variables set REQUEST_LOG_STORE_DSN=/data/logs.db
```

> **Warning:** Never hardcode API keys in your Dockerfile or config files. Always use Railway environment variables — they are encrypted at rest.

**Step 5 — Attach a volume (SQLite only):**

In the Railway dashboard, go to your service → **Settings** → **Volumes** and mount a volume at `/data`.

**Step 6 — Deploy:**

```bash
railway up
```

#### Adding more providers

```bash
railway variables set ANTHROPIC_API_KEY=sk-ant-...
railway variables set GEMINI_API_KEY=...
railway variables set MISTRAL_API_KEY=...
```

The gateway picks up new keys on restart.

#### Verify the deployment

```bash
# Health check
curl https://your-project.up.railway.app/health

# Test request
curl https://your-project.up.railway.app/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $MASTER_KEY" \
  -d '{
    "model": "gpt-4o",
    "messages": [{"role": "user", "content": "Hello from Railway!"}]
  }'
```

> **Tip:** Your Railway URL is shown in the dashboard under your service's **Settings** tab, or in the output of `railway up`.

> **Security note:** For production workloads, restrict ingress to your application's IP range using Railway's networking settings, and ensure Ferro API keys are issued with minimal required scopes via the Admin API.

### 3.3 Build from Source

Requires Go 1.24+.

```bash
git clone https://github.com/ferro-labs/ai-gateway.git
cd ai-gateway

export OPENAI_API_KEY=sk-your-key
make run
# Server starts on :8080
```

#### Available Make targets

| Command | Description |
|---------|-------------|
| `make build` | Compile binary to `./bin/ferrogw` |
| `make run` | Build and start the gateway on `:8080` |
| `make test` | Run Go tests (short mode, with race detection) |
| `make test-coverage` | Run tests with coverage report |
| `make test-integration` | Run integration tests |
| `make lint` | Run golangci-lint |
| `make fmt` | Format code with gofmt |
| `make clean` | Remove build artifacts |

---

## 4. REST API Reference

The gateway exposes an OpenAI-compatible API surface so you can reuse existing clients. Any HTTP client works — no SDK required.

### 4.1 Authentication

Every request must include a Ferro API key:

```bash
Authorization: Bearer <api-key>
```

Keys are created and managed via the [Admin API](#5-admin-api).

### 4.2 Base URL

```
http://localhost:8080
```

### 4.3 OpenAI-compatible Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/chat/completions` | POST | Chat completions (streaming supported) |
| `/v1/completions` | POST | Legacy completions |
| `/v1/embeddings` | POST | Text embeddings |
| `/v1/images/generations` | POST | Image generation |
| `/v1/models` | GET | List available models |

### 4.4 Core Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/health` | GET | Provider availability and model counts |
| `/metrics` | GET | Prometheus metrics |
| `/dashboard` | GET | Minimal admin UI |

### 4.5 Proxy Pass-through

All other `/v1/*` requests are proxied to the selected provider, including:

- `/v1/files`
- `/v1/batches`
- `/v1/fine_tuning`
- `/v1/responses`
- `/v1/audio/*`
- `/v1/images/edits`
- `/v1/realtime`

### 4.6 Provider Selection

For proxy routes, the gateway resolves the provider in this order:

1. `X-Provider` header (for example: `openai` or `anthropic`)
2. `model` field in the JSON body

### 4.7 Ferro Request Extensions

These fields can be added to any `/v1/chat/completions` request:

| Field | Type | Description |
|-------|------|-------------|
| `template_id` | string | ID of a server-side prompt template defined in gateway config |
| `template_variables` | object | Key/value pairs rendered into the template at request time |
| `route_tag` | string | Overrides the active routing strategy for this single request only |

### 4.8 Ferro Response Fields

Every inference response includes these additional fields:

| Field | Type | Description |
|-------|------|-------------|
| `provider` | string | Upstream provider that served the request (e.g. `"openai"`, `"anthropic"`) |
| `trace_id` | string | Correlates this response with gateway request logs |
| `latency_ms` | integer | End-to-end gateway latency in milliseconds |
| `usage.cost_usd` | float | Estimated cost in USD based on the provider's pricing table |
| `usage.cache_hit` | boolean | `true` if served from the gateway's semantic cache |

**Example:**

```bash
curl http://localhost:8080/v1/chat/completions \
  -H "Authorization: Bearer sk-ferro-your-key" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4o",
    "messages": [{"role": "user", "content": "Hello!"}]
  }'
```

---

## 5. Admin API

Admin endpoints are mounted under `/admin` and protected with a bearer token.

```bash
Authorization: Bearer <api-key>
```

### 5.1 Scopes

| Scope | Description |
|-------|-------------|
| `admin` | Full access to all admin endpoints |
| `read_only` | Read endpoints only |

### 5.2 Read Endpoints

| Endpoint | Description |
|----------|-------------|
| `GET /admin/dashboard` | Dashboard summary (providers, keys, logs) |
| `GET /admin/keys` | List all API keys |
| `GET /admin/keys/usage` | Key usage statistics |
| `GET /admin/keys/{id}` | Get specific key details |
| `GET /admin/logs` | List request logs |
| `GET /admin/logs/stats` | Request log statistics |
| `GET /admin/providers` | List available providers |
| `GET /admin/health` | Health check |
| `GET /admin/plugins` | List enabled plugins |
| `GET /admin/config` | Get current configuration |
| `GET /admin/config/history` | Configuration version history |

### 5.3 Write Endpoints (admin scope)

| Endpoint | Description |
|----------|-------------|
| `POST /admin/keys` | Create new API key |
| `PUT /admin/keys/{id}` | Update API key |
| `DELETE /admin/keys/{id}` | Delete API key |
| `POST /admin/keys/{id}/revoke` | Revoke API key |
| `POST /admin/keys/{id}/rotate` | Rotate API key |
| `DELETE /admin/logs` | Delete request logs |
| `POST /admin/config` | Create configuration |
| `PUT /admin/config` | Update configuration |
| `DELETE /admin/config` | Delete configuration |
| `POST /admin/config/rollback/{version}` | Rollback to previous config version |

### 5.4 Query Parameters

#### `GET /admin/keys/usage`

| Parameter | Default | Max | Description |
|-----------|---------|-----|-------------|
| `limit` | 20 | 100 | Number of results |
| `offset` | 0 | - | Pagination offset |
| `sort` | `usage` | - | Sort by `usage` or `last_used` |
| `active` | - | - | Filter by `true` or `false` |
| `since` | - | - | RFC3339 timestamp filter |

#### `GET /admin/logs`

| Parameter | Default | Max | Description |
|-----------|---------|-----|-------------|
| `limit` | 50 | 200 | Number of results |
| `offset` | 0 | - | Pagination offset |
| `stage` | - | - | Filter by stage |
| `model` | - | - | Filter by model |
| `provider` | - | - | Filter by provider |
| `since` | - | - | RFC3339 timestamp filter |

#### `GET /admin/logs/stats`

| Parameter | Default | Max | Description |
|-----------|---------|-----|-------------|
| `limit` | - | 100 | Top-N buckets |
| `stage` | - | - | Filter by stage |
| `model` | - | - | Filter by model |
| `provider` | - | - | Filter by provider |
| `since` | - | - | RFC3339 timestamp filter |

#### `DELETE /admin/logs`

| Parameter | Required | Description |
|-----------|----------|-------------|
| `before` | Yes | RFC3339 timestamp (delete logs before this time) |
| `stage` | No | Filter by stage |
| `model` | No | Filter by model |
| `provider` | No | Filter by provider |

When request log storage is disabled, log endpoints return `501 Not Implemented`.

### 5.5 Routing Strategies

Config updates via `PUT /admin/config` are zero-downtime hot reloads. All changes are versioned and can be rolled back via `POST /admin/config/rollback/{version}`.

| Mode | Behavior |
|------|----------|
| `single` | Always route to one target |
| `fallback` | Try providers in order; exponential backoff on failure |
| `loadbalance` | Distribute load across providers proportionally by `weight` |
| `conditional` | Route based on model name or `route_tag` |
| `least-latency` | Route to the provider with the lowest observed latency |
| `cost-optimized` | Route to the cheapest provider capable of serving the model |
| `content-based` | Route based on request content analysis |
| `ab-test` | Split traffic between providers for comparison |

For detailed routing configuration, see [Routing Strategies Documentation](./routing-strategies.md).

### 5.6 Bootstrap Keys

Use bootstrap keys to access admin endpoints on first run. They are only honored when the API key store is empty.

```bash
export ADMIN_BOOTSTRAP_KEY=change-me
export ADMIN_BOOTSTRAP_READ_ONLY_KEY=change-me
export ADMIN_BOOTSTRAP_ENABLED=true
```

---

## 6. Python SDK

The official Python SDK (`ferrolabsai`) provides typed clients, streaming helpers, async support, and full admin API access.

```bash
pip install ferrolabsai
```

### Authentication

The client reads credentials from environment variables or constructor arguments:

```bash
export FERRO_API_KEY=sk-ferro-...
export FERRO_BASE_URL=http://localhost:8080   # default
```

Or pass them directly:

```python
from ferrolabsai import FerroClient

client = FerroClient(
    api_key="sk-ferro-...",
    base_url="http://localhost:8080",
)
```

The client also checks `OPENAI_API_KEY` as a fallback, so existing OpenAI key setups work out of the box.

### FerroClient Constructor Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `api_key` | `str` | `None` | Falls back to `FERRO_API_KEY` then `OPENAI_API_KEY` env vars |
| `base_url` | `str` | `"http://localhost:8080"` | Falls back to `FERRO_BASE_URL` env var |
| `timeout` | `float` | `120.0` | HTTP timeout in seconds |
| `max_retries` | `int` | `2` | Retry count for connection errors and timeouts |
| `default_headers` | `dict` | `None` | Extra headers merged into every request |
| `http_client` | `httpx.Client` | `None` | Bring your own `httpx` client |

### Chat Completions

```python
from ferrolabsai import FerroClient

client = FerroClient()

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Hello from Ferro Labs!"}],
)

print(response.content)           # shortcut to choices[0].message.content
print(f"Provider: {response.provider}")
print(f"Cost: ${response.usage.cost_usd:.6f}")
print(f"Trace ID: {response.trace_id}")
```

**Streaming:**

```python
stream = client.chat.completions.create(
    model="claude-3-5-sonnet-20241022",
    messages=[{"role": "user", "content": "Write a haiku about AI gateways"}],
    stream=True,
)

for chunk in stream:
    print(chunk.choices[0].delta.content or "", end="", flush=True)
```

**Ferro-specific parameters:**

```python
response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Summarize this document"}],
    template_id="summarizer-v2",            # server-side prompt template
    template_variables={"tone": "formal"},  # template variables
    route_tag="premium",                    # override routing strategy
)
```

### Embeddings

```python
response = client.embeddings.create(
    model="text-embedding-3-small",
    input=["Ferro routes LLM requests", "across 30 providers"],
)

for item in response.data:
    print(f"Dimension: {len(item.embedding)}")
```

### Image Generation

```python
response = client.images.generate(
    model="dall-e-3",
    prompt="A futuristic AI gateway routing requests across the cosmos",
    size="1024x1024",
)

print(response.data[0].url)
```

### Model Discovery

```python
# List all models from a specific provider
models = client.models.list(provider="anthropic")
for m in models:
    print(f"{m.id} — {m.context_window:,} tokens")

# Search the catalog
results = client.models.search("embedding")

# Retrieve a specific model
model = client.models.retrieve("gpt-4o")
```

### Admin API (Python)

Admin methods require an API key with `admin` scope.

```python
# Keys
keys = client.admin.keys.list()
new_key = client.admin.keys.create(name="backend", scopes=["chat"])
client.admin.keys.rotate("key_abc123")
client.admin.keys.revoke("key_abc123")

# Config
config = client.admin.config.get()
client.admin.config.update(config={"routing": {"strategy": "latency"}})
client.admin.config.rollback(version=3)

# Logs
logs = client.admin.logs.list(limit=50, provider="openai")
stats = client.admin.logs.stats(since="2025-01-01T00:00:00Z")
client.admin.logs.delete(before="2024-01-01T00:00:00Z")

# Providers, plugins, dashboard
providers = client.admin.providers.list()
plugins = client.admin.plugins.list()
dashboard = client.admin.dashboard()
health = client.admin.health()
```

### Async Usage

`AsyncFerroClient` mirrors the `FerroClient` interface. All methods return awaitables and streaming returns `AsyncIterator[ChatCompletionChunk]`.

```python
from ferrolabsai import AsyncFerroClient

async with AsyncFerroClient() as client:
    response = await client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": "Hello from async!"}],
    )
    print(response.content)
```

**Async streaming:**

```python
stream = await client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "Count from 1 to 10"}],
    stream=True,
)

async for chunk in stream:
    print(chunk.choices[0].delta.content or "", end="", flush=True)
```

### Error Handling

The SDK maps every failure to a typed exception:

```
FerroError
├── FerroAPIError
│   ├── FerroAuthError          # 401
│   ├── FerroRateLimitError     # 429
│   ├── FerroNotFoundError      # 404
│   └── FerroServerError        # 5xx
├── FerroConnectionError        # network / timeout (retried automatically)
└── FerroStreamError            # SSE parse failure during streaming
```

```python
from ferrolabsai import (
    FerroClient, FerroAuthError, FerroRateLimitError,
    FerroServerError, FerroConnectionError, FerroAPIError,
)

client = FerroClient()

try:
    response = client.chat.completions.create(
        model="gpt-4o",
        messages=[{"role": "user", "content": "Hello"}],
    )
except FerroAuthError:
    print("Invalid or missing API key")
except FerroRateLimitError:
    print("Rate limit exceeded — back off and retry")
except FerroServerError as e:
    print(f"Server error ({e.status_code}): {e.message}")
except FerroConnectionError:
    print("Network error — retries exhausted")
except FerroAPIError as e:
    print(f"API error {e.status_code}: {e.message} (request_id={e.request_id})")
```

Connection errors and timeouts are retried automatically up to `max_retries` (default `2`). HTTP 4xx/5xx errors are never retried — the gateway handles provider-level retries internally.

---

## 7. Observability

The gateway emits observability data at three levels:

### Per-response Metadata

Every inference response includes Ferro-specific fields with no extra calls needed:

| Field | Type | Description |
|-------|------|-------------|
| `provider` | string | Upstream provider that served the request |
| `trace_id` | string | Unique ID correlating this response with gateway logs |
| `latency_ms` | integer | End-to-end gateway latency in milliseconds |
| `usage.cost_usd` | float | Estimated cost in USD based on the provider's token pricing |
| `usage.cache_hit` | boolean | `true` if served from the gateway's semantic cache |

### Prometheus Metrics

Available at `GET /metrics`:

- Request count by provider, model, and status
- Latency histograms (p50, p95, p99)
- Token usage counters
- Cache hit rate

### Structured Request Logs

When the `request-logger` plugin is enabled, every request is logged as a JSON entry with:

- `trace_id`
- `model`
- `provider`
- `stage`
- `total_tokens`
- `error_message`
- timestamps

Queryable via the [Admin API logs endpoint](#52-read-endpoints).

---

## 8. Contributing & Development

```bash
git clone https://github.com/ferro-labs/ai-gateway
cd ai-gateway

make run    # start the gateway on :8080
make test   # run Go tests
make lint   # golangci-lint
make build  # compile binary to ./bin/
```

### Priority Contribution Areas

- New LLM provider adapters
- New middleware plugins
- Expanded test coverage

See [CONTRIBUTING.md](https://github.com/ferro-labs/ai-gateway/blob/main/CONTRIBUTING.md) for style guidelines and PR process.

See [ROADMAP.md](https://github.com/ferro-labs/ai-gateway/blob/main/ROADMAP.md) for the planned release roadmap, including persistent config storage, Helm charts, and OpenTelemetry export.

---

## 9. Useful Links

| Resource | URL |
|----------|-----|
| AI Gateway (GitHub) | https://github.com/ferro-labs/ai-gateway |
| Official docs | https://docs.ferrolabs.ai |
| Interactive API spec | https://docs.ferrolabs.ai/api/ |
| Gateway Docker image | `ghcr.io/ferro-labs/ai-gateway:latest` |
| Roadmap | https://github.com/ferro-labs/ai-gateway/blob/main/ROADMAP.md |
| Contributing guide | https://github.com/ferro-labs/ai-gateway/blob/main/CONTRIBUTING.md |
| Issue tracker | https://github.com/ferro-labs/ai-gateway/issues |

---

## Related Documentation

- [Routing Strategies](./routing-strategies.md) - Detailed guide on all 8 routing strategies
- [Plugins](./plugins.md) - Comprehensive plugin documentation

---

*Apache 2.0 Licensed — Ferro Labs*
