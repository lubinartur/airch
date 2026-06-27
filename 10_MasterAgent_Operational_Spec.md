# Master Agent — Operational Spec

> The orchestrator. Connects Brain, Memory, Reasoning, Stillness, Today, Dialogue, and Spaces into one system.

---

## Role

The Master Agent is the central nervous system of AIRCH.

It does not think. It does not generate content. It does not hold beliefs.

It orchestrates: it receives signals, routes them to the right subsystem, collects outputs, applies Stillness, and decides what reaches the user.

Everything the user experiences passes through the Master Agent.

---

## Architecture Overview

```
External Input
  (user message / domain data / scheduled job / app open)
        ↓
  Master Agent
        ↓
  ┌─────────────────────────────────────────┐
  │  1. Input Classification                │
  │  2. Context Builder                     │
  │  3. Route to Subsystem(s)               │
  │     ├── Brain (belief formation)        │
  │     ├── Memory (retrieval + storage)    │
  │     ├── Reasoning Engine (priority)     │
  │     ├── Stillness Check (gate)          │
  │     └── Domain Agents (Finance / Health │
  │         / Projects / Life)              │
  │  4. Collect outputs                     │
  │  5. Resolve conflicts                   │
  │  6. Route to surface                    │
  │     ├── Today                           │
  │     ├── Dialogue                        │
  │     └── Spaces                          │
  │  7. Trigger post-processing             │
  └─────────────────────────────────────────┘
```

---

## Input Types

The Master Agent handles four input types:

### 1. User Message
Source: Dialogue or Space-level chat.
Trigger: immediate.
Pipeline: classify → build context → route to Domain Agent or MasterAgent LLM → generate response → extract memory → update state.

### 2. Domain Data Ingestion
Source: finance upload, health sync, calendar update.
Trigger: on data arrival.
Pipeline: parse → extract signals → route to Brain → update Memory → run Reasoning Engine → evaluate Stillness → surface if warranted.

### 3. Scheduled Job
Source: internal scheduler (morning run, nightly run, post-session run).
Trigger: time-based.
Pipeline: defined per job type (see 13_Scheduler_Operational_Spec.md).

### 4. App Open
Source: user opens AIRCH.
Trigger: on open, after ≥ 4 hours since last open.
Pipeline: check Today cache → if stale, rebuild → check for proactive Dialogue opening → surface.

---

## Step 1 — Input Classification

Every input is classified before routing.

```
Input type:
  user_message
  domain_data
  scheduled_job
  app_open

If user_message:
  message_type:
    question          — user asks something specific
    statement         — user shares information
    decision          — user records a choice
    emotional         — user expresses a feeling or state
    command           — user asks AIRCH to do something
    continuation      — user continues a previous thread

  domain_hint:        which Space/domain is this primarily about?
  urgency:            does this require immediate response?
  has_open_loop:      does this relate to an existing open loop?
```

Classification uses a fast LLM call (Haiku-level). Output feeds routing logic.

---

## Step 2 — Context Builder

Before any LLM call, the Master Agent builds a context package.

See `11_ContextBuilder_Operational_Spec.md` for full detail.

Summary:
- Always included: user profile, confirmed facts, Direction
- Query-matched: semantic memory retrieval (top 3–5 relevant memories)
- Domain-specific: data from the relevant Space
- Recency layer: last 5 significant events, open loops
- Hard token budget: never exceeded

The Context Builder runs before every LLM call. The Master Agent does not call LLMs without a context package.

---

## Step 3 — Routing

Based on input classification, the Master Agent routes to one or more subsystems.

### Routing table

```
Input type          Primary route              Secondary route
───────────────────────────────────────────────────────────────
user_message        Domain Agent or Master LLM  Memory extraction (async)
  + question        → Agent LLM call
  + statement       → Memory extraction first, then response
  + decision        → Decision tracker + Memory + response
  + emotional       → Master LLM (full context)
  + command         → Parse command → execute → confirm

domain_data         Brain                       Reasoning Engine (after Brain)
  + finance         → Finance signal processor
  + health          → Health signal processor
  + calendar        → Calendar signal processor

scheduled_job       Scheduler                   see Scheduler spec
  + morning_run     → Today builder + Reasoning Engine
  + nightly_run     → Memory consolidation + Pattern detection
  + post_session    → Extraction pipeline

app_open            Today cache check           Proactive Dialogue check
  + today stale     → Today builder
  + today fresh     → display cached Today
  + dialogue check  → evaluate proactive opening
```

### Domain Agent routing

When the user is in a specific Space, the Master Agent routes to the Domain Agent for that Space:

```
Finance Space    → FinanceAgent
Health Space     → HealthAgent
Projects Space   → ProjectsAgent
Personal Spaces  → LifeAgent
Today            → MasterAgent LLM (full context)
Dialogue         → MasterAgent LLM (full context) or Domain Agent if Space-scoped
```

Domain Agents use the same LLM but different system prompts and context packages. From the user's perspective: one AIRCH, different rooms.

---

## Step 4 — Collect Outputs

After routing, the Master Agent collects:

- LLM response text (if applicable)
- Brain output: new Beliefs, confidence updates
- Memory updates: new Facts, Pattern changes, open loops
- Reasoning Engine output: surfacing decision + output type
- Stillness verdict: pass or withhold

All outputs arrive before the Master Agent decides what to surface.

---

## Step 5 — Conflict Resolution

Multiple subsystems may produce competing outputs. The Master Agent resolves conflicts.

### Rules

**Stillness overrides everything.**
If Stillness says withhold — nothing surfaces. Even a high-confidence Belief.

**Warnings override Today content.**
A Warning replaces the Morning Brief if one fires.

**User-triggered responses are never withheld by Stillness.**
If the user asked something, they get an answer. Stillness applies only to proactive outputs.

**Higher confidence wins between competing signals.**
If two Beliefs compete for Today's Primary Thought: higher confidence wins. If equal: more connected to active Direction wins.

**Cross-surface deduplication.**
If a topic is surfaced in Today, it is not also surfaced proactively in Dialogue or a Space on the same day.

**Memory extraction does not delay response.**
Extraction runs async after the response is delivered. The user never waits for memory processing.

---

## Step 6 — Surface Routing

The Master Agent routes final output to the correct surface.

```
Output type         Surface
────────────────────────────────────────
Morning Brief       Today (Primary Thought)
Recommendation      Today (if day's first) or Dialogue (user-triggered)
Question            Dialogue (proactive) or within conversation
Warning             Today (overrides) + optionally Dialogue
Space update        Relevant Space
Memory update       Mirror (silent)
Continuity update   Continuity (silent)
Silence             No surface — nothing happens
```

---

## Step 7 — Post-Processing

After the output is delivered, the Master Agent triggers async post-processing:

```
1. Memory extraction
   Extract events, facts, decisions, preferences from the interaction.
   Run in background. Does not block response.

2. Confidence updates
   Apply user engagement signals to source Beliefs.
   Did they engage, dismiss, or ignore?

3. Open loop management
   Was an open loop addressed? Mark as resolved or update status.
   Did this interaction create a new open loop?

4. Thread evaluation
   Did this interaction add evidence to an active Thread?
   Did it create conditions for a new Thread?

5. Scheduler queue
   If this interaction warrants a nightly re-evaluation, flag it.
```

All post-processing is fire-and-forget. The user sees no indication it is running.

---

## Master Agent LLM Calls

The Master Agent makes two types of LLM calls:

### Fast call (classification, extraction, gate checks)
- Model: Haiku-equivalent
- Purpose: input classification, Stillness gate checks, memory extraction, Pattern detection
- Latency target: < 300ms
- Never shown to user directly

### Full call (response generation)
- Model: Sonnet-equivalent
- Purpose: Dialogue responses, Morning Brief, Recommendations, Questions
- Latency target: streaming begins < 500ms
- Context package always attached

The Master Agent never makes a full LLM call without first running a fast call to determine if it is warranted.

---

## State the Master Agent Maintains

Between calls, the Master Agent maintains:

```
session_state:
  current_surface         — where the user is right now
  active_domain           — which Space if any
  conversation_context    — last N turns for Dialogue continuity
  today_cache             — current Today content + build timestamp
  cooldown_registry       — recent outputs + timestamps
  open_loops              — active unresolved threads
  pending_extractions     — async jobs in progress
```

Session state is held in memory during the session. Persisted to storage at session end.

---

## What the Master Agent Does Not Do

- Generate content without a context package
- Store memory directly (delegates to Memory subsystem)
- Form Beliefs (delegates to Brain)
- Make priority decisions without Reasoning Engine
- Surface anything that has not passed Stillness
- Make multiple full LLM calls in sequence for a single user interaction (if parallel calls are needed, they run concurrently)

---

## Failure Modes

### LLM timeout
If full LLM call exceeds timeout:
- Dialogue: "Give me a moment." — retry once. If second fails: "Something went wrong. Try again."
- Today: show cached state. Log for nightly rebuild.

### Extraction failure
Post-processing failed silently. No impact on user experience. Logged for retry in nightly run.

### Confidence system inconsistency
If two subsystems produce contradictory confidence updates for the same memory item: hold both, flag for nightly reconciliation. Do not resolve in real-time.

### Domain data ingestion error
Parse fails or data format unexpected. Log error. Do not surface partial or corrupted signals. Notify user: "The upload could not be processed. [Reason if available.]"

### No LLM available
Fallback to cached Today. Dialogue responds: "I am not available right now." Do not attempt to respond with degraded output.
