# Context Builder — Operational Spec

> What goes into the LLM — and what stays out.

---

## Role

The Context Builder assembles the context package that is passed to every LLM call.

Its job is not to include as much as possible. Its job is to include exactly what is needed — and nothing else.

The enemy of good context is completeness. A context package that tries to include everything includes nothing useful.

---

## Core Principle

**Minimum sufficient context.**

Not maximum available context. Minimum sufficient.

Every token that does not contribute to a better response is a token that dilutes the ones that do.

---

## Hard Token Budget

```
Total context budget:       4,000 tokens (adjustable by model)
  ├── Core layer:           ~400 tokens (always included, non-negotiable)
  ├── Retrieved memory:     ~600 tokens (semantic match, top results)
  ├── Domain layer:         ~800 tokens (relevant to this request)
  ├── Recency layer:        ~400 tokens (recent events, open loops)
  └── Conversation history: ~1,800 tokens (last N turns of Dialogue)

System prompt:              separate, not counted against context budget
```

If assembled context exceeds budget: truncate by priority, lowest priority first. Never truncate the core layer.

---

## Context Layers

### Layer 1 — Core (always included)

```
user_profile:
  name, city, profession
  monthly income (rounded to nearest 100)
  primary goals (top 3 from Direction)
  key context (free text, max 150 tokens)

confirmed_facts:
  all Facts with confidence ≥ 0.70
  max 20 items; if more exist, prioritize by recency + relevance
  format: plain statements, one per line

direction:
  current Direction interpretation (2–3 sentences)
  top 2 active Values
```

Total core layer: ~400 tokens.

The core layer is never compressed. If it exceeds 400 tokens, confirmed_facts are trimmed first (lowest confidence removed).

---

### Layer 2 — Retrieved Memory (semantic match)

Results of Memory Retrieval for this specific query.

See `12_MemoryRetrieval_Operational_Spec.md` for retrieval logic.

Format in context:
```
[RELEVANT MEMORY]
- [fact / pattern / event] — [confidence framing if < 0.85]
- ...
max 5 items, most relevant first
~120 tokens per item max
```

If retrieval returns nothing relevant: layer is omitted. Do not pad with low-relevance results.

---

### Layer 3 — Domain Layer (request-specific)

Included based on input classification and current surface.

```
If financial context:
  spending summary (current cycle):
    total income, total debit, category breakdown (top 5)
    free capital, days until next salary
    anomalies flagged by Brain
  recent transactions: last 10 debits, not internal transfers
  active subscriptions: name + amount + billing day
  active obligations: name + amount + due date

If health context:
  workouts: last 10 with type, duration, date
  body metrics: last 3 entries
  biomarker summary: last checkup, flagged markers only
  energy pattern: AIRCH's current read (1 sentence)

If projects context:
  active projects: name, status, last activity date, momentum signal
  recent logs: last 5 entries across all active projects
  stall signals: projects inactive > 7 days

If life / personal context:
  recent events: last 10 events, domain=life or personal
  active goals: name, progress indicator
  open dilemmas: title + status

If today / overview context:
  summary across all domains: one line per domain with current signal
  active threads: name + status
  open loops: top 3 by priority
```

Only the relevant domain layer is included. Not all domains simultaneously unless the surface is Today/Overview (where a summary of all domains is appropriate).

---

### Layer 4 — Recency Layer

Always included. Captures what has been happening recently regardless of domain.

```
recent_events:
  last 5 events with importance ≥ medium
  any domain
  format: date + domain + one-sentence description

open_loops:
  top 3 open loops by priority
  format: topic + how long open + last mention

current_observation:
  if Reasoning Engine has a pending Belief relevant to this request:
  include it (framed as hypothesis, not fact)
  max 1 item
```

Total recency layer: ~400 tokens.

---

### Layer 5 — Conversation History

For Dialogue: last N turns of the conversation.

```
Budget allocation: up to 1,800 tokens
Turn format: [role]: [content]

Truncation rules:
  - Always keep last 4 turns (minimum continuity)
  - Fill remaining budget with turns in reverse chronological order
  - Never truncate mid-thought (cut at turn boundary)
  - If a turn references an open loop, prefer to keep it
```

For Today generation or Space updates: no conversation history in context (not a conversational request).

---

## Context Assembly Algorithm

```
1. Start with Layer 1 (Core). Always.

2. Run Memory Retrieval for this query.
   Add Layer 2 results. If nothing relevant: skip.

3. Classify request domain.
   Add relevant Layer 3 block. If multiple domains: Today/Overview mode.

4. Add Layer 4 (Recency). Always.

5. Add Layer 5 (Conversation History) if this is a Dialogue call.

6. Check total token count.
   If over budget:
     a. Trim Layer 5 first (older turns)
     b. Trim Layer 3 next (less critical domain data)
     c. Trim Layer 2 next (lowest relevance results)
     d. Never trim Layer 1 or Layer 4 below their minimum

7. Add system prompt (separate from context budget).

8. Package and pass to LLM.
```

---

## What Never Goes Into Context

- Raw transaction lists beyond the last 10 (use category summaries instead)
- Full workout history (use recency + anomalies only)
- Full chat history beyond budget (truncate from oldest)
- Low-confidence memory (< 0.3) — never injected
- Sensitive memory unless user raised the topic in this session
- Internal system state (cooldown registry, Stillness verdicts, confidence scores)
- Embeddings or vector IDs
- Technical metadata (database IDs, timestamps in raw format)
- Other users' data (obvious, but stated explicitly)

---

## Context Framing

The context package is not injected as a raw data dump. It is formatted to be legible to the LLM as natural language.

### Principles
- No JSON blobs in the main context (use natural language)
- Numbers are rounded and contextualized: not "€338.42" but "~€340"
- Dates are relative when recent: "3 days ago", "last Tuesday"
- Uncertainty is expressed in language: "appears to be", "based on recent data"
- Domain summaries come before details
- The most important information appears earliest

### Example context excerpt (Finance domain)

```
[CORE]
User: Artur, 32, Tallinn. Building a startup (AIRCH). Monthly income ~€3,800.
Direction: Becoming someone who builds independently. Values autonomy over income.
Key facts: Does not want external investment. Works best late at night. Avoids financial review when stressed.

[RELEVANT MEMORY]
- Restaurant spending rises during high-stress project periods (pattern, confirmed twice)
- Last financial review was 23 days ago

[FINANCE]
Current cycle (May 10 – Jun 9): Income €3,800. Spent €2,940. Free capital: €860.
Categories: Restaurants €420 (↑89% vs last cycle), Subscriptions €205, Transport €180, Other €2,135.
Flagged: Restaurant spike coincides with AIRCH project stall (11 days inactive).
Upcoming: Telia €29.90 on Jun 15. Coop loan €320 on Jun 10.

[RECENT]
3 days ago: Uploaded Swedbank statement (May cycle)
5 days ago: Noted in Dialogue that AIRCH project feels stuck
8 days ago: Archived SkipMar project

[OPEN LOOPS]
- What to do about the investment question (open 18 days)
```

This is readable to a human and to an LLM. It gives the LLM what it needs to respond well — without overwhelming it with raw data.

---

## Context for Different Call Types

### Dialogue response call
All 5 layers. Conversation history prioritized.

### Morning Brief generation
Layers 1, 3 (all domains summary), 4. No conversation history.

### Today assembly
Layers 1, 3 (all domains summary), 4. No conversation history.

### Space update (AIRCH's Read)
Layers 1, 3 (domain-specific only), 4 (domain-filtered). No conversation history.

### Memory extraction (post-processing)
Layer 5 only (the conversation being processed). No other context needed.

### Stillness fast check
Layer 1 only + the candidate output. Haiku-level call.

### Pattern detection (nightly)
Layers 1, 2. No domain layer. No conversation history.
