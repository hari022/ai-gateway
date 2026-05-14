# Routing Strategies

This document provides a comprehensive guide to the routing strategies available in the AI Gateway. Routing strategies determine how incoming requests are distributed across configured provider targets.

## Table of Contents

- [Overview](#overview)
- [Available Strategies](#available-strategies)
  - [Single](#single)
  - [Fallback](#fallback)
  - [Load Balance](#load-balance)
  - [Least Latency](#least-latency)
  - [Cost Optimized](#cost-optimized)
  - [Conditional](#conditional)
  - [Content-Based](#content-based)
  - [A/B Test](#ab-test)
- [Configuration Reference](#configuration-reference)
- [Target Configuration](#target-configuration)
- [Best Practices](#best-practices)

---

## Overview

The AI Gateway uses a pluggable strategy system to route requests to one or more AI provider targets. Each strategy implements a common interface and can be selected via the `strategy.mode` configuration field.

```yaml
strategy:
  mode: fallback  # Options: single | fallback | loadbalance | conditional | content-based | ab-test | least-latency | cost-optimized
```

All strategies automatically filter targets based on model compatibility. A target is only considered if its provider supports the requested model.

---

## Available Strategies

### Single

**Mode:** `single`

The simplest routing strategy. All requests are sent to a single configured provider target.

**Use Cases:**
- Development and testing environments
- Single-provider deployments
- When no failover or load distribution is needed

**Configuration:**

```yaml
strategy:
  mode: single

targets:
  - virtual_key: openai
```

**Behavior:**
- Routes all requests to the first target in the list
- Fails immediately if the provider is unavailable or doesn't support the requested model

---

### Fallback

**Mode:** `fallback`

Provides high availability by trying targets in order. If one provider fails, the request is automatically retried with the next provider in the list.

**Use Cases:**
- Production environments requiring high availability
- Multi-provider deployments with primary/backup configuration
- Graceful degradation during provider outages

**Configuration:**

```yaml
strategy:
  mode: fallback

targets:
  - virtual_key: openai
    retry:
      attempts: 3
      on_status_codes: [429, 502, 503]
      initial_backoff_ms: 100
  - virtual_key: anthropic
    retry:
      attempts: 2
  - virtual_key: gemini
```

**Retry Configuration Options:**

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `attempts` | int | 1 | Maximum attempts per target (1 = no retries) |
| `on_status_codes` | []int | [] | Limit retries to specific HTTP status codes. Empty = retry on any error |
| `initial_backoff_ms` | int | 100 | Base backoff for exponential back-off: `delay = initial_backoff_ms * 2^(attempt-1)` |

**Behavior:**
- Targets are tried in declaration order
- Per-target retry policies allow fine-grained control
- Exponential backoff between retry attempts
- Skips targets that don't support the requested model
- Only moves to the next target after exhausting retries on the current target

---

### Load Balance

**Mode:** `loadbalance`

Distributes requests across multiple providers using weighted random selection. Useful for spreading load and utilizing multiple API quotas.

**Use Cases:**
- Distributing load across multiple API keys/accounts
- Utilizing rate limits from multiple providers
- Cost optimization by mixing providers

**Configuration:**

```yaml
strategy:
  mode: loadbalance

targets:
  - virtual_key: openai
    weight: 70
  - virtual_key: anthropic
    weight: 30
```

**Behavior:**
- Targets are selected randomly based on their relative weights
- Weight of 0 or omitted is treated as 1 (equal distribution)
- Only compatible targets (those supporting the requested model) are considered
- Each request is independently routed (no sticky sessions)

**Weight Calculation Example:**
```
openai:    70 / (70 + 30) = 70% of traffic
anthropic: 30 / (70 + 30) = 30% of traffic
```

---

### Least Latency

**Mode:** `least-latency`

Routes requests to the provider with the lowest observed p50 (median) latency. The gateway tracks latency metrics for each provider and optimizes routing decisions in real-time.

**Use Cases:**
- Latency-sensitive applications
- Auto-optimization without manual tuning
- Dynamic provider selection based on real performance

**Configuration:**

```yaml
strategy:
  mode: least-latency

targets:
  - virtual_key: openai
  - virtual_key: anthropic
  - virtual_key: gemini
```

**Behavior:**
- Tracks p50 latency for each provider
- During cold start, rotates through unseen providers to gather initial latency samples
- Once all providers are sampled, consistently routes to the fastest one
- Automatically adapts as provider performance changes over time
- Only considers providers that support the requested model

---

### Cost Optimized

**Mode:** `cost-optimized`

Routes requests to the cheapest compatible provider based on estimated input cost from the model catalog. Useful for minimizing costs without manual provider selection.

**Use Cases:**
- Cost-conscious deployments
- Automatic cost optimization across providers
- Budget management

**Configuration:**

```yaml
strategy:
  mode: cost-optimized

targets:
  - virtual_key: openai
  - virtual_key: anthropic
  - virtual_key: gemini
  - virtual_key: deepseek
```

**Behavior:**
- Estimates prompt tokens (~4 characters per token)
- Looks up pricing from the model catalog for each compatible provider
- Routes to the provider with the lowest estimated input cost
- Falls back to the first compatible provider if no pricing data is available
- Only considers providers that support the requested model

---

### Conditional

**Mode:** `conditional`

Routes requests based on request field matching rules. Useful for directing specific models to specific providers.

**Use Cases:**
- Model-specific routing (e.g., GPT models to OpenAI, Claude to Anthropic)
- Separating traffic by model family
- Custom routing logic based on request properties

**Configuration:**

```yaml
strategy:
  mode: conditional
  conditions:
    - key: model
      value: gpt-4o
      target_key: openai
    - key: model
      value: gpt-4o-mini
      target_key: openai
    - key: model_prefix
      value: claude
      target_key: anthropic
    - key: model_prefix
      value: gemini
      target_key: gemini

targets:
  - virtual_key: openai
  - virtual_key: anthropic
  - virtual_key: gemini
```

**Condition Keys:**

| Key | Description |
|-----|-------------|
| `model` | Exact match on the requested model name |
| `model_prefix` | Prefix match on the requested model name |

**Behavior:**
- Rules are evaluated in declaration order
- First matching rule wins
- Falls back to the first target if no rule matches
- Does NOT check model support — ensure your rules route to correct providers

---

### Content-Based

**Mode:** `content-based`

Routes requests based on the textual content of prompt messages. Enables intelligent routing based on what the user is asking about.

**Use Cases:**
- Routing code-related questions to specialized coding models
- Directing specific topics to domain-expert models
- Content filtering (routing away from certain topics)
- Cost optimization based on query complexity

**Configuration:**

```yaml
strategy:
  mode: content-based
  content_conditions:
    - type: prompt_regex
      value: "(?i)\\b(code|function|class|implement|debug)\\b"
      target_key: deepseek
    - type: prompt_contains
      value: "translate"
      target_key: gemini
    - type: prompt_not_contains
      value: "confidential"
      target_key: openai

targets:
  - virtual_key: deepseek
  - virtual_key: gemini
  - virtual_key: openai
```

**Condition Types:**

| Type | Description |
|------|-------------|
| `prompt_contains` | Case-insensitive substring match on user messages |
| `prompt_not_contains` | True when NO user message contains the value |
| `prompt_regex` | Go regular expression match on user messages |

**Behavior:**
- Rules are evaluated in declaration order
- First matching rule wins
- Falls back to the first target if no rule matches
- Only examines messages with `role: "user"`
- Regex patterns are compiled at startup; invalid patterns cause startup failure

**Regex Tips:**
- Use `(?i)` at the start for case-insensitive matching
- Use `\\b` for word boundaries
- Escape special characters with double backslashes in YAML

---

### A/B Test

**Mode:** `ab-test`

Implements weighted random traffic splitting across labeled variants. Essential for comparing model quality, performance, or cost during migrations or experiments.

**Use Cases:**
- Gradual model migrations (e.g., GPT-4 to GPT-4o)
- Quality comparisons between providers
- Cost/performance experiments
- Canary deployments for new models

**Configuration:**

```yaml
strategy:
  mode: ab-test
  ab_variants:
    - target_key: openai
      weight: 80
      label: control
    - target_key: anthropic
      weight: 20
      label: challenger

targets:
  - virtual_key: openai
  - virtual_key: anthropic
```

**Variant Configuration:**

| Field | Type | Description |
|-------|------|-------------|
| `target_key` | string | The virtual_key of the provider for this variant |
| `weight` | float | Relative traffic share (0 = equal distribution) |
| `label` | string | Human-readable identifier for logging/analytics |

**Behavior:**
- Variants are selected randomly based on their relative weights
- Selected variant is logged with field `ab_variant` on every request
- All traffic goes to real providers (not shadow traffic)
- Weight of 0 is treated as 1 (equal distribution)
- Negative weights cause startup failure

**Observability:**
The selected variant label is emitted as a structured log field (`ab_variant`) on every request, enabling correlation with your observability stack (e.g., filtering metrics by variant in Grafana/Datadog).

---

## Configuration Reference

### Strategy Configuration Schema

```yaml
strategy:
  mode: string  # Required: single | fallback | loadbalance | conditional | content-based | ab-test | least-latency | cost-optimized
  
  # For conditional mode
  conditions:
    - key: string        # model | model_prefix
      value: string      # Value to match
      target_key: string # Provider virtual_key
  
  # For content-based mode
  content_conditions:
    - type: string       # prompt_contains | prompt_not_contains | prompt_regex
      value: string      # Substring or regex pattern
      target_key: string # Provider virtual_key
  
  # For ab-test mode
  ab_variants:
    - target_key: string # Provider virtual_key
      weight: float      # Relative traffic share
      label: string      # Variant identifier for logging
```

---

## Target Configuration

Targets define the provider endpoints available for routing. All strategies use the same target configuration.

```yaml
targets:
  - virtual_key: openai
    weight: 70                    # For loadbalance strategy
    retry:                        # For fallback strategy
      attempts: 3
      on_status_codes: [429, 502, 503]
      initial_backoff_ms: 100
    circuit_breaker:              # Optional resilience
      failure_threshold: 5
      success_threshold: 1
      timeout: 30s
```

### Circuit Breaker

Optional per-target circuit breaker for additional resilience:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `failure_threshold` | int | 5 | Consecutive failures before opening |
| `success_threshold` | int | 1 | Successes in half-open state to close |
| `timeout` | string | "30s" | Duration the circuit stays open |

---

## Best Practices

### Choosing a Strategy

| Scenario | Recommended Strategy |
|----------|---------------------|
| Single provider, simple setup | `single` |
| High availability required | `fallback` |
| Multiple API quotas to utilize | `loadbalance` |
| Latency-critical applications | `least-latency` |
| Budget-conscious deployments | `cost-optimized` |
| Model-family specific routing | `conditional` |
| Topic-based intelligent routing | `content-based` |
| Model migration or experiments | `ab-test` |

### Combining Strategies

For complex requirements, consider:

1. **Fallback + Retry**: Configure retry policies on each target for maximum resilience
2. **Conditional + Content-Based**: Use conditional for model routing, content-based for topic routing
3. **A/B Test + Monitoring**: Always pair A/B testing with proper observability to measure results

### Performance Considerations

- **Least Latency**: Cold start requires sampling all providers; initial requests may not be optimally routed
- **Cost Optimized**: Relies on accurate model catalog data; ensure catalog is up-to-date
- **Content-Based**: Regex matching adds slight overhead; use simple patterns when possible
- **A/B Test**: No performance overhead beyond random selection

### Resilience Patterns

1. **Always configure multiple targets** for production deployments
2. **Use retry configuration** with appropriate status code filtering
3. **Enable circuit breakers** to prevent cascading failures
4. **Monitor provider health** and adjust weights/order as needed
