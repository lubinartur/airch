# Technical Architecture

## Principles

**Local first.**
All personal data lives on the user's device. Nothing leaves without explicit permission. This is not a constraint — it is the product's core trust proposition.

**Minimum viable intelligence.**
The system should use the simplest approach that achieves the right output. Not the most impressive model. The most appropriate one.

**Silence is cheap. Noise is expensive.**
Architectural decisions that reduce false positives are worth more than those that increase recall.

---

## Stack

**Frontend**
- React + TypeScript
- Tailwind CSS
- Vite

**Backend**
- FastAPI (Python)
- SQLite — primary data store
- ChromaDB — vector embeddings for semantic memory search

**AI Layer**
- Claude (Sonnet) — primary reasoning, Dialogue, Morning Brief, complex analysis
- Claude (Haiku) — fast extraction, classification, background tasks
- Local LLM (optional) — full-local mode for users who want zero cloud dependency
- Embeddings — for semantic search across memory

---

## Agent Architecture

AIRCH uses specialized agents with shared memory.

For the user, there is always one AIRCH. The agent routing is invisible.

```
MasterAgent      — Today, Dialogue, cross-domain reasoning
FinanceAgent     — Finance Space
HealthAgent      — Health Space
ProjectsAgent    — Work/Startup Space
LifeAgent        — Personal Spaces, Goals, Direction
```

**Routing logic:**
- Page context determines agent
- MasterAgent active on Today and full Dialogue
- Specialized agents active within their Spaces
- MasterAgent sees everything; specialized agents see only their domain

---

## Data Model (overview)

```
user_profile         — single row, core identity
user_facts           — permanent beliefs about the user
events               — timestamped occurrences across domains
memories             — processed and confidence-weighted facts
patterns             — recurring structures across events
observations         — short-lived signals (expire in ~7 days)
threads              — cross-domain story links
decisions            — choices made, with follow-up tracking
dialogue_sessions    — conversation history
spaces               — domain rooms with metadata
continuity_entries   — long-term life arc data points
direction_items      — directional goals and values
```

Full schema: see DATABASE_SCHEMA.md

---

## Memory Pipeline

```
Input (chat / data / behavior)
↓
Extraction (Haiku — fast, async)
↓
Deduplication + confidence scoring
↓
Storage (SQLite)
↓
Embedding (ChromaDB)
↓
Available to Reasoning Engine
```

Extraction runs fire-and-forget after every interaction. It does not block the response.

---

## Context Management

Every agent request receives a context package built by the Context Manager.

**Token budget:** strict limit per request.

**Priority:**
1. User profile (always)
2. Confirmed facts (always)
3. Semantic matches (query-relevant memories)
4. Domain-specific data (by page/agent)
5. Recent events (last 5, importance ≥ medium)
6. Open observations (only if unread and recent)

Context is assembled fresh per request. No stale injection.

---

## Reasoning Engine Pipeline

```
1. Collect signals from all domains
2. Score by priority (safety → direction → commitments → momentum)
3. Gate check — is this worth surfacing?
4. If yes: generate output (Brief / Recommendation / Question / Warning)
5. If no: silence
6. Track user response → update Memory confidence
```

Gate check runs on Haiku before generating with Sonnet. Cheap filter before expensive generation.

---

## Privacy Modes

**Full Local (default)**
- Ollama for all LLM calls
- Zero external requests
- Complete data sovereignty

**Smart Mode**
- Claude API for complex reasoning
- Anonymization layer strips PII before any cloud call
- User sees preview of what is being sent
- Every cloud call is logged locally

---

## Performance Targets

- Today screen load: < 1s
- Dialogue response start (streaming): < 500ms
- Morning Brief generation: async, ready before user opens app
- Background extraction: non-blocking, max 3s after message

---

## Known Debt (current)

- Two LLM clients (sync SDK + async httpx) — merge into one
- Extraction pipeline: 4–5 sequential LLM calls after each message — consolidate into one
- No authentication — acceptable locally, needs solution before any network exposure
- Memory lifecycle (archival, summarization) — not yet implemented
- Vector search (ChromaDB) — schema defined, runtime integration pending
