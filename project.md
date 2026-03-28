# Velum — Product Healing Agent: Project Overview

> **Purpose:** This file exists to give AI coding assistants and new contributors a complete understanding of the Velum codebase. It is gitignored and not shipped.

---

## What Is Velum?

Velum is the **Product Healing Agent** — a zero-config behavioral pattern detection engine written in Go that detects hidden UX friction and product gaps from real user behavior, then tells your team what to fix first. You send it raw product analytics events (any domain — e-commerce, ride-hailing, streaming, fintech) via a single POST endpoint, and it:

1. **Learns your vocabulary** — classifies unknown words in event names via LLM (Groq)
2. **Discovers your schema** — identifies which JSON field is the user ID, timestamp, event name, etc., via LLM
3. **Normalizes events** — transforms raw JSON into a canonical internal format
4. **Reconstructs sessions** — groups events into user sessions and flow instances
5. **Tags behaviors** — detects explore, attempt, succeed, retry, abandon, hesitate, bypass, progress
6. **Detects anti-patterns** — aggregates behaviors into patterns like retry storms, silent abandonment, confusion loops
7. **Compares baselines** — tracks patterns historically and detects trends (increasing, decreasing, new)
8. **Diagnoses & recommends** — generates AI-powered analysis with prioritized product/UX healing recommendations

All from a **single `POST /api/v1/analyze`** request with raw JSON events.

---

## Tech Stack

| Component        | Technology                                    |
|------------------|-----------------------------------------------|
| Language         | Go 1.23+                                      |
| HTTP Router      | [chi/v5](https://github.com/go-chi/chi)       |
| Database         | PostgreSQL (vocab, property registry, baselines) |
| LLM Provider     | Groq (llama / GPT-oss models via REST API)    |
| Config           | YAML (`config.yaml`) + env var overrides      |
| Logging          | Go `log/slog` (text in dev, JSON in prod)     |
| Auth             | SHA-256 API key hash (`X-Infra-Key` header)   |
| Rate Limiting    | `go-chi/httprate`                             |

---

## Module & Import Path

```
module github.com/velum
```

All internal packages are under `internal/` — they are **not importable** from outside this module.

---

## Directory Structure

```
.
├── cmd/velum/main.go              # Entrypoint — loads config, starts HTTP server, handles graceful shutdown
├── config.yaml                    # Runtime config (gitignored — contains secrets)
├── example.config.yaml            # Template config checked into git
├── go.mod / go.sum                # Go module definition
│
├── internal/
│   ├── api/
│   │   ├── server.go              # Chi router setup: CORS, rate limiting, middleware, route registration
│   │   ├── handlers/
│   │   │   └── handler.go         # HTTP handlers (Health, Analyze), pipeline wiring, storage init
│   │   └── middleware/
│   │       └── security.go        # X-Infra-Key API key auth (SHA-256 constant-time comparison)
│   │
│   ├── canonical/
│   │   ├── context.go             # EventContext struct (dimensions, targets, conditions, measures)
│   │   └── registry.go            # Built-in dimension names, core field lists, recommended property checks
│   │
│   ├── config/
│   │   └── config.go              # Config structs, defaults, YAML loader, env var overrides (VELUM_*)
│   │
│   ├── layers/
│   │   ├── layer.go               # Layer interface, Pipeline orchestrator, ExecuteWithContext
│   │   │
│   │   ├── vocabagent/            # Layer 0 — Vocab Enricher
│   │   │   ├── agent.go           # Groq LLM client for word classification
│   │   │   ├── enricher.go        # Enricher layer: tokenize events → classify unknowns → store
│   │   │   ├── seed.go            # Seeds built-in vocabulary on first run
│   │   │   ├── storage.go         # VocabStorage interface
│   │   │   ├── storage_postgres.go # PostgreSQL implementation (vocabulary table)
│   │   │   ├── storage_memory.go  # In-memory implementation (tests)
│   │   │   ├── circuit_breaker.go # Circuit breaker wrapper for LLM calls
│   │   │   ├── types.go           # VocabEntry, Config, etc.
│   │   │   └── vocabulary.go      # Static built-in word classifications
│   │   │
│   │   ├── propertyagent/         # Layer 1 — Property/Context Agent
│   │   │   ├── agent.go           # Groq LLM client for property role classification
│   │   │   ├── enricher.go        # Enricher layer: classify unknown properties → store
│   │   │   ├── seed.go            # Seeds built-in dimensions on first run
│   │   │   ├── storage.go         # PropertyStorage interface
│   │   │   ├── storage_postgres.go # PostgreSQL implementation (property_registry table)
│   │   │   ├── circuit_breaker.go # Circuit breaker wrapper
│   │   │   └── types.go           # PropertyEntry, Config, etc.
│   │   │
│   │   ├── eventadapter/          # Layer 2 — Event Adapter
│   │   │   ├── adapter.go         # Normalizes raw events → CanonicalEvent using vocab + property lookups
│   │   │   └── vocabulary.go      # Static fallback vocabulary for token classification
│   │   │
│   │   ├── datamapper/            # Pre-Layer — Data Mapper (optional)
│   │   │   └── mapper.go          # Declarative field mapping (config-driven schema transforms)
│   │   │
│   │   ├── sessionflow/           # Layer 3 — Session Flow Reconstructor
│   │   │   ├── reconstructor.go   # Groups events → sessions → flow instances (retry cycle merging, folding)
│   │   │   └── types.go           # FlowInstance, SessionData, etc.
│   │   │
│   │   ├── behavior/              # Layer 4 — Behavior Analyzer
│   │   │   ├── analyzer.go        # Tags flows with behavioral signals (retry, abandon, hesitate, etc.)
│   │   │   └── types.go           # AnalysisContext, AnalyzedFlow, BehaviorTag, etc.
│   │   │
│   │   ├── pattern/               # Layer 5 — Pattern Detector
│   │   │   ├── detector.go        # Aggregates behaviors into anti-patterns across users
│   │   │   └── types.go           # DetectedPattern, PatternType constants
│   │   │
│   │   ├── baseline/              # Layer 6 — Baseline Comparator
│   │   │   ├── detector.go        # Compares current patterns vs historical snapshots, stores new snapshots
│   │   │   └── types.go           # ChangeResult, Config, etc.
│   │   │
│   │   ├── ai/                    # Layer 7 — AI Analyzer
│   │   │   ├── groq.go            # Groq LLM client for natural-language pattern analysis
│   │   │   ├── circuit_breaker.go # Circuit breaker wrapper
│   │   │   └── types.go           # AIAnalysis, Config, etc.
│   │   │
│   │   └── aggregation/           # Report Builder
│   │       ├── aggregator.go      # Extracts typed data from pipeline output, builds structured report
│   │       └── types.go           # ReportInput, AnalysisReport, etc.
│   │
│   └── storage/
│       ├── storage.go             # Storage interface (FetchBaselineSnapshots, StoreSnapshot, Cleanup, Ping, Close)
│       ├── factory.go             # NewStorage() — creates backend based on config
│       ├── postgres.go            # PostgreSQL implementation (pattern_snapshots_{project_id} tables)
│       └── memory.go              # In-memory implementation (tests)
```

---

## Processing Pipeline

Events flow through an 8-layer sequential pipeline. Each layer implements the `Layer` interface (`Name()` + `Process()`). Some layers implement `ContextAwareLayer` to receive batch-level metadata.

```
Raw JSON Events ([]map[string]interface{})
  │
  ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 0: Context Enricher (propertyagent)                          │
│    • Classifies unknown event properties into roles:                │
│      - Dimension (built-in list: device, country, platform, etc.)   │
│      - Measure (type inference: numeric values)                     │
│      - Target vs Condition (AI classification via Groq)             │
│    • Stores classifications in PostgreSQL (property_registry table) │
│    • Attaches EventContext to each event under "_context" key       │
│    • Optional — skipped if disabled or no API key                   │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 1: Vocab Enricher (vocabagent)                               │
│    • Tokenizes event names (snake_case, camelCase, kebab-case, dot) │
│    • Looks up each token in PostgreSQL vocab table                  │
│    • Unknown tokens batched → Groq LLM classifies as:              │
│      - Surface (the "what": checkout, payment, ride)                │
│      - Status  (the "state": failed, success, click)               │
│      - Flow    (grouping: cart, auth, booking)                      │
│      - Noise   (irrelevant: the, total, count)                     │
│    • Stores learned words to PostgreSQL for future requests         │
│    • Optional — skipped if disabled or no API key                   │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 2: Event Adapter (eventadapter)                              │
│    • Normalizes raw events into CanonicalEvent structs              │
│    • Uses dual-lookup: PostgreSQL vocab first → static fallback     │
│    • Extracts: Flow, Action, Status, UserID, SessionID, Timestamp  │
│    • Builds structured context from property registry               │
│    • Always active                                                  │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 3: Session Flow Reconstructor (sessionflow)                  │
│    • Groups events by user_id + session_id                          │
│    • Splits sessions at 30-minute inactivity gaps                   │
│    • Builds FlowInstance objects:                                   │
│      - Groups contiguous events by flow name                        │
│      - Entry-status events start new instances                      │
│      - Lifecycle events (session_end, app_closed) are filtered      │
│      - Surface-fallback: events without own flow fold into parent   │
│      - Retry cycle merging: fail→re-attempt collapses into 1 flow  │
│    • Always active                                                  │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 4: Behavior Analyzer (behavior)   [ContextAwareLayer]        │
│    • Analyzes each FlowInstance for behavioral signals:             │
│      - explore, attempt, succeed, retry, abandon, progress          │
│      - hesitate (long pauses), bypass (skipped steps)               │
│    • Classifies flow intent: "transact", "browse", "unknown"       │
│    • Receives AnalysisContext metadata (scope, funnel definitions)  │
│    • Always active                                                  │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 5: Pattern Detector (pattern)     [ContextAwareLayer]        │
│    • Aggregates behaviors across all users into named patterns:     │
│      - retry_storm (≥30% users retrying)                            │
│      - masked_failure (fail then eventual success)                  │
│      - silent_abandonment (explore but never attempt)               │
│      - early_dropoff (bounce immediately, no attempt)               │
│      - confusion_loop (same event repeated ≥3 times)               │
│    • Cross-instance detection (across multiple flow instances/user) │
│    • Patterns keyed by (flow, contextKey) for granular tracking     │
│    • Always active                                                  │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 6: Baseline Comparator (baseline)                            │
│    • Compares detected patterns against historical snapshots in DB  │
│    • Computes: new pattern, significant increase/decrease, stable   │
│    • Uses configurable window (default 28 days), std dev multiplier │
│    • Stores current observation as new snapshot                     │
│    • Supports "daily" (cached) or "always" (per-request) modes     │
│    • Storage: pattern_snapshots_{project_id} tables in PostgreSQL   │
│    • Always active (degrades gracefully without storage)            │
├─────────────────────────────────────────────────────────────────────┤
│  Layer 7: AI Analyzer (ai)                                          │
│    • Takes patterns + baselines + event evidence → Groq LLM        │
│    • Builds enriched prompt with:                                   │
│      - Pattern metadata, error code distributions                   │
│      - Sample user journeys (actual event sequences)                │
│      - Baseline comparison results                                  │
│    • Strict system prompt: numbers required, no hallucination       │
│    • Diagnoses UX/product gaps and recommends prioritized fixes     │
│    • Returns: summary, details[], hypotheses[], recommendations[],  │
│              confidence_note                                        │
│    • Optional — skipped if disabled or no API key                   │
└─────────────────────────────────────────────────────────────────────┘
  │
  ▼
Structured JSON Response
```

---

## API Surface

### Endpoints

| Method | Path             | Auth                          | Description                    |
|--------|------------------|-------------------------------|--------------------------------|
| GET    | `/health`        | None                          | Health check (DB connectivity) |
| POST   | `/api/v1/analyze` | `X-Infra-Key` (if security enabled) | Analyze events            |

### Required Headers

| Header         | When Required             | Description                                         |
|----------------|---------------------------|-----------------------------------------------------|
| `X-Project-ID` | Always                    | 1–64 alphanumeric/hyphens/underscores. Scopes storage. |
| `X-Infra-Key`  | When `security.enabled`   | Raw API key (server compares SHA-256 hash)           |

### Request Body (`POST /api/v1/analyze`)

```json
{
  "events": [
    {
      "event": "checkout_payment_click",
      "ts": 1707500000000,
      "user_id": "usr-123",
      "session_id": "sess-abc",
      "device": "mobile",
      "error_code": "card_declined"
    }
  ],
  "analysis_context": {
    "scope": "full",
    "project_id": "ignored-overridden-by-header"
  }
}
```

**Required event fields:** `event`, `ts` (epoch ms), `user_id`. All other fields are extra properties auto-classified into roles.

### Response Shape

When AI is enabled:
```json
{
  "success": true,
  "message": "Behavioral analysis complete",
  "request_id": "...",
  "data": {
    "ai_analysis": {
      "summary": "...",
      "details": ["..."],
      "hypotheses": ["..."],
      "recommendations": [
        "[HIGH] Specific, actionable product/UX fix for the highest-severity pattern.",
        "[MEDIUM] Fix for the next pattern, citing data points.",
        "[MONITOR] What to track and when to revisit."
      ],
      "confidence_note": "..."
    }
  }
}
```

When AI is disabled, `data.patterns` is returned instead of `data.ai_analysis`.

---

## Configuration

Config is loaded from: `config.yaml` → `config.yml` → `/etc/velum/config.yaml` (first found wins), then overridden by `VELUM_*` environment variables.

### Key Config Sections

| Section          | Purpose                                                    |
|------------------|------------------------------------------------------------|
| `server`         | Port, host, environment, timeouts                          |
| `storage`        | PostgreSQL connection (host, port, database, user, password, ssl_mode, max_connections), retention_days |
| `security`       | API key auth toggle + SHA-256 hash of key                  |
| `cors`           | Allowed origins, methods, headers                          |
| `resiliency`     | Rate limit (req/s), circuit breaker (failure threshold, reset timeout) |
| `baseline`       | Window days, min days, computation mode, trend thresholds  |
| `ai_analyzer`    | Enable + Groq API key + model for Layer 7                  |
| `vocab_agent`    | Enable + Groq API key + model for Layer 1                  |
| `context_agent`  | Enable + Groq API key + model for Layer 0                  |
| `data_mapping`   | Declarative field mapping for custom event schemas         |

### Environment Variable Overrides

| Variable                       | Overrides                |
|--------------------------------|--------------------------|
| `VELUM_PORT`                   | `server.port`            |
| `VELUM_ENV`                    | `server.environment`     |
| `VELUM_AI_API_KEY`             | `ai_analyzer.api_key`    |
| `VELUM_VOCAB_AGENT_API_KEY`    | `vocab_agent.api_key`    |
| `VELUM_CONTEXT_AGENT_API_KEY`  | `context_agent.api_key`  |

---

## Database Schema

Velum creates tables automatically on first connection.

### Tables

| Table                              | Package        | Purpose                                        |
|------------------------------------|----------------|-------------------------------------------------|
| `vocabulary`                       | vocabagent     | AI-learned word classifications (token → role)  |
| `property_registry`               | propertyagent  | AI-learned property classifications (field → role) |
| `pattern_snapshots_{project_id}`  | storage        | Historical pattern observations for baselines (per-project) |

### Multi-Tenancy

- Each `X-Project-ID` gets its own `pattern_snapshots_{project_id}` table
- Vocabulary and property registry are **shared** across projects
- Background goroutine runs daily cleanup of snapshots older than `retention_days`

---

## Key Interfaces

### `layers.Layer`
```go
type Layer interface {
    Name() string
    Process(input interface{}) (interface{}, error)
}
```
All pipeline layers implement this. Pipeline calls them in registration order.

### `layers.ContextAwareLayer`
```go
type ContextAwareLayer interface {
    Layer
    ProcessWithContext(input interface{}, metadata interface{}) (interface{}, error)
}
```
Behavior analyzer and pattern detector implement this to receive `AnalysisContext`.

### `storage.Storage`
```go
type Storage interface {
    FetchBaselineSnapshots(ctx, projectID, patternType, flow, contextKey, endDate, windowDays) ([]*PatternSnapshot, error)
    StoreSnapshot(ctx, snapshot) error
    Cleanup(ctx) (int64, error)
    Ping(ctx) error
    Close() error
}
```

### `vocabagent.VocabStorage`
Word lookup and storage for the vocabulary system.

### `propertyagent.PropertyStorage`
Property role lookup and storage for the context enricher.

---

## Canonical Types

### `canonical.EventContext`
Attached to every event under `_context` key. Contains:
- `Dimensions` — device, country, platform (built-in list)
- `Targets` — plan_name, product_id (AI-classified)
- `Conditions` — error_code, ab_variant (AI-classified)
- `Measures` — cart_value, load_time_ms (numeric type inference)

### `canonical.PropertyRole`
Enum: `dimension`, `target`, `condition`, `measure`

### Key Pipeline Data Types (by layer)

| Layer Output        | Type                       | Description                                    |
|---------------------|----------------------------|------------------------------------------------|
| Event Adapter       | `[]CanonicalEvent`         | Normalized events with Flow/Action/Status      |
| Session Flow        | `[]FlowInstance`           | Grouped events per user per flow               |
| Behavior Analyzer   | `[]AnalyzedFlow`           | Flows tagged with behavioral signals           |
| Pattern Detector    | `[]DetectedPattern`        | Aggregated anti-patterns across users          |
| Baseline Comparator | `[]ChangeResult`           | Trend analysis vs history                      |
| AI Analyzer         | `AIAnalysis`               | NL summary + hypotheses + recommendations      |

---

## Resiliency Features

- **Circuit Breaker** — wraps all Groq LLM calls. After N consecutive failures, circuit opens and all AI calls short-circuit (graceful degradation) until reset timeout elapses.
- **Rate Limiting** — configurable req/s via `go-chi/httprate`. Returns 429 when exceeded.
- **Request Body Limit** — 10MB max to prevent OOM.
- **Graceful Shutdown** — catches SIGTERM/SIGINT, waits for active requests to finish (configurable timeout).
- **Background Cleanup** — daily goroutine deletes old snapshots per retention policy. Cancellable on shutdown.
- **Health Check** — `/health` endpoint checks DB connectivity, returns "degraded" if unreachable.

---

## Vocabulary System

Event name tokens are classified into:

| Category    | Examples                        | Meaning              |
|-------------|---------------------------------|----------------------|
| **Surface** | checkout, payment, button, cart | The "what" / "where" |
| **Status**  | click, success, failed, retry   | The "state"          |
| **Flow**    | auth, registration, booking     | User intent/journey  |
| **Noise**   | the, a, total, count            | Irrelevant           |

**Dual-lookup:** PostgreSQL vocab table checked first → static vocabulary fallback → "uncategorized" if both miss. Unknown words are learned on the *next* request when vocab agent is enabled.

**Event name parsing** supports: `snake_case`, `camelCase`, `kebab-case`, `dot.notation`.

---

## Detected Anti-Patterns

| Pattern              | Threshold | Description                                               |
|----------------------|-----------|-----------------------------------------------------------|
| `retry_storm`        | ≥30% ratio | Users retrying the same action repeatedly after failures  |
| `masked_failure`     | ≥2 affected flows | Users failing but eventually succeeding (hiding problems) |
| `silent_abandonment` | ≥2 affected users | Users explore but never attempt any action                |
| `early_dropoff`      | ≥40% ratio | Users bounce immediately, no attempt or interaction       |
| `confusion_loop`     | ≥30% ratio | Same event repeated ≥3 times (going in circles)           |

Detection is **cross-instance**: patterns are detected across multiple flow instances per user (e.g., user fails booking, creates new booking → counts as retry across instances).

---

## Data Mapping (Optional)

When `data_mapping.enabled: true`, raw events are transformed before entering the pipeline using declarative path mappings:

```yaml
data_mapping:
  enabled: true
  mapping:
    event:
      paths: ["payload.event.action", "event_name"]
      required: true
    ts:
      paths: ["meta.time", "timestamp"]
      format: "epoch_ms"
      required: true
```

Supports dot-notation paths with fallback order. Extra properties pass through automatically.

---

## Testing

```bash
go test ./...                # All tests
go test ./... -v             # Verbose
go test ./... -cover         # With coverage
```

Test files use both in-memory storage implementations and table-driven tests.

---

## Build & Run

```bash
cp example.config.yaml config.yaml   # Edit with your Postgres + Groq credentials
go mod tidy
go run cmd/velum/main.go              # Starts on :8080
# OR
go build -o velum cmd/velum/main.go   # Build binary
./velum
```

---

## External Dependencies

| Dependency                   | Purpose                   |
|------------------------------|---------------------------|
| `github.com/go-chi/chi/v5`  | HTTP router               |
| `github.com/go-chi/cors`    | CORS middleware            |
| `github.com/go-chi/httprate` | Rate limiting middleware  |
| `github.com/lib/pq`         | PostgreSQL driver          |
| `gopkg.in/yaml.v3`          | YAML config parsing        |

No ORM. All SQL is hand-written in the storage implementations.

---

## Architecture Decisions

| Decision                        | Rationale                                                                 |
|---------------------------------|---------------------------------------------------------------------------|
| Zero-config                     | Property Agent + Vocab Agent eliminate per-customer schema configuration   |
| `internal/` for all packages    | Enforces encapsulation; no external consumers (yet)                       |
| Sequential pipeline             | Layers depend on prior layer output; parallelism not needed at this scale |
| Dual vocab lookup               | Static vocab prevents cold-start; DB vocab learns over time               |
| Surface-fallback folding        | Events without their own flow fold into nearest active flow               |
| Retry cycle merging             | `booking → driver(fail) → booking` = 1 flow instance, not 3              |
| Cross-instance pattern detection | Patterns detected across multiple flow instances per user                |
| Lifecycle event filtering       | `session_end`, `app_closed` don't create ghost flows                      |
| LLM prompt enrichment           | AI receives error distributions + sample journeys, not just pattern names |
| Per-project snapshot tables     | Multi-tenancy isolation without row-level filtering overhead              |
| Circuit breaker on LLM calls    | Prevents cascading failures when Groq is down                             |
| SHA-256 key auth                | Simple, no external auth service needed for self-hosted                   |
