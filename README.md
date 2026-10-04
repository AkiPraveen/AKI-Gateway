# AKI-Gateway

Minimal gateway that routes LLM requests between a fast model (Gemini Flash Lite) and a thinking model (Grok 4.7). Side project focused on system design. Status: planning. No implementation yet.

## Stack

- TypeScript on Cloudflare Workers. Scale to zero, no servers.
- Hono for routing, middleware, and streaming.
- Cloudflare D1 (SQLite) for request history.

## Request lifecycle

1. Middleware assigns `request_id`, records start time.
2. Router selects tier: `fast` or `think`.
3. Direct provider adapter executes the call. On eligible failure, OpenRouter adapter executes it.
4. Response is returned or streamed to the client in a normalized format.
5. D1 row is written asynchronously after the response.

## What was explored

### 1. Normalizing provider output

Gemini and xAI differ in request shape, response shape, streaming events, error bodies, and usage fields. Each provider gets an adapter that maps to and from internal types: request, response, stream chunk, error, usage. The rest of the gateway sees only internal types. Responses include `tier` and `provider_path` (`direct` | `openrouter`).

### 2. Raw fetch adapters vs SDK

Rejected: Vercel AI SDK as the provider layer. It owns the translation, streaming normalization, and usage mapping, which are the core of a gateway. Adapters would reduce to config.

Chosen: hand-written adapters over `fetch`. Full control over timeouts, retries, error mapping, and token accounting. Cost: two streaming parsers and two auth schemes to maintain. xAI is OpenAI-compatible, so its adapter is thin. Gemini requires real translation. Generic utilities (SSE parsing, schema validation) may use small libraries.

### 3. Fallback to OpenRouter

Primary path: direct provider API. Fallback: OpenRouter, which exposes both models through one OpenAI-compatible API and reuses most of the xAI adapter's translation.

| Condition | Action |
| --- | --- |
| Timeout, connection error, 5xx | Switch to OpenRouter immediately |
| 429 | Switch to OpenRouter immediately |
| Other 4xx (bad request, auth) | No fallback. Return error. Config or client bug. |

- No retry against the failing provider. OpenRouter is the retry.
- Fallback keeps the same model. No tier downgrade.
- Tight per-tier timeouts so fallback triggers fast.
- Later: circuit breaker to skip the direct path after repeated failures.

### 4. Router: heuristic vs classifier

No client override. The router decides alone.

- v1: heuristic. Inputs: prompt length, keyword signals. Zero added latency or cost.
- Later: LLM classifier (simple vs complex) via the direct Gemini Flash Lite adapter.

Classifier concerns: extra round trip on every request, per-request cost, misclassification, prompt text can game routing toward the expensive tier. If the classifier fails, default to the heuristic or `fast`; never block the request and never fall back to OpenRouter for classification. The router sits behind one interface so both strategies can be compared on the same prompts using logged data.

### 5. Queryable observability in D1

Every request writes one row, including failures. Draft `requests` columns:

- `id`, `created_at`
- `tier_chosen`, `router_type` (`heuristic` | `classifier`)
- `classifier_ran`, `classifier_decision` (`simple` | `complex`, nullable), `classifier_tokens`
- `provider_path`, `provider`, `model`, `fallback_reason`
- `input_tokens`, `output_tokens`, `reasoning_tokens`, `cached_tokens`
- `estimated_cost_usd`, `reported_cost_usd`
- `latency_ms`, `status` (`ok` | `error`), `error_code`

Full prompts are not stored by default (privacy, size).

### 6. Async logging off the request path

D1 insert runs after the response via the Workers `waitUntil` mechanism. Write failures are caught and logged; they never affect the client response. Later option: queue with retries for lossless logging.

### 7. Streaming

- Client format: custom minimal SSE events (`delta`, `usage`, `done`, `error`). Not OpenAI-compatible. Evolves with the project.
- Each adapter parses provider SSE and yields normalized chunks. Hono `streamSSE` writes them to the client.
- Usage often arrives only in the final chunk. Tokens are accumulated in-flight; the D1 row is written after stream end.
- Fallback is only possible before the first byte is sent. Mid-stream failure emits an `error` event and is logged.
- Workers CPU limits count active compute, not time spent waiting on I/O, so long streams are cheap. Each request makes at most a few subrequests.

### 8. Language and runtime

| | TypeScript + Workers + Hono | Python + FastAPI |
| --- | --- | --- |
| Deployment | Serverless, edge, scale to zero | Container or VM to operate |
| Database | Native D1 binding | Postgres or SQLite over network, pooling |
| Streaming | Workable, more verbose | Mature, ergonomic (httpx, StreamingResponse) |
| Types | Static types map to normalized types | Pydantic |
| AI ecosystem | Smaller | Larger, mostly irrelevant here |

"Python is the AI language" applies to training, RAG, and agent frameworks. A gateway is an HTTP proxy with a routing decision. Chosen: TypeScript on Workers, driven by D1, scale to zero, and low operational overhead.

### 9. Normalized cost monitoring

Provider cost data is inconsistent:

- Gemini: token counts in `usageMetadata` (prompt, output, thinking). No dollar amount.
- xAI: OpenAI-style `usage` with token counts, including reasoning tokens. Dollar cost not relied on.
- OpenRouter: can report per-request cost in its usage data.

Approach:

- Always store raw token counts by type. Never discard them.
- `model_prices` table: provider, model, per-million-token price per token type, `effective_from`. Prices are versioned by inserting rows, never overwritten.
- `estimated_cost_usd` computed at write time from the price in effect. Recomputable from raw counts.
- `reported_cost_usd` stored when a provider supplies it. OpenRouter reported cost vs local estimate validates the price table.
- Classifier calls are tracked separately so routing overhead cost is visible.

Scope: raw token counts in v1. Price table and cost queries after streaming.

## Milestones

1. Normalized types; Gemini and xAI direct adapters (non-streaming).
2. Heuristic router.
3. OpenRouter fallback.
4. D1 request logging.
5. Streaming.
6. Cost table and queries.
7. Classifier router; compare against heuristic.

## Configuration

Secrets are set as Workers secrets and referenced by name only. Never commit real values.

```
GEMINI_API_KEY=YOUR_GEMINI_API_KEY
XAI_API_KEY=YOUR_XAI_API_KEY
OPENROUTER_API_KEY=YOUR_OPENROUTER_API_KEY
```

Database bindings and account identifiers are environment-specific and are not stored in this repository.
