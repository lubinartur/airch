# Background Jobs — Technical Spec

> Workers, queues, retries, async.
> All jobs run locally. No external queue service in MVP.

---

## Architecture

MVP uses Python's `asyncio` + FastAPI `BackgroundTasks` for lightweight jobs, and APScheduler for time-based jobs.

No Redis, no Celery, no external queue. Everything in-process.

```
FastAPI process
  ├── Request handlers (sync responses)
  ├── BackgroundTasks (fire-and-forget per request)
  └── APScheduler (time-based jobs, runs in background thread)
       ├── morning_run     — daily 05:00
       ├── nightly_run     — daily 01:00
       └── space_reassess  — per-space, every 14 days
```

Post-session extraction and domain ingestion are triggered by request handlers via `BackgroundTasks` — they do not go through APScheduler.

---

## Job Implementations

### Morning Run

```python
# services/scheduler/morning_run.py

async def morning_run(db: Database):
    job_id = await start_job(db, "morning_run")
    try:
        # 1. Check if today already has a valid cache
        today = date.today().isoformat()
        cached = db.execute(
            "SELECT id FROM today_cache WHERE date = ?", (today,)
        ).fetchone()
        if cached:
            await complete_job(db, job_id, items_processed=0)
            return

        # 2. Collect active Beliefs (memory_items type=pattern|fact, confidence >= 0.45)
        beliefs = db.execute("""
            SELECT * FROM memory_items
            WHERE item_type IN ('pattern', 'fact')
            AND confidence >= 0.45
            AND status = 'active'
            ORDER BY confidence DESC
        """).fetchall()

        # 3. Collect active Threads
        threads = db.execute("""
            SELECT * FROM threads WHERE status IN ('active', 'resolving')
        """).fetchall()

        # 4. Collect open loops (top 3 by priority)
        loops = db.execute("""
            SELECT * FROM open_loops
            WHERE status = 'open'
            ORDER BY priority DESC, created_at ASC
            LIMIT 3
        """).fetchall()

        # 5. Build context for Reasoning Engine
        context = build_today_context(beliefs, threads, loops, db)

        # 6. Run Reasoning Engine priority ordering
        top_signal = reasoning_engine.select_top_signal(beliefs, threads, loops)

        # 7. Stillness check
        if not await stillness_check(top_signal, db):
            await save_today_cache(db, today, state="quiet")
            await complete_job(db, job_id, items_processed=len(beliefs))
            return

        # 8. Generate Morning Brief (Sonnet call)
        brief = await llm_generate_today(top_signal, context)

        # 9. Save to today_cache
        await save_today_cache(db, today, state="normal", brief=brief, signal=top_signal)

        await complete_job(db, job_id, items_processed=len(beliefs))

    except Exception as e:
        await fail_job(db, job_id, str(e))
        # Fallback: save quiet state so Today doesn't break
        await save_today_cache(db, date.today().isoformat(), state="quiet")
```

---

### Nightly Run

```python
# services/scheduler/nightly_run.py

async def nightly_run(db: Database):
    job_id = await start_job(db, "nightly_run")
    counts = {"observations_processed": 0, "patterns_updated": 0, "threads_updated": 0}
    try:
        await expire_observations(db, counts)
        await detect_patterns(db, counts)
        await update_thread_status(db, counts)
        await apply_confidence_decay(db, counts)
        await check_mirror_conflicts(db)
        await update_direction(db)
        await update_continuity(db)
        await cleanup_cooldowns(db)
        await complete_job(db, job_id, items_processed=sum(counts.values()))
    except Exception as e:
        await fail_job(db, job_id, str(e))
```

**Key sub-functions:**

```python
async def expire_observations(db, counts):
    expired = db.execute("""
        UPDATE memory_items
        SET status = 'archived'
        WHERE item_type = 'observation'
        AND expires_at < datetime('now')
        AND status = 'active'
    """)
    counts["observations_processed"] += expired.rowcount

async def apply_confidence_decay(db, counts):
    # Facts: -0.02 per month past 30 days without update
    db.execute("""
        UPDATE memory_items
        SET confidence = MAX(0.1, confidence - 0.02),
            updated_at = datetime('now')
        WHERE item_type IN ('fact', 'preference', 'habit')
        AND status = 'active'
        AND julianday('now') - julianday(last_confirmed) > 30
        AND confirmed_by != 'user'
    """)

async def detect_patterns(db, counts):
    # Group facts by domain, look for recurrence
    # Min: 3 supporting facts, 2 distinct time periods, 14-day span
    # Implementation: call Haiku with facts batch, return candidate patterns
    facts = db.execute("""
        SELECT * FROM memory_items
        WHERE item_type = 'fact'
        AND status = 'active'
        AND confidence >= 0.4
        AND created_at > datetime('now', '-30 days')
    """).fetchall()

    if len(facts) < 3:
        return

    candidates = await llm_detect_patterns(facts)  # Haiku call
    for candidate in candidates:
        await upsert_pattern(db, candidate, counts)
```

---

### Post-Session Extraction

```python
# services/scheduler/post_session.py

async def post_session_extract(turn_ids: list[int], db: Database):
    job_id = await start_job(db, "post_session")
    try:
        # 1. Fetch conversation turns
        turns = db.execute(
            f"SELECT role, content FROM dialogue_turns WHERE id IN ({','.join('?' * len(turn_ids))})"
            " ORDER BY id ASC",
            turn_ids
        ).fetchall()

        conversation_text = format_turns_for_extraction(turns)

        # 2. Single unified extraction call (Haiku)
        extracted = await llm_extract_all(conversation_text)
        # Returns: {events, facts, decisions, preferences, open_loops}

        # 3. Store extracted items
        for event in extracted.get("events", []):
            await store_memory_item(db, item_type="event", **event)

        for fact in extracted.get("facts", []):
            await store_memory_item(db, item_type="observation", **fact)
            # stored as observation first; Brain promotes to fact on recurrence

        for decision in extracted.get("decisions", []):
            await store_memory_item(db, item_type="decision", **decision)

        for pref in extracted.get("preferences", []):
            await store_memory_item(db, item_type="observation", **pref)

        for loop in extracted.get("open_loops", []):
            await upsert_open_loop(db, **loop, source_turn_id=turn_ids[-1])

        # 4. Generate embeddings for new items (async, non-blocking)
        asyncio.create_task(generate_embeddings_for_new_items(db))

        # 5. Check for Thread evidence
        asyncio.create_task(evaluate_threads(db))

        await complete_job(db, job_id, items_processed=len(turns))

    except Exception as e:
        await fail_job(db, job_id, str(e))
        # Conversation is always saved in dialogue_turns — no data loss
        # Nightly run will retry extraction from unprocessed turns
```

---

### Domain Data Ingestion

```python
# services/scheduler/domain_ingestion.py

async def process_finance_upload(upload_id: int, db: Database):
    job_id = await start_job(db, "domain_ingestion")
    try:
        # 1. Categorize uncategorized transactions (Haiku, batched by 20)
        uncategorized = db.execute("""
            SELECT id, description, amount FROM transactions
            WHERE upload_id = ? AND category IS NULL
        """, (upload_id,)).fetchall()

        for batch in chunks(uncategorized, 20):
            categories = await llm_categorize_batch(batch)
            for tx_id, category in categories.items():
                db.execute(
                    "UPDATE transactions SET category = ? WHERE id = ?",
                    (category, tx_id)
                )

        # 2. Generate domain signals (spending summary, anomalies)
        signals = compute_finance_signals(upload_id, db)

        # 3. Route signals to Brain (create observations)
        for signal in signals:
            await create_observation(db, **signal)

        # 4. Check for immediate surfacing (Warning threshold)
        warning = await check_warning_threshold(signals, db)
        if warning:
            await surface_warning(warning, db)

        # 5. Update Finance Space
        await update_space_read(db, domain="finance")

        db.commit()
        await complete_job(db, job_id, items_processed=len(uncategorized))

    except Exception as e:
        db.rollback()
        await fail_job(db, job_id, str(e))
```

---

## Job State Management

```python
async def start_job(db, job_type: str) -> int:
    cursor = db.execute("""
        INSERT INTO scheduler_jobs (job_type, status, started_at)
        VALUES (?, 'running', datetime('now'))
    """, (job_type,))
    db.commit()
    return cursor.lastrowid

async def complete_job(db, job_id: int, items_processed: int = 0):
    db.execute("""
        UPDATE scheduler_jobs
        SET status = 'completed',
            completed_at = datetime('now'),
            duration_ms = CAST((julianday('now') - julianday(started_at)) * 86400000 AS INTEGER),
            items_processed = ?
        WHERE id = ?
    """, (items_processed, job_id))
    db.commit()

async def fail_job(db, job_id: int, error: str):
    db.execute("""
        UPDATE scheduler_jobs
        SET status = 'failed',
            completed_at = datetime('now'),
            error_message = ?
        WHERE id = ?
    """, (error[:500], job_id))
    db.commit()
```

---

## APScheduler Setup

```python
# main.py

from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.cron import CronTrigger

scheduler = AsyncIOScheduler()

@asynccontextmanager
async def lifespan(app: FastAPI):
    scheduler.add_job(
        morning_run_wrapper,
        CronTrigger(hour=5, minute=0),
        id="morning_run",
        replace_existing=True
    )
    scheduler.add_job(
        nightly_run_wrapper,
        CronTrigger(hour=1, minute=0),
        id="nightly_run",
        replace_existing=True
    )
    scheduler.start()
    yield
    scheduler.shutdown()

app = FastAPI(lifespan=lifespan)
```

---

## Retry Policy

### Post-session extraction
- No automatic retry (data is safe in `dialogue_turns`)
- Nightly run picks up unprocessed sessions: checks for turns with no extracted memory items created within 5 minutes after the turn

### Morning Run
- If fails: retry once after 10 minutes
- If second attempt fails: save quiet Today state, log error

### Nightly Run
- If fails mid-way: log which step failed
- Completed steps are idempotent (safe to re-run)
- Manual trigger available via `/internal/scheduler/nightly-run`

### Domain ingestion
- If categorization LLM call fails: retry batch after 30 seconds, max 3 attempts
- If still fails: mark transactions as uncategorized, continue with rest
- Surface to user: "5 transactions could not be categorized automatically."

---

## Concurrency Guards

No two instances of the same job run simultaneously:

```python
async def morning_run_wrapper():
    running = db.execute("""
        SELECT id FROM scheduler_jobs
        WHERE job_type = 'morning_run'
        AND status = 'running'
        AND started_at > datetime('now', '-30 minutes')
    """).fetchone()
    if running:
        return  # already running, skip
    await morning_run(db)
```

---

## LLM Call Budget per Job

```
morning_run:           1 Sonnet call (Today generation)
                       0–1 Haiku calls (Stillness check)

nightly_run:           1–3 Haiku calls (pattern detection, direction update, continuity)
                       0 Sonnet calls

post_session:          1 Haiku call (unified extraction)
                       0 Sonnet calls

domain_ingestion:      N Haiku calls (categorization batches, ~1 per 20 transactions)
                       0–1 Haiku calls (signal generation)
                       0 Sonnet calls

space_reassessment:    1 Haiku call (AIRCH Read regeneration)
                       0 Sonnet calls
```

Total daily baseline (no user interaction): 2–5 Haiku calls + 1 Sonnet call.
Per user message: 1 Sonnet call (streaming) + 1 Haiku call (extraction, async).
