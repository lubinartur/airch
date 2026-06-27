# MVP Build Order

> Step-by-step. Cursor reads this before starting any sprint.
> Each step has: what to build, what to skip, done criteria.

---

## Philosophy

Build the minimum that produces a real "чёрт… он прав" moment.

Not the minimum that compiles. The minimum that is genuinely useful.

Every step either enables the next step or produces user value. Nothing else gets built.

---

## The Target

After MVP is complete, the user can:
1. Open AIRCH and see one meaningful thought about their day (Today)
2. Talk to AIRCH and have it remember what was said last time (Dialogue with memory)
3. Upload a bank statement and have AIRCH say something non-obvious about it (Finance signal)
4. See what AIRCH knows about them and correct it (Mirror)

That is it. Everything else is later.

---

## Step 1 — Foundation

**What:** Database + FastAPI skeleton + basic config.

**Build:**
- `airch.db` with migrations system (`_app_meta` table)
- Tables: `user_profile`, `memory_items`, `dialogue_turns`, `open_loops`, `today_cache`, `scheduler_jobs`
- `database.py`: connection, WAL pragma, migration runner
- `main.py`: FastAPI app, lifespan context manager, `/health` endpoint
- `.env`: `ANTHROPIC_API_KEY`, `DB_PATH`, `LLM_FAST_MODEL`, `LLM_SMART_MODEL`
- `llm_client.py`: two functions — `llm_stream()` (Sonnet, SSE) and `llm_call()` (Haiku, sync)

**Skip:** Everything else.

**Done when:** `GET /health` returns `{"status": "ok", "db": "ok", "llm": "ok"}` and migrations run cleanly on fresh DB.

---

## Step 2 — Dialogue (no memory yet)

**What:** Working chat. Streaming. No memory. No extraction.

**Build:**
- `routers/chat.py`: `POST /api/chat` with SSE streaming
- `GET /api/chat/history`
- System prompt: basic AIRCH character (from `CHARACTER_SYSTEM.md`)
- Context: only `user_profile` (hardcoded test profile for now)
- Store turns to `dialogue_turns` after each exchange

**Skip:** Memory extraction, open loops, agents, context builder.

**Done when:** User can have a streaming conversation with AIRCH. Turns are saved. History loads on refresh.

---

## Step 3 — User Profile + Basic Context

**What:** Profile setup + profile goes into every chat context.

**Build:**
- `GET/PUT /api/profile`
- Profile page in frontend: name, city, profession, income, key_context
- `context_builder.py` (Layer 1 only): reads `user_profile`, formats for LLM
- Wire context into `POST /api/chat`

**Skip:** Memory retrieval, domain context, conversation history in context.

**Done when:** AIRCH knows who it is talking to. Profile fields appear naturally in responses.

---

## Step 4 — Memory Extraction

**What:** AIRCH starts remembering what was said.

**Build:**
- `services/extraction/unified_extractor.py`: single Haiku call after each session
  - Extracts: events, facts, preferences, open loops
  - All stored as `memory_items` with `item_type='observation'` or `item_type='event'`
  - Open loops stored to `open_loops`
- Wire into `routers/chat.py` as `BackgroundTasks` (fire-and-forget)
- `services/embeddings.py`: generate and store embeddings for new memory items

**Skip:** Pattern detection, confidence decay, Mirror UI.

**Done when:** After a conversation where the user mentions something about themselves, that thing appears in `memory_items`. Check via sqlite3 CLI.

---

## Step 5 — Memory Retrieval + Context Layer 2

**What:** AIRCH uses past memory in current conversations.

**Build:**
- `services/memory_retrieval.py`:
  - Semantic retrieval (cosine similarity against embeddings)
  - Recency retrieval (always-on)
  - Composite scoring + top-5 selection
- Wire into `context_builder.py` as Layer 2
- Wire retrieval into `POST /api/chat` pipeline

**Skip:** Keyword/structural retrieval (add in Phase 2), full confidence system.

**Done when:** User mentions something from a previous session and AIRCH knows it without being told again.

---

## Step 6 — Mirror (read-only first)

**What:** User can see what AIRCH knows.

**Build:**
- `GET /api/mirror` — returns all `memory_items` with confidence ≥ 0.3, active status
- Mirror page in frontend: grouped by type (facts, patterns, preferences)
- Confidence labels ("This appears to be a pattern", etc.)
- No editing yet — read-only

**Skip:** User actions (confirm/correct/delete) — next step.

**Done when:** Mirror page shows real extracted memory from conversations.

---

## Step 7 — Mirror (editable)

**What:** User can correct what AIRCH knows.

**Build:**
- `PATCH /api/mirror/{id}` — handles: confirm, mark_outdated, correct, delete
- `memory_corrections` table (add migration)
- Mirror UI: confirm/edit/delete actions per item
- Confidence updates after each action

**Done when:** User can delete a wrong fact and it disappears from context on the next conversation.

---

## Step 8 — Finance Upload + Signals

**What:** Upload bank statement. AIRCH says something about it.

**Build:**
- `uploads`, `transactions`, `subscriptions`, `obligations` tables (migrations)
- `POST /api/finance/upload` — parse Swedbank CSV, deduplicate, store
- `services/finance/categorizer.py` — Haiku, batch of 20, category per transaction
- `GET /api/finance/summary` — totals, category breakdown, delta vs previous cycle
- `services/finance/signals.py` — generate observations from summary (spending spike, category anomaly)
- Store observations to `memory_items`
- Finance Space page: summary card, category breakdown, top transactions

**Skip:** Finance Agent (use Master LLM for now), subscriptions UI, obligations UI.

**Done when:** User uploads CSV → sees spending breakdown → AIRCH has created observations about anomalies in `memory_items`.

---

## Step 9 — Today

**What:** AIRCH assembles one primary thought each morning.

**Build:**
- `today_cache` table (already exists from Step 1)
- `services/scheduler/morning_run.py` — collects beliefs, runs Reasoning Engine, generates brief
- `GET /api/today` — returns cached state or triggers morning run
- `services/reasoning_engine.py` — priority ordering (simplified: confidence × recency)
- `services/stillness.py` — Gates 1, 2, 5, 7 (basic implementation; Gate 3 is hard, defer)
- Today page in frontend: primary thought, focus domain, quiet state
- APScheduler wired in `main.py` (morning run at 05:00)

**Skip:** Thread references in Today, supporting context items (add in Phase 2), Gate 3 of Stillness (the hard one).

**Done when:** Today shows a real primary thought based on actual memory and finance signals. Quiet state works correctly.

---

## Step 10 — Nightly Run

**What:** Memory consolidates overnight. Patterns emerge.

**Build:**
- `services/scheduler/nightly_run.py`:
  - Expire old observations
  - Promote observations to facts (recurrence detection: same title/domain, 2+ occurrences in 30 days)
  - Basic pattern detection (Haiku call with recent facts)
  - Confidence decay
- APScheduler wired (nightly at 01:00)

**Skip:** Thread evaluation in nightly run (add in Phase 2), direction update, continuity update.

**Done when:** After a week of use, `memory_items` contains items with `item_type='pattern'`. Today starts surfacing pattern-based thoughts.

---

## Step 11 — Open Loops in Dialogue

**What:** AIRCH remembers unresolved threads and returns to them.

**Build:**
- `GET /api/memory/open-loops`
- Wire open loops into context (Layer 4 recency)
- `GET /api/chat/morning-brief` — proactive Dialogue opening based on open loops
- Dialogue: show morning brief as first message if present
- `PATCH /api/memory/open-loops/{id}` — resolve

**Done when:** User mentions something unresolved in conversation → AIRCH returns to it next day without being asked.

---

## Step 12 — Spaces (Finance + Projects)

**What:** Finance and Projects get dedicated rooms.

**Build:**
- `spaces` table + seed with system spaces (Finance, Projects)
- `GET /api/spaces`, `GET /api/spaces/{id}`
- `services/spaces/space_updater.py` — recalculates AIRCH Read after data changes
- Finance Space page: AIRCH Read + summary (reuse Step 8 components)
- Projects tables + basic Projects Space page
- `POST /api/projects`, `GET /api/projects`, `POST /api/projects/{id}/logs`

**Skip:** Threads, dormancy detection, Space Reassessment scheduler job (Phase 2).

**Done when:** Finance Space shows real AIRCH Read based on current signals. Projects Space shows active projects with logs.

---

## What MVP Does NOT Include

These are explicitly deferred. Do not build them during MVP.

```
Threads                 — Phase 2
Continuity              — Phase 2
Direction page          — Phase 2
Health module           — Phase 2
Space Reassessment job  — Phase 2
Domain Agents           — Phase 2 (use Master LLM everywhere in MVP)
Gate 3 Stillness        — Phase 2 (hardest gate; requires enough data to test)
Memory Lifecycle        — Phase 2 (archival, summarization)
Embedding index         — Phase 2 (use sqlite LIKE for MVP retrieval if needed)
Authentication          — Phase 2
Mobile                  — Phase 3
Voice                   — Phase 3
```

---

## Build Order Summary

```
Step 1   Foundation          DB + FastAPI + LLM client
Step 2   Dialogue            Streaming chat, no memory
Step 3   Profile             User identity in context
Step 4   Memory Extraction   AIRCH starts remembering
Step 5   Memory Retrieval    AIRCH uses memory in chat
Step 6   Mirror (read)       User sees what AIRCH knows
Step 7   Mirror (edit)       User corrects AIRCH
Step 8   Finance             Upload → signals → observations
Step 9   Today               One primary thought per day
Step 10  Nightly Run         Patterns emerge overnight
Step 11  Open Loops          AIRCH returns to unresolved threads
Step 12  Spaces              Finance + Projects rooms
```

Each step ships working software. No step is "infrastructure only" — every step produces something the user can see or feel.

---

## Done Criteria for MVP

MVP is done when:

1. User uploads a bank statement and AIRCH says something they did not already know
2. User has a conversation and AIRCH remembers it the next day
3. Today shows one real primary thought based on actual data (not placeholder)
4. User can open Mirror and see what AIRCH believes about them — and correct it
5. At least one moment where the user thinks: "чёрт… он прав"

If all five are true — MVP is done. Start using it. Iterate from real data.
