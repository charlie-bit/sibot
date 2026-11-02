# sibot — Technical Design

**Status:** draft (v0) · **Owner:** @charlie-bit · **Last updated:** 2026-11-02

## 1. What sibot is

sibot is an **AI control center**: one place to talk to many models from many providers,
switch between them at will, and keep a single continuous conversation while doing so.

The problem it solves: every model now lives behind a different API with a different
message format, stream protocol, tool-call encoding, and usage schema. Applications end up
wired to one vendor, and switching means a rewrite. sibot puts a stable internal contract
in front of all of them, then manages sessions, routing, cost, and history on top.

### v1 capabilities

1. **Multi-provider** — Anthropic, OpenAI, and any OpenAI-compatible endpoint (gateways,
   local runtimes such as Ollama / vLLM).
2. **Model switching** — change model between turns, or per request, without losing the
   conversation.
3. **Routing policy** — explicit model, capability-constrained choice, fallback chain on
   error / rate limit / timeout.
4. **Sessions** — persisted conversations with full history, replay, and fork.
5. **Usage ledger** — tokens, latency, and cost per run, per model, per session.
6. **Interfaces** — HTTP + SSE API first; CLI and web UI on top of the same API.

## 2. Goals and non-goals

### Goals

- A provider contract stable enough that adding a provider never touches core logic.
- Zero-friction switching: the same transcript works everywhere.
- Local-first: one process, one SQLite file, no infrastructure.
- Honest accounting: every token and cent attributable to a run.
- Boring, typed, testable TypeScript.

### Non-goals (v1)

- Multi-tenant hosting, accounts, RBAC.
- Training, fine-tuning, embeddings search.
- Being a full LiteLLM replacement for arbitrary third-party clients **(see Q1)**.
- Agent marketplace / plugin directory.
- Mobile or desktop apps.

## 3. Key decisions

| # | Decision | Rationale |
|---|---|---|
| D1 | TypeScript, Node ≥ 22, ESM, `pnpm` | Requested; ESM-only avoids dual-format pain in a greenfield repo |
| D2 | Own canonical types + hand-written adapters for v1 | The abstraction **is** the product; delegating it to a SDK means our contract is someone else's |
| D3 | Two adapters cover most of the market: **OpenAI-compatible** + **Anthropic native** | One generic adapter (baseURL + key) unlocks gateways, local runtimes, and most vendors |
| D4 | Single package, layered directories — not a monorepo | Split into packages only when a second consumer exists |
| D5 | SQLite (WAL) for sessions/usage | Local-first, zero ops, full SQL; swap the repository layer later if needed |
| D6 | HTTP + SSE as the only transport contract | CLI, web UI, and scripts all consume one API; SSE beats WebSocket for one-way streams |
| D7 | Session transcript stored canonically; provider-specific payloads kept in a sidecar column | Enables switching and replay without re-deriving vendor structures |
| D8 | Secrets from env or OS keychain, never from the repo or the DB | Standard; also matches `CONTRIBUTING.md` §2 |
| D9 | Bind `127.0.0.1` and require a bearer token | "Control center" implies API keys for many providers sitting behind one port |

## 4. Architecture

```
┌──────────┐   ┌──────────┐   ┌──────────┐
│   CLI    │   │  Web UI  │   │ Scripts  │        clients (all speak the same API)
└────┬─────┘   └────┬─────┘   └────┬─────┘
     └──────────────┼──────────────┘
                    │  HTTP + SSE
            ┌───────▼────────┐
            │    server/     │  auth, validation, streaming
            └───────┬────────┘
                    │
            ┌───────▼────────────────────────────┐
            │            core/                   │
            │  session service · router · ledger │
            └───────┬───────────────────┬────────┘
                    │                   │
        ┌───────────▼──────┐   ┌────────▼─────────┐
        │   storage/       │   │   providers/     │
        │  SQLite repos    │   │  registry +      │
        │  sessions, runs  │   │  adapters        │
        └──────────────────┘   └────────┬─────────┘
                                        │
                     ┌──────────────────┼──────────────────┐
              OpenAI-compatible     Anthropic          (future adapters)
              gateways, Ollama      Messages API
```

Layering rule: `server` depends on `core`; `core` depends on `providers` and `storage`
**contracts only**, never on adapter implementations or the HTTP layer.

## 5. Canonical model layer

Everything inside sibot is expressed in one normalized shape. Adapters translate at the
edge, in both directions.

```ts
type Role = "system" | "user" | "assistant" | "tool";

type ContentPart =
  | { type: "text"; text: string }
  | { type: "image"; mediaType: string; data: string }        // base64 or URL
  | { type: "tool_call"; id: string; name: string; input: unknown }
  | { type: "tool_result"; toolCallId: string; content: string; isError?: boolean }
  | { type: "reasoning"; text: string };                       // provider thinking blocks

interface Message {
  role: Role;
  content: ContentPart[];
  /** Provider-specific payload needed to replay this message verbatim. */
  providerMeta?: Record<string, unknown>;
}

interface ChatRequest {
  model: ModelRef;                    // "anthropic:claude-sonnet-5-5"
  messages: Message[];
  system?: string;
  tools?: ToolSpec[];
  temperature?: number;
  maxTokens?: number;
  stop?: string[];
  responseFormat?: "text" | "json";
}

type StreamEvent =
  | { type: "text_delta"; text: string }
  | { type: "reasoning_delta"; text: string }
  | { type: "tool_call"; id: string; name: string; input: unknown }
  | { type: "usage"; inputTokens: number; outputTokens: number; cachedInputTokens?: number }
  | { type: "done"; finishReason: "stop" | "length" | "tool_calls" | "error" }
  | { type: "error"; error: ProviderError };
```

Rules that keep this honest:

- Unknown provider fields are **dropped, not smuggled** — except under `providerMeta`,
  which only the originating adapter may read.
- A message that carries `providerMeta` from a different provider is still valid: the new
  adapter ignores the sidecar and uses the normalized parts.
- Token counts are always from the provider when available; never estimated silently.

## 6. Provider layer

### Contract

```ts
interface Provider {
  readonly id: string;                     // "anthropic" | "openai-compatible"
  readonly label: string;
  listModels(): Promise<ModelInfo[]>;
  stream(req: ChatRequest, signal: AbortSignal): AsyncIterable<StreamEvent>;
}

interface ModelInfo {
  ref: ModelRef;                            // "openai-compatible:qwen3-coder"
  contextWindow: number;
  maxOutputTokens?: number;
  capabilities: Capability[];               // "tools" | "vision" | "reasoning" | "json"
  pricing?: { inputPerMTok: number; outputPerMTok: number; currency: "USD" };
  deprecation?: { at: string; replacement?: ModelRef };
}
```

`stream()` is the only required method. Blocking completion is built on top of it by the
core, so there is exactly one code path to test.

### Adapters

| Adapter | Covers | Notes |
|---|---|---|
| `openai-compatible` | OpenAI, Azure-style gateways, OpenRouter, DeepSeek, Qwen, Mistral, Ollama, vLLM, llama.cpp | Config differs only by `baseURL`, key, and a per-model capability/pricing table |
| `anthropic` | Claude Messages API | Native: `system` is a top-level field, tool results are content blocks, thinking blocks map to `reasoning` |
| *(later)* `gemini` | Google Generative Language API | Only if a real use case appears — do not build speculatively |

### Error taxonomy

Every adapter maps its failures into one set, so routing and retry behave uniformly:

```ts
type ProviderErrorCode =
  | "auth" | "rate_limit" | "quota" | "context_overflow"
  | "invalid_request" | "server_error" | "timeout" | "network" | "aborted" | "unknown";

class ProviderError extends Error {
  code: ProviderErrorCode;
  retryable: boolean;                       // rate_limit/server_error/timeout/network
  retryAfterMs?: number;
  provider: string;
  status?: number;
}
```

`context_overflow` is special: the router may transparently retry the same request on a
model with a larger window, and the client is told it happened.

## 7. Model catalog and routing

### ModelRef

```
<provider-id>:<model-id>        anthropic:claude-sonnet-5-5
                                 openai-compatible:llama3.3:70b      (last segment may contain ':')
```

Parsing splits on the **first** colon only.

### Catalog

Models come from three places, merged with this precedence:
`config.models` (hand-declared) → adapter `listModels()` (live) → built-in table (fallback
for capability and pricing when the provider reports none).

The catalog is cached in memory with a TTL and exposed at `GET /v1/models`, including a
`health` field from the last probe.

### Selection

Resolution order for every request:

1. **Explicit** — caller passed `model`.
2. **Session default** — `sessions.default_model`.
3. **Config default** — `routing.default`.
4. **Fallback chain** — `routing.fallbacks: ModelRef[][]`, tried in order on retryable
   errors or when a required capability is missing.

Capability requirements are checked against the catalog before dispatch; a mismatch is an
`invalid_request` error naming the missing capability, never a silent degradation.

### Mid-session switching

The transcript is canonical, so switching is a re-serialization, not a migration:

```http
POST /v1/sessions/{id}/messages
{ "input": "continue", "model": "openai-compatible:gpt-5.1" }
```

`messages.seq` is monotonic per session; each row records the `model_ref` that produced it,
so a mixed-model transcript replays exactly. Fork creates a new session with a copied prefix.

## 8. Sessions and storage

SQLite, WAL mode, `~/.sibot/sibot.db` by default.

```sql
CREATE TABLE sessions (
  id            TEXT PRIMARY KEY,
  title         TEXT,
  default_model TEXT,
  created_at    INTEGER NOT NULL,
  updated_at    INTEGER NOT NULL,
  metadata      TEXT                          -- JSON
);

CREATE TABLE messages (
  id            TEXT PRIMARY KEY,
  session_id    TEXT NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
  seq           INTEGER NOT NULL,
  role          TEXT NOT NULL,
  content       TEXT NOT NULL,                -- canonical ContentPart[] as JSON
  provider_meta TEXT,                         -- sidecar, adapter-private
  model_ref     TEXT,
  created_at    INTEGER NOT NULL,
  UNIQUE (session_id, seq)
);

CREATE TABLE runs (
  id             TEXT PRIMARY KEY,
  session_id     TEXT NOT NULL REFERENCES sessions(id) ON DELETE CASCADE,
  message_id     TEXT REFERENCES messages(id) ON DELETE SET NULL,
  provider       TEXT NOT NULL,
  model          TEXT NOT NULL,
  status         TEXT NOT NULL,               -- running | ok | error | aborted
  error_code     TEXT,
  latency_ms     INTEGER,
  input_tokens   INTEGER,
  output_tokens  INTEGER,
  cached_tokens  INTEGER,
  cost_usd       REAL,
  created_at     INTEGER NOT NULL
);

CREATE INDEX runs_session_created ON runs (session_id, created_at);
```

`runs` is the ledger: written at stream start (`running`), updated at completion. A crashed
process leaves `running` rows, which the next start marks `aborted` — never silently deleted.

## 9. Server API

Base path `/v1`, bearer token in `Authorization` (see D9).

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/health` | liveness, no auth |
| `GET` | `/models` | catalog + last probe result |
| `POST` | `/models/{ref}/probe` | one-token call to verify key/model |
| `POST` | `/sessions` | create |
| `GET` | `/sessions` | list, paginated by `updated_at` |
| `GET` | `/sessions/{id}` | session + messages |
| `PATCH` | `/sessions/{id}` | title, default model |
| `DELETE` | `/sessions/{id}` | delete |
| `POST` | `/sessions/{id}/fork` | copy prefix into a new session |
| `POST` | `/sessions/{id}/messages` | append input, stream the reply (SSE) |
| `POST` | `/runs/{id}/abort` | cancel an in-flight stream |
| `GET` | `/usage` | aggregate tokens/cost by day, model, session |

### Stream events (SSE)

```
event: text_delta      data: {"text":"..."}
event: reasoning_delta data: {"text":"..."}
event: tool_call       data: {"id":"...","name":"...","input":{...}}
event: usage           data: {"inputTokens":12,"outputTokens":340}
event: router          data: {"from":"anthropic:x","to":"openai-compatible:y","reason":"rate_limit"}
event: error           data: {"code":"rate_limit","retryable":true,"message":"..."}
event: done            data: {"finishReason":"stop","runId":"..."}
```

`router` makes fallbacks visible instead of magic.

## 10. CLI

```
sibot serve                     # start the server
sibot chat [--model ref]        # interactive session against the local server
sibot chat --resume <id>
sibot models [--probe]
sibot usage [--since 7d]
sibot config validate
```

The CLI is a thin client: it speaks the same HTTP API as the web UI. No logic lives in it
that the API cannot do.

## 11. Configuration and secrets

`sibot.config.ts` at the repo root (or `~/.sibot/config.ts`), typed and validated at start:

```ts
import { defineConfig, env } from "sibot/config";

export default defineConfig({
  providers: {
    anthropic: { apiKey: env("ANTHROPIC_API_KEY") },
    openai: { baseURL: "https://api.openai.com/v1", apiKey: env("OPENAI_API_KEY") },
    local: { baseURL: "http://127.0.0.1:11434/v1" },   // keyless
  },
  models: {
    "local:llama3.3": { contextWindow: 131_072, capabilities: ["tools"] },
  },
  routing: {
    default: "anthropic:claude-sonnet-5-5",
    fallbacks: [["openai:gpt-5.1", "local:llama3.3"]],
  },
  server: { port: 7801, host: "127.0.0.1", token: env("SIBOT_TOKEN") },
  storage: { path: "~/.sibot/sibot.db" },
});
```

- Config is validated with a schema (invalid config fails fast with a readable diff).
- `env()` is resolved at load time and only ever appears in memory.
- Logs, error messages, and the `usage` API redact anything key-shaped.
- A missing key disables that provider and marks it `unavailable` in `/v1/models`, rather
  than crashing the process.

## 12. Observability and cost

- Structured JSON logs (`pino`), one line per run: provider, model, latency, tokens, cost,
  status, fallback hops. **No prompt or response text by default.**
- `--log-prompts` opt-in, off by default, warns on enable.
- Cost computed at run end from catalog pricing; unknown pricing records `cost_usd = NULL`
  and shows as "unknown" rather than `0`.
- Nothing phones home. No telemetry endpoint.

## 13. Security posture

- Default bind `127.0.0.1`; exposing to a network requires setting `host` **and** a token —
  the server refuses to start on `0.0.0.0` without one.
- Constant-time token comparison; token never logged.
- Request bodies validated at the boundary (schema), size-capped.
- Provider URLs come from config only — no user-supplied `baseURL` per request (SSRF guard).
- Tool execution (later) is explicit, allow-listed, and off by default.
- Dependencies: minimal, pinned, `pnpm audit` in CI.

## 14. Testing

- `vitest` for unit and integration.
- **Contract tests**: one shared suite that every adapter must pass, driven by recorded
  fixtures — no live API calls in CI. A real-call suite runs only with credentials set,
  manually.
- Golden-file tests for canonical → provider serialization (system prompt placement, tool
  call shapes, image blocks).
- Router tests: fallback ordering, capability rejection, context-overflow retry, abort.
- SSE tests: event order and termination, including mid-stream abort.
- SQLite tests against a temp file (not `:memory:`) so WAL and migration paths are real.

## 15. Repository layout and tooling

```
src/
  core/          session service, router, ledger, canonical types
    types.ts
    session.ts
    router.ts
    ledger.ts
  providers/
    contract.ts
    registry.ts
    adapters/openai-compatible.ts
    adapters/anthropic.ts
  storage/
    db.ts            migrations
    sessions.ts
    runs.ts
  server/
    app.ts           routes
    sse.ts
    auth.ts
  cli/
    index.ts
  config/
    schema.ts
    load.ts
test/
  contract/
  fixtures/
  core/
  server/
tech.md
CONTRIBUTING.md
```

Tooling: `typescript --strict`, `tsx` for dev, `vitest`, `eslint` (flat config) + `prettier`,
`pnpm` scripts `dev / build / test / lint / typecheck`, GitHub Actions running
`typecheck + lint + test` on every push. No bundler in v1 — Node runs the TypeScript output
via `tsc`.

## 16. Milestones

| Milestone | Contents | Done when |
|---|---|---|
| **M0 Skeleton** | tooling, config loader, logging, health endpoint | `pnpm dev` starts and `GET /health` answers |
| **M1 Providers** | contract + two adapters + catalog + probe | contract suite green for both adapters against fixtures |
| **M2 Sessions** | SQLite, session service, streaming server, CLI chat | a real conversation persists and resumes |
| **M3 Switching** | explicit switch, fallbacks, capability routing, ledger, usage API | switching mid-session works across both adapters; costs reconcile |
| **M4 Surfaces** | web UI, fork/replay, prompt search | UI usable for daily driving |

## 17. Open questions

| # | Question | Why it matters |
|---|---|---|
| Q1 | Is sibot only for its owner on localhost, or should third-party clients be able to point at it as an OpenAI-compatible gateway? | Changes auth, rate limiting, and whether we expose `/v1/chat/completions` verbatim |
| Q2 | Which surface ships first: CLI or web UI? | Shapes M2 scope |
| Q3 | Do tools / MCP execution belong in v1, or is sibot chat-only first? | Tool support is the largest single sub-system |
| Q4 | Should mid-turn switching (interrupt a stream, resume on another model) be supported, or only between turns? | Mid-turn needs partial-output replay semantics in every adapter |
| Q5 | Local models in scope from day one (Ollama / vLLM)? | Second adapter already covers them, but pricing/health differ |
| Q6 | Any privacy constraint that forbids storing prompts on disk by default? | Would force an ephemeral-session mode |
