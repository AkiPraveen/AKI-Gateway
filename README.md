# AKI-Gateway

- Routes LLM requests between Gemini Flash Lite (fast) and Grok 4.7 (think).
- Stack: TypeScript, Cloudflare Workers, Hono, D1.
- Status: planning.

## Explored

### 1. Normalized output
- Per-provider adapters map to internal request, response, chunk, error, usage types.
- Rest of gateway is provider-agnostic.

### 2. Raw fetch vs SDK
- Hand-written fetch adapters, no SDK.
- Translation, streaming, usage mapping are the core; owning them is the point.

### 3. Fallback
- Direct API first, OpenRouter on timeout, 5xx, or 429.
- No fallback on other 4xx.
- Same model, no retry on failing provider.

### 4. Router
- v1: heuristic (length, keywords), no client override.
- Later: classifier. Costs latency and tokens, can be gamed.

### 5. Observability
- One D1 row per request: tier, classifier decision, provider path, tokens, latency, status, error.
- No full prompts.

### 6. Async logging
- D1 write after response via waitUntil.
- Write failures caught; request path unaffected.

### 7. Streaming
- Custom SSE events: delta, usage, done, error.
- Fallback only before first byte.
- Log written after stream end.

### 8. Language
- TypeScript on Workers: D1 binding, scale to zero, no servers.
- Python streaming is nicer, but deployment heavier.

### 9. Cost
- Store raw token counts per type.
- Compute cost from versioned model_prices table.
- Compare with OpenRouter reported cost.

## Milestones
- Adapters, heuristic router, fallback, D1 logging, streaming, cost, classifier.

## Config
- Workers secrets, by name only: GEMINI_API_KEY, XAI_API_KEY, OPENROUTER_API_KEY.
- Values are never committed.
