# Velum — Architecture & Pipeline Deep Dive

## What is Velum?

Velum is a **zero-config behavioral pattern detection engine**. You send raw product analytics events (any domain — e-commerce, ride-hailing, streaming, fintech), and Velum:

1. Learns your vocabulary automatically
2. Reconstructs user sessions and flows
3. Tags behavioral signals (retry, abandon, explore, succeed...)
4. Detects anti-patterns (retry storms, masked failures, silent abandonment...)
5. Compares against historical baselines
6. Generates AI-powered natural language analysis

All from a **single POST request** with raw JSON events.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     POST /analyze                           │
│                   (Raw JSON Events)                         │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                  PROCESSING PIPELINE                        │
│                                                             │
│  Layer 0 ──► Context Enricher (AI property role detection)   │
│  Layer 1 ──► Vocab Enricher (AI word classification)        │
│  Layer 2 ──► Event Adapter (Token → Canonical Event)        │
│  Layer 3 ──► Session Flow Reconstructor (Events → Flows)    │
│  Layer 4 ──► Behavior Analyzer (Flows → Behavioral Tags)    │
│  Layer 5 ──► Pattern Detector (Behaviors → Anti-Patterns)   │
│  Layer 6 ──► Baseline Comparator (Patterns → Trends)        │
│  Layer 7 ──► AI Analyzer (Trends → Natural Language)        │
│                                                             │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    JSON Response                            │
│  { patterns, baselines, ai_analysis, flows, behaviors }     │
└─────────────────────────────────────────────────────────────┘
```

---

## Pipeline — Step by Step

### Layer 0: Context Enricher (Property Agent)

**File:** `internal/layers/propertyagent/enricher.go`

**Purpose:** Identifies the **role** of each property/field in your event JSON — which field is the user ID, which is the timestamp, which is the event name, etc.

**Why it exists:** Different products send events in completely different schemas:

```json
// Product A
{"event_name": "checkout", "uid": "u1", "time": 170850000, "amount": 320}

// Product B  
{"action": "purchase", "user": "u1", "timestamp": "2024-02-21T10:00:00Z", "cart_value": 320}

// Product C
{"type": "payment_failed", "customer_id": "u1", "ts": 170850000, "error": "bank_timeout"}
```

Velum doesn't know which field is what. The Property Agent figures it out.

**How it works:**
1. Takes a sample of events from the batch
2. Sends them to **Groq LLM** with a classification prompt
3. The LLM returns a mapping:
   ```json
   {
     "event_name_field": "event",
     "user_id_field": "user_id",
     "timestamp_field": "ts",
     "session_id_field": "session_id",
     "error_field": "error_code",
     "amount_field": "cart_value"
   }
   ```
4. This mapping is cached and used by Layer 2 to extract fields correctly

**Example:**
```
Input sample:
  {"id":"e01", "event":"checkout_initiated", "ts":170850000, "user_id":"u1", "session_id":"s1", "cart_value":320}

Property Agent output:
  event_name  → "event"
  timestamp   → "ts"
  user_id     → "user_id"
  session_id  → "session_id"

Now Layer 2 knows: evt["event"] is the event name, evt["ts"] is the timestamp, etc.
```

**Key point:** Without this, Velum would need a config file per customer specifying their schema. The Property Agent makes Velum truly **zero-config**.

---

### Layer 1: Vocab Enricher

**File:** `internal/layers/vocabagent/enricher.go`

**Purpose:** Learns the meaning of unknown words in your event names.

**How it works:**
1. Tokenizes every event name (e.g., `payment_failed` → `["payment", "failed"]`)
2. Checks each token against PostgreSQL vocab storage
3. Unknown tokens are batched and sent to **Groq LLM** for classification
4. The LLM classifies each word into a role:
   - **Surface** — the "what" (payment, checkout, ride, playback)
   - **Status** — the "state" (failed, success, initiated, completed)
   - **Flow** — higher-level grouping (cart, order, booking)
   - **Noise** — irrelevant words (the, a, total, count)
5. Classifications are stored back to PostgreSQL for future requests

**Example:**
```
Input events: ["booking_requested", "driver_cancelled", "ride_estimated"]

Unknown tokens: ["booking", "requested", "driver", "cancelled", "ride", "estimated"]

LLM classifies:
  booking   → Surface
  requested → Status (normalized: "request")
  driver    → Surface
  cancelled → Status (normalized: "cancelled")
  ride      → Surface
  estimated → Status (normalized: "estimated")

Stored to PostgreSQL → available for all future requests
```

**Key point:** Words learned here are immediately available to Layer 2 (Event Adapter) on the **same request** via dual-lookup (PostgreSQL first, static vocabulary fallback).

---

### Layer 2: Event Adapter

**File:** `internal/layers/eventadapter/adapter.go`

**Purpose:** Transforms raw JSON events into **canonical events** — Velum's internal standardized format.

**How it works:**
1. Uses the Property Agent's field mapping to extract: event name, user_id, session_id, timestamp
2. Tokenizes the event name: `payment_failed` → `["payment", "failed"]`
3. Classifies each token using **dual-lookup**:
   - **PostgreSQL first** (AI-learned vocab from Layer 0)
   - **Static vocabulary fallback** (hardcoded common words)
4. Builds a canonical event:
   ```go
   CanonicalEvent{
     Flow:       "payment",     // from Surface token
     Action:     "payment",     // from Surface token  
     Status:     "failed",      // from Status token
     UserID:     "u1",
     SessionID:  "s1",
     Timestamp:  1708500015000,
     RawProperties: { ... }     // original JSON preserved
   }
   ```

**The dual-lookup guarantee:**
```
Token "playback" arrives
  → Check PostgreSQL: found! classified as Surface ✅
  → Done

Token "xyzwidget" arrives (company-specific)
  → Check PostgreSQL: not found ❌
  → Check static vocabulary: not found ❌
  → Classified as "uncategorized"
  → Next request: Vocab Enricher (Layer 1) will learn it via LLM
```

---

### Layer 3: Session Flow Reconstructor

**File:** `internal/layers/sessionflow/reconstructor.go`

**Purpose:** Groups canonical events into **user sessions** and **flow instances**.

**How it works:**

**Step 1: Group by user**
```
u1: [app_opened, destination_entered, ride_estimated, booking_requested, ...]
u2: [app_opened, destination_entered, ride_estimated, session_end]
```

**Step 2: Split into sessions** (30-min gap threshold)
```
u1 session 1: [all events within 30 min of each other]
u1 session 2: [events after 30 min gap]  (if any)
```

**Step 3: Build flow instances**

Events are grouped by their **flow name** (derived from the Surface token). Key rules:

| Rule | Example |
|------|---------|
| Contiguous events with same flow → same instance | `playback_started, playback_error, playback_retry` → 1 playback instance |
| Entry status starts a new instance | `booking_requested` (status: request=entry) → new booking instance |
| System lifecycle events are skipped | `session_end`, `app_closed` → no flow created |
| Supply failure events fold into parent | `no_drivers_available` → folds into active booking flow |
| Surface-fallback folding | `driver_assigned` (no own flow) → folds into active booking flow |

**Retry cycle detection:**
```
booking_requested → driver_assigned → driver_cancelled → booking_requested

Flow A (booking) → Flow B (driver, cancelled) → Flow A again

Since B ended in failure → merge second booking back into first booking instance
Result: 1 booking flow with retry evidence
```

**Output:**
```
FlowInstance{
  FlowName:  "booking",
  UserID:    "u1",
  SessionID: "s1",
  Events:    [booking_requested, driver_assigned, driver_cancelled, booking_requested, driver_assigned, ride_started, ride_completed],
  StartTime: 1708700045000,
  EndTime:   1708701000000,
}
```

---

### Layer 4: Behavior Analyzer

**File:** `internal/layers/behavior/analyzer.go`

**Purpose:** Tags each flow instance with **behavioral signals** — what the user was doing.

**Behavior taxonomy:**

| Behavior | Meaning | Detection Rule |
|----------|---------|----------------|
| `explore` | User looked around | Entry events, view events |
| `attempt` | User tried to do something | Action events (submit, request, pay) |
| `succeed` | User completed the goal | Success/complete status |
| `retry` | User tried again after failure | Error → same action repeated |
| `abandon` | User left without completing | No success + session ends or flow changes |
| `hesitate` | User paused before acting | Long delay between events |
| `bypass` | User skipped expected steps | Jumped past expected flow stages |
| `progress` | User moved to next funnel step | Flow transition to deeper step |

**Example:**
```
Flow: booking (u1)
Events: booking_requested → driver_assigned → driver_cancelled → booking_requested → driver_assigned → ride_started

Behaviors: [attempt, retry, progress]
  - attempt:  booking_requested
  - retry:    second booking_requested after failure
  - progress: reached ride_started (deeper funnel step)
```

**Flow Intent classification:**
```
FlowIntent: "transact"   — user intended to complete a transaction
FlowIntent: "browse"     — user was just looking around  
FlowIntent: "unknown"    — can't determine
```

---

### Layer 5: Pattern Detector

**File:** `internal/layers/pattern/detector.go`

**Purpose:** Detects **anti-patterns** across all users — systemic problems, not individual quirks.

**How it works:**
1. Groups all flow instances by `(flowName, contextKey)` — e.g., all "booking" flows, or all "booking" flows from "mobile/IN"
2. Runs 5 detection algorithms across each group
3. Each detector calculates a **ratio** against configurable thresholds

**Pattern definitions:**

#### 1. Retry Storm
```
Question: Are too many users retrying?
Detection: Count users with error→action sequences OR multiple flow instances with failures
Threshold: ≥ 30% of users retrying
```

#### 2. Masked Failure
```
Question: Are users failing but eventually succeeding? (hiding real problems)
Detection: Users who have both failed AND succeeded instances of the same flow
Threshold: ≥ 2 affected flows (absolute count)
```

#### 3. Silent Abandonment
```
Question: Are users leaving without even trying?
Detection: Users who explored but never attempted any action
Threshold: ≥ 2 affected users (absolute count)
```

#### 4. Early Dropoff
```
Question: Are users bouncing immediately?  
Detection: Flows with only explore/abandon behaviors, no attempt/retry/succeed
Threshold: ≥ 40% of eligible flows dropping off early
Excludes: Single-event flows, browse-intent flows, flows where user progressed
```

#### 5. Confusion Loop
```
Question: Are users going in circles?
Detection: Same event type repeated ≥ 3 times within a flow (e.g., searching 3 times)
Threshold: ≥ 30% of users showing loops
```

**Cross-instance detection** (critical for accuracy):
```
u1: booking instance 1 (fail) → booking instance 2 (success)
    → retry_storm: YES (had failed instance before succeeding)
    → masked_failure: YES (failed then eventually succeeded)

u3: booking instance 1 (fail) → booking instance 2 (fail) → booking instance 3 (fail)  
    → retry_storm: YES
    → masked_failure: NO (never succeeded)
    → silent_abandonment: NO (they attempted, they just failed)
```

**Output:**
```json
{
  "pattern": "retry_storm",
  "flow": "booking",
  "affected_users": 3,
  "total_flows": 4,
  "impact_ratio": 0.75,
  "sample_flow_ids": ["flow_1708700045_3", "flow_1708700015_2"],
  "description": "High frequency of retry attempts in booking flow"
}
```

---

### Layer 6: Baseline Comparator

**File:** `internal/layers/baseline/detector.go`

**Purpose:** Compares current patterns against **historical baselines** stored in PostgreSQL.

**How it works:**
1. For each detected pattern, queries PostgreSQL for historical occurrences of the same `(pattern, flow, context)` combination
2. Computes:
   - **Is this new?** First time this pattern was seen
   - **Is this worse?** Impact ratio increased beyond `std_deviation_multiplier × σ`
   - **Is this better?** Impact ratio decreased significantly
   - **Is this stable?** Within normal range
3. Stores the current observation as the new baseline data point

**Example:**
```
Current:  retry_storm on booking, impact_ratio = 0.75
Baseline: last 7 observations averaged 0.35 with σ = 0.08

Deviation: (0.75 - 0.35) / 0.08 = 5.0 standard deviations
Threshold: 2.0 (from config)

Result: SIGNIFICANT INCREASE — "retry_storm on booking has increased significantly"
```

**First observation:**
```
Current:  retry_storm on booking, impact_ratio = 0.75
Baseline: no prior data

Result: NEW PATTERN — "retry_storm on booking observed for the first time, establishing baseline"
```

---

### Layer 7: AI Analyzer

**File:** `internal/layers/ai/groq.go`

**Purpose:** Generates **natural language analysis** of detected patterns using Groq LLM.

**How it works:**
1. Collects all detected patterns + baseline comparisons
2. Builds an enriched prompt with:
   - Pattern metadata (name, flow, affected users, ratios)
   - **Event-level evidence** (error code distributions, status distributions)
   - **Sample user journeys** (event sequences showing the actual user path)
   - Baseline comparison results
3. Sends to Groq (llama-3.1-8b-instant) with a strict system prompt
4. System prompt enforces:
   - Numbers and percentages in every detail
   - Error codes must be cited
   - Hypotheses must reference specific data points
   - No generic filler like "requires further investigation"

**Prompt structure:**
```
SYSTEM: You are a behavioral analytics expert. 
Every detail MUST include: affected_users/total, error codes with counts...

USER: 
## Detected Patterns
Pattern 1: retry_storm in playback flow
- Affected: 2, Total: 12, Impact: 0.42
- Error distribution: buffer_timeout=2, drm_license_failed=3
- Sample journey u1: playback_started(start) → playback_error(error, buffer_timeout) → playback_started(attempt) → playback_error(error, buffer_timeout) → playback_started(attempt) → playback_completed(success)

## Baseline Comparison
- retry_storm in playback: FIRST OBSERVATION (no prior baseline)
```

**Output:**
```json
{
  "summary": "A retry storm affecting 42% of playback users...",
  "details": ["retry_storm in playback: 2 affected users out of 12 total. Error codes: buffer_timeout (2), drm_license_failed (3)..."],
  "hypotheses": ["DRM licensing may be broken for IN region — 3 of 5 failures are drm_license_failed on mobile"],
  "confidence_note": "These are hypotheses based on observed behavioral changes."
}
```

---

## Full Example: Ride-Hailing App

### Input (6 users, 55 events)

```json
{
  "events": [
    {"id":"r01","event":"app_opened","ts":1708700000000,"user_id":"u1","session_id":"s1","device":"mobile","country":"IN","city":"bangalore"},
    {"id":"r06","event":"booking_requested","ts":1708700045000,"user_id":"u1","session_id":"s1","device":"mobile","country":"IN","fare":790},
    {"id":"r07","event":"driver_assigned","ts":1708700060000,"user_id":"u1","session_id":"s1","device":"mobile","country":"IN","driver_id":"d01"},
    {"id":"r08","event":"driver_cancelled","ts":1708700090000,"user_id":"u1","session_id":"s1","device":"mobile","country":"IN","cancel_reason":"too_far"},
    {"id":"r09","event":"booking_requested","ts":1708700095000,"user_id":"u1","session_id":"s1","device":"mobile","country":"IN","fare":790},
    {"id":"r10","event":"driver_assigned","ts":1708700110000,"user_id":"u1","session_id":"s1","device":"mobile","country":"IN","driver_id":"d02"},
    {"id":"r11","event":"ride_started","ts":1708700180000,"user_id":"u1","session_id":"s1","device":"mobile","country":"IN"},
    {"id":"r12","event":"ride_completed","ts":1708701000000,"user_id":"u1","session_id":"s1","device":"mobile","country":"IN"}
  ]
}
```

### What happens at each layer

```
Layer 0 (Context Enricher / Property Agent):
  Schema detected: event_name="event", timestamp="ts", user_id="user_id", session_id="session_id"
  
Layer 1 (Vocab Enricher):
  Tokens: [app, opened, booking, requested, driver, assigned, cancelled, ride, started, completed]
  Unknown: none (all in static vocab or previously learned)

Layer 2 (Event Adapter):
  "booking_requested" → CanonicalEvent{Flow:"booking", Status:"request", Action:"booking"}
  "driver_assigned"   → CanonicalEvent{Flow:"driver", Status:"assigned", Action:"driver"}
  "driver_cancelled"  → CanonicalEvent{Flow:"driver", Status:"cancelled", Action:"driver"}
  "ride_completed"    → CanonicalEvent{Flow:"ride", Status:"completed", Action:"ride"}

Layer 3 (Session Flow Reconstructor):
  u1 session s1:
    Flow 1: booking [booking_requested, driver_assigned, driver_cancelled, booking_requested, driver_assigned]
            ↑ driver events folded in (surface-fallback)
            ↑ second booking merged back (retry cycle: driver was cancelled)
    Flow 2: ride [ride_started, ride_completed]
            ↑ new flow (entry status: "started")

Layer 4 (Behavior Analyzer):
  Flow 1 (booking): [attempt, retry, progress]
    attempt  → booking_requested
    retry    → second booking_requested after failure
    progress → eventually reached ride flow
  
  Flow 2 (ride): [attempt, succeed]
    attempt  → ride_started
    succeed  → ride_completed

Layer 5 (Pattern Detector):
  Across all users:
  - retry_storm on booking: 3/4 users retried (75%) → DETECTED
  - masked_failure on booking: 2/4 users failed then succeeded → DETECTED

Layer 6 (Baseline Comparator):
  - retry_storm on booking: FIRST OBSERVATION → establishing baseline
  - masked_failure on booking: FIRST OBSERVATION → establishing baseline

Layer 7 (AI Analyzer):
  "retry_storm in booking flow: 3 affected users out of 4 total (75%).
   Error: driver_cancelled with cancel_reason=too_far occurring in 2 of 3 cases.
   Hypothesis: Driver supply may be insufficient for long-distance rides in Bangalore."
```

### Output

```json
{
  "data": {
    "patterns": [
      {
        "pattern": "retry_storm",
        "flow": "booking",
        "affected_users": 3,
        "total_flows": 4,
        "impact_ratio": 0.75
      },
      {
        "pattern": "masked_failure",
        "flow": "booking",
        "affected_users": 2,
        "total_flows": 4,
        "impact_ratio": 0.50
      }
    ],
    "ai_analysis": {
      "enabled": true,
      "summary": "Booking flow shows 75% retry storm rate due to driver cancellations...",
      "details": ["..."],
      "hypotheses": ["Driver supply insufficient for long-distance Bangalore rides..."]
    }
  }
}
```

---

## Data Flow Diagram

```
Raw Events (any schema, any domain)
    │
    ▼
┌──────────────┐     ┌──────────────┐
│ Context      │────►│  PostgreSQL   │ (property registry)
│ Enricher     │     │  prop table   │
│ (Groq LLM)  │     └──────────────┘
└──────┬───────┘
       │ field mapping
       ▼
┌──────────────┐     ┌──────────────┐
│ Vocab Agent  │────►│  PostgreSQL   │ (learned vocabulary)
│ (Groq LLM)  │     │  vocab table  │
└──────────────┘     └──────┬───────┘
                            │
                            │ dual-lookup
                            ▼
┌──────────────────────────────┐
│      Event Adapter           │
│  raw JSON → Canonical Event  │
│  (uses vocab + field map)    │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│  Session Flow Reconstructor  │
│  Canonical Events → Flows    │
│  (session splits, retry      │
│   cycle merging, lifecycle   │
│   event filtering)           │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│    Behavior Analyzer         │
│  Flows → Behavioral Tags     │
│  (explore, attempt, retry,   │
│   abandon, succeed, fail)    │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│    Pattern Detector          │
│  Behaviors → Anti-Patterns   │
│  (retry_storm, masked_fail,  │
│   silent_abandon, dropoff,   │
│   confusion_loop)            │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐     ┌──────────────┐
│   Baseline Comparator        │────►│  PostgreSQL   │ (historical patterns)
│  Patterns → Trend Analysis   │     │  baseline tbl │
└──────────────┬───────────────┘     └──────────────┘
               │
               ▼
┌──────────────────────────────┐
│      AI Analyzer             │
│  Patterns + Baselines +      │
│  Event Evidence → Natural    │
│  Language Analysis           │
│  (Groq LLM, structured)     │
└──────────────┬───────────────┘
               │
               ▼
          JSON Response
```

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Language | Go 1.23+ |
| HTTP Router | [chi/v5](https://github.com/go-chi/chi) |
| LLM Provider | Groq (llama-3.1-8b-instant) |
| Database | PostgreSQL (vocab + baselines) |
| Rate Limiting | [go-chi/httprate](https://github.com/go-chi/httprate) |
| Config | YAML (config.yaml) |

---

## Key Design Decisions

| Decision | Why |
|----------|-----|
| **Zero-config** | Property Agent + Vocab Agent eliminate schema configuration |
| **Dual vocab lookup** | Static vocab prevents cold-start "unknown" flows; DB vocab learns over time |
| **Surface-fallback folding** | Events without their own flow fold into the nearest active flow (driver_assigned → booking) |
| **Retry cycle merging** | booking → driver(fail) → booking creates 1 flow instance, not 3 |
| **Cross-instance detection** | Patterns detected across multiple flow instances per user, not just within |
| **Lifecycle event filtering** | session_end, app_closed don't create ghost flows |
| **LLM prompt enrichment** | AI receives error distributions + sample journeys, not just pattern names |