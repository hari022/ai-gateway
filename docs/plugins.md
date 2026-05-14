# Plugins

Plugins extend the AI Gateway's functionality by hooking into the request pipeline at various lifecycle stages. They can inspect, modify, reject, or skip requests and responses, enabling features like rate limiting, content filtering, caching, logging, and budget enforcement.

## Table of Contents

- [Overview](#overview)
- [Plugin Architecture](#plugin-architecture)
  - [Lifecycle Stages](#lifecycle-stages)
  - [Plugin Types](#plugin-types)
  - [Plugin Context](#plugin-context)
- [Built-in Plugins](#built-in-plugins)
  - [Word Filter](#word-filter)
  - [Max Token](#max-token)
  - [Rate Limit](#rate-limit)
  - [Budget](#budget)
  - [Response Cache](#response-cache)
  - [Request Logger](#request-logger)
- [Configuration Reference](#configuration-reference)
- [Plugin Execution Flow](#plugin-execution-flow)
- [Creating Custom Plugins](#creating-custom-plugins)
- [Best Practices](#best-practices)

---

## Overview

The plugin system provides a flexible middleware layer for the AI Gateway. Each plugin:

- Implements a standard interface (`Plugin`)
- Registers itself with a factory function
- Executes at one or more lifecycle stages
- Can access and modify request/response data through a shared context

Plugins are configured in your `config.yaml` file and are executed in the order they are defined.

---

## Plugin Architecture

### Lifecycle Stages

Plugins can be registered at three distinct stages in the request lifecycle:

| Stage | Constant | Description |
|-------|----------|-------------|
| **Before Request** | `before_request` | Executes before the request is sent to the LLM provider. Use for validation, filtering, caching lookups, and rate limiting. |
| **After Request** | `after_request` | Executes after receiving a response from the provider. Use for logging, caching storage, and response transformation. |
| **On Error** | `on_error` | Executes when an error occurs during request processing. Use for error logging and alerting. |

### Plugin Types

Plugins are categorized by their primary function:

| Type | Constant | Description |
|------|----------|-------------|
| **Guardrail** | `guardrail` | Validates and filters requests/responses (e.g., word filter, max token) |
| **Rate Limit** | `ratelimit` | Controls request throughput and budget (e.g., rate limiter, budget) |
| **Transform** | `transform` | Modifies requests/responses (e.g., response cache) |
| **Logging** | `logging` | Records request/response data (e.g., request logger) |
| **Metrics** | `metrics` | Collects performance and usage metrics |
| **Auth** | `auth` | Handles authentication and authorization |

### Plugin Context

The `plugin.Context` struct provides access to request and response data:

```go
type Context struct {
    Request  *providers.Request   // The incoming LLM request
    Response *providers.Response  // The LLM response (nil in before_request stage)
    Metadata map[string]interface{} // Shared metadata between plugins
    Error    error                // Error from previous stages
    Skip     bool                 // Set to true to skip remaining plugins
    Reject   bool                 // Set to true to reject the request
    Reason   string               // Rejection reason message
}
```

**Key fields:**
- `Request`: Contains model, messages, max_tokens, and other request parameters
- `Response`: Contains choices, usage statistics, and provider information (only available in `after_request`)
- `Metadata`: Shared key-value store for passing data between plugins (e.g., `api_key`, `cache_hit`)
- `Skip`: When set to `true`, remaining plugins in the current stage are skipped
- `Reject`: When set to `true`, the request is rejected with an error response

---

## Built-in Plugins

### Word Filter

**Name:** `word-filter`  
**Type:** `guardrail`  
**Stage:** `before_request`

Blocks requests containing specified words or phrases. Useful for content moderation and preventing sensitive information from being sent to LLM providers.

#### Configuration

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `blocked_words` | `[]string` | `[]` | List of words or phrases to block |
| `case_sensitive` | `bool` | `false` | Whether matching should be case-sensitive |

#### Example

```yaml
plugins:
  - name: word-filter
    type: guardrail
    stage: before_request
    enabled: true
    config:
      blocked_words:
        - "password"
        - "secret"
        - "api_key"
        - "confidential"
      case_sensitive: false
```

#### Behavior

- Scans all message content in the request
- Rejects the request immediately when a blocked word is detected
- Returns rejection reason: `"blocked word detected: <word>"`

---

### Max Token

**Name:** `max-token`  
**Type:** `guardrail`  
**Stage:** `before_request`

Enforces limits on token count, message count, and total input length. Prevents runaway costs and protects against excessively large requests.

#### Configuration

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `max_tokens` | `int` | `4096` | Maximum allowed value for `max_tokens` in the request |
| `max_messages` | `int` | `100` | Maximum number of messages allowed in the conversation |
| `max_input_length` | `int` | `0` | Maximum total character length of all messages (0 = no limit) |

#### Example

```yaml
plugins:
  - name: max-token
    type: guardrail
    stage: before_request
    enabled: true
    config:
      max_tokens: 8192
      max_messages: 50
      max_input_length: 100000
```

#### Behavior

- Checks `max_tokens` parameter in the request against the configured limit
- Counts total messages and rejects if over `max_messages`
- Sums character lengths of all message contents and rejects if over `max_input_length`
- Returns specific rejection reason indicating which limit was exceeded

---

### Rate Limit

**Name:** `rate-limit`  
**Type:** `ratelimit`  
**Stage:** `before_request`

Enforces request rate limits using token bucket algorithms. Supports global, per-API-key, and per-user rate limiting.

#### Configuration

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `requests_per_second` | `float64` | `100` | Global requests per second limit |
| `burst` | `float64` | `= rps` | Burst capacity (defaults to `requests_per_second`) |
| `key_rpm` | `float64` | *none* | Per-API-key rate limit in requests per minute |
| `user_rpm` | `float64` | *none* | Per-user rate limit in requests per minute |

#### Example

```yaml
plugins:
  - name: rate-limit
    type: guardrail
    stage: before_request
    enabled: true
    config:
      requests_per_second: 50
      burst: 100
      key_rpm: 120
      user_rpm: 30
```

#### Behavior

Three layers of rate limiting are applied in order:

1. **Global limiter**: Applied to all traffic based on `requests_per_second` and `burst`
2. **Per-API-key limiter**: Applied per unique API key from `Metadata["api_key"]`
3. **Per-user limiter**: Applied per unique user ID from `Request.User`

A request is rejected as soon as any limiter denies it.

#### Rate Limit Types

| Rejection Reason | Description |
|-----------------|-------------|
| `"rate limit exceeded"` | Global rate limit exceeded |
| `"per-key rate limit exceeded"` | API key's rate limit exceeded |
| `"per-user rate limit exceeded"` | User's rate limit exceeded |

---

### Budget

**Name:** `budget`  
**Type:** `ratelimit`  
**Stage:** `before_request` and `after_request`

Enforces per-API-key USD spend limits. Tracks cumulative costs based on token usage and configured pricing.

#### Configuration

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `store_id` | `string` | `"default"` | Shared store identifier for spend tracking |
| `spend_limit_usd` | `float64` | `0` | Maximum cumulative spend per API key (0 = unlimited) |
| `input_per_m_tokens` | `float64` | `0` | Cost per 1 million prompt tokens (USD) |
| `output_per_m_tokens` | `float64` | `0` | Cost per 1 million completion tokens (USD) |
| `max_keys` | `int` | `10000` | Maximum API keys tracked in memory |

#### Example

```yaml
plugins:
  # Check budget before request
  - name: budget
    type: guardrail
    stage: before_request
    enabled: true
    config:
      store_id: default
      spend_limit_usd: 50.0
      input_per_m_tokens: 3.0
      output_per_m_tokens: 15.0
      max_keys: 10000

  # Record spend after request
  - name: budget
    type: guardrail
    stage: after_request
    enabled: true
    config:
      store_id: default
      spend_limit_usd: 50.0
      input_per_m_tokens: 3.0
      output_per_m_tokens: 15.0
```

#### Behavior

**Before Request Stage:**
- Checks if the API key's accumulated spend exceeds `spend_limit_usd`
- Rejects the request if over budget

**After Request Stage:**
- Calculates cost from response token usage:
  ```
  cost = (prompt_tokens / 1,000,000) * input_per_m_tokens +
         (completion_tokens / 1,000,000) * output_per_m_tokens
  ```
- Adds the cost to the API key's accumulated spend

#### Important Notes

- Spend data is **in-memory only** and does not survive process restarts
- Suitable for session-scoped soft limits and development quotas
- When `max_keys` is reached, the key with the lowest accumulated spend is evicted
- API key is read from `Metadata["api_key"]`; requests without a key are not tracked

---

### Response Cache

**Name:** `response-cache`  
**Type:** `transform`  
**Stage:** `before_request` and `after_request`

Caches LLM responses in memory using exact-match hashing. Reduces provider costs and latency for repeated identical requests.

#### Configuration

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `max_age` | `int` | `300` | Cache entry TTL in seconds |
| `max_entries` | `int` | `1000` | Maximum number of cached responses |

#### Example

```yaml
plugins:
  - name: response-cache
    type: transform
    stage: before_request
    enabled: true
    config:
      max_age: 600
      max_entries: 5000
```

#### Behavior

**Cache Key Generation:**
- SHA-256 hash of model name + all message content (role, name, content)
- Exact match required for cache hits

**Before Request Stage:**
- Looks up the cache key
- On hit: Sets `pctx.Response` with cached response, sets `pctx.Skip = true`, sets `Metadata["cache_hit"] = true`
- On miss: Continues to provider

**After Request Stage:**
- Stores the response if not already a cache hit
- Evicts the entry with the earliest expiration when at capacity

---

### Request Logger

**Name:** `request-logger`  
**Type:** `logging`  
**Stage:** `before_request`, `after_request`, and `on_error`

Emits structured log entries for every request, response, and error. Optionally persists logs to SQLite or PostgreSQL.

#### Configuration

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `level` | `string` | `"info"` | Log level: `debug`, `info`, `warn`, `error` |
| `persist` | `bool` | `false` | Enable persistent storage |
| `backend` | `string` | `"sqlite"` | Storage backend: `sqlite` or `postgres` |
| `dsn` | `string` | `""` | SQLite file path or PostgreSQL connection string |

#### Example

```yaml
plugins:
  - name: request-logger
    type: logging
    stage: before_request
    enabled: true
    config:
      level: info
      persist: true
      backend: sqlite
      dsn: ferrogw-requests.db
```

#### Logged Information

**Before Request:**
- Model name
- Message count
- Stream flag
- Timestamp

**After Request:**
- Model and provider
- Token usage (prompt, completion, total)
- Choice count
- Timestamp

**On Error:**
- Model name
- Error message
- Timestamp

---

## Configuration Reference

### Plugin Configuration Schema

```yaml
plugins:
  - name: <plugin-name>       # Required: Registered plugin name
    type: <plugin-type>       # Required: guardrail, ratelimit, transform, logging, metrics, auth
    stage: <lifecycle-stage>  # Required: before_request, after_request, on_error
    enabled: <bool>           # Required: Whether the plugin is active
    config:                   # Optional: Plugin-specific configuration
      <key>: <value>
```

### Complete Configuration Example

```yaml
plugins:
  # Content moderation
  - name: word-filter
    type: guardrail
    stage: before_request
    enabled: true
    config:
      blocked_words: ["password", "secret", "api_key"]
      case_sensitive: false

  # Request size limits
  - name: max-token
    type: guardrail
    stage: before_request
    enabled: true
    config:
      max_tokens: 4096
      max_messages: 50
      max_input_length: 100000

  # Rate limiting
  - name: rate-limit
    type: guardrail
    stage: before_request
    enabled: true
    config:
      requests_per_second: 100
      burst: 150
      key_rpm: 60
      user_rpm: 30

  # Budget enforcement (before)
  - name: budget
    type: guardrail
    stage: before_request
    enabled: true
    config:
      store_id: production
      spend_limit_usd: 100.0
      input_per_m_tokens: 2.5
      output_per_m_tokens: 10.0

  # Response caching
  - name: response-cache
    type: transform
    stage: before_request
    enabled: true
    config:
      max_age: 300
      max_entries: 1000

  # Request logging
  - name: request-logger
    type: logging
    stage: before_request
    enabled: true
    config:
      level: info
      persist: true
      backend: postgres
      dsn: "postgres://user:pass@localhost/ferrogw?sslmode=disable"

  # Budget tracking (after)
  - name: budget
    type: guardrail
    stage: after_request
    enabled: true
    config:
      store_id: production
      spend_limit_usd: 100.0
      input_per_m_tokens: 2.5
      output_per_m_tokens: 10.0
```

---

## Plugin Execution Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Incoming Request                              │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    BEFORE_REQUEST Stage                              │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐               │
│  │ word-filter  │→ │  max-token   │→ │  rate-limit  │→ ...          │
│  └──────────────┘  └──────────────┘  └──────────────┘               │
│                                                                      │
│  • Plugins execute in order                                          │
│  • If pctx.Reject = true → Return error immediately                 │
│  • If pctx.Skip = true → Skip remaining plugins                     │
│  • If pctx.Response != nil → Skip LLM call (cache hit)              │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      LLM Provider Call                               │
│                                                                      │
│  Request → Provider → Response (or Error)                            │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    ▼                           ▼
            [Success]                      [Error]
                    │                           │
                    ▼                           ▼
┌───────────────────────────────┐ ┌───────────────────────────────────┐
│     AFTER_REQUEST Stage       │ │         ON_ERROR Stage            │
│                               │ │                                   │
│  ┌────────────┐  ┌─────────┐  │ │  ┌────────────────┐               │
│  │   budget   │→ │  logger │  │ │  │ request-logger │               │
│  └────────────┘  └─────────┘  │ │  └────────────────┘               │
│                               │ │                                   │
│  • Response available         │ │  • pctx.Error contains error      │
│  • Record metrics/costs       │ │  • Log errors, send alerts        │
└───────────────────────────────┘ └───────────────────────────────────┘
                    │                           │
                    └─────────────┬─────────────┘
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       Return Response                                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Creating Custom Plugins

To create a custom plugin, implement the `Plugin` interface and register it with a factory function:

```go
package myplugin

import (
    "context"
    "github.com/ferro-labs/ai-gateway/plugin"
)

func init() {
    plugin.RegisterFactory("my-plugin", func() plugin.Plugin {
        return &MyPlugin{}
    })
}

type MyPlugin struct {
    // Plugin configuration fields
    threshold int
}

func (p *MyPlugin) Name() string {
    return "my-plugin"
}

func (p *MyPlugin) Type() plugin.PluginType {
    return plugin.TypeGuardrail
}

func (p *MyPlugin) Init(config map[string]interface{}) error {
    // Parse configuration
    if v, ok := config["threshold"].(float64); ok {
        p.threshold = int(v)
    }
    return nil
}

func (p *MyPlugin) Execute(ctx context.Context, pctx *plugin.Context) error {
    // Plugin logic here
    
    // To reject a request:
    // pctx.Reject = true
    // pctx.Reason = "rejection reason"
    
    // To skip remaining plugins:
    // pctx.Skip = true
    
    return nil
}
```

### Registration

Import your plugin package with a blank import in your main application:

```go
import (
    _ "your-module/internal/plugins/myplugin"
)
```

---

## Best Practices

### Plugin Ordering

1. **Guardrails first**: Place validation plugins (word-filter, max-token) early to reject invalid requests before expensive operations
2. **Rate limiting second**: Apply rate limits before processing to protect downstream systems
3. **Cache lookups third**: Check cache before making provider calls
4. **Logging throughout**: Register loggers at multiple stages for comprehensive observability

### Performance Considerations

- **Minimize allocations**: The plugin context is pooled to reduce GC pressure
- **Use Skip wisely**: Set `pctx.Skip = true` to short-circuit when further processing is unnecessary
- **Cache expensive computations**: Store computed values in `pctx.Metadata` for use by later plugins

### Error Handling

- **Graceful degradation**: Non-critical plugins (logging, metrics) should log errors but not fail the request
- **Clear rejection reasons**: Always set `pctx.Reason` when rejecting to help users understand the issue
- **Use appropriate stages**: Register error-handling plugins at `on_error` stage

### Security

- **Input validation**: Always validate and sanitize configuration values in `Init()`
- **Memory limits**: Set `max_keys` and `max_entries` to prevent unbounded memory growth
- **Sensitive data**: Avoid logging sensitive information from requests/responses

### Configuration Tips

- **Use store_id**: Share state between before/after instances of the same plugin
- **Enable selectively**: Only enable plugins you need to minimize overhead
- **Test thoroughly**: Verify plugin behavior with edge cases before production deployment
