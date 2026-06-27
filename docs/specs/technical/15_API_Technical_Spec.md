# API — Technical Spec

> FastAPI. All endpoints local (127.0.0.1:8000).
> No authentication in MVP. Add X-AIRCH-Key in Phase 2.

---

## Base

```
Base URL:     http://127.0.0.1:8000
Content-Type: application/json
Streaming:    text/event-stream (SSE) for chat
```

---

## Chat

### POST /api/chat
Send a message. Returns streaming SSE response.

**Request:**
```json
{
  "message": "I keep avoiding the investor call",
  "surface": "dialogue",
  "agent": "master",
  "space_id": null
}
```

**Response:** `text/event-stream`
```
data: {"type": "delta", "content": "We"}
data: {"type": "delta", "content": " have"}
data: {"type": "delta", "content": " been here before."}
data: {"type": "done", "turn_id": 42}
```

**After stream completes:**
- Turn saved to `dialogue_turns`
- Post-session extraction triggered async

---

### GET /api/chat/history
```
Query: ?limit=50&surface=dialogue
Response: [{"role": "user"|"assistant", "content": "...", "created_at": "..."}]
```

---

### GET /api/chat/morning-brief
Returns today's proactive opening for Dialogue (if any).
```json
{
  "has_brief": true,
  "content": "We left the Mihkel question unresolved. You have the meeting today.",
  "open_loop_id": 7
}
```
Returns `{"has_brief": false}` if no proactive opening warranted.
Cached per day. Regenerates after midnight.

---

## Today

### GET /api/today
Returns cached Today content.

```json
{
  "date": "2026-06-13",
  "state": "normal",
  "primary_thought": "The Mihkel conversation is today. You have been avoiding preparing for it.",
  "focus_domain": "projects",
  "supporting": [
    {"domain": "finance", "text": "Spending running 18% above last cycle."}
  ],
  "thread": {
    "id": 3,
    "name": "The investment question"
  },
  "generated_at": "2026-06-13T05:14:22"
}
```

State values: `normal` | `quiet` | `warning`

Cache miss (no today_cache row for today) → triggers Morning Run → returns result.
Timeout > 10s → returns quiet state.

---

## Memory / Mirror

### GET /api/mirror
Returns all memory items visible in Mirror (confidence ≥ 0.3, status=active).

```
Query: ?type=fact|pattern|habit|preference|value&domain=finance|health|...
Response: [{
  "id": 1,
  "item_type": "pattern",
  "title": "Restaurant spending rises during high-stress project periods",
  "body": "...",
  "domain": "finance",
  "confidence": 0.72,
  "confidence_label": "This appears to be a pattern",
  "domains_involved": ["finance", "projects"],
  "is_sensitive": false,
  "first_observed": "2026-03-01",
  "last_confirmed": "2026-05-14"
}]
```

---

### PATCH /api/mirror/{id}
User action on a memory item.

**Request:**
```json
{
  "action": "confirm" | "mark_outdated" | "correct" | "add_context" | "delete" | "dismiss",
  "new_value": "corrected text (if action=correct)",
  "user_note": "annotation (if action=add_context)"
}
```

**Effect:** updates `memory_items.confidence`, logs to `memory_corrections`, returns updated item.

---

### GET /api/memory/open-loops
```json
[{
  "id": 7,
  "topic": "What to do about the investment question",
  "domain": "projects",
  "priority": 3,
  "days_open": 18
}]
```

---

## Spaces

### GET /api/spaces
```json
[{
  "id": 1,
  "name": "Finance",
  "space_type": "system",
  "domain": "finance",
  "airch_read": "Spending this cycle is running 18% above the last three.",
  "momentum": "stalling",
  "last_meaningful_activity": "2026-06-10"
}]
```

### GET /api/spaces/{id}
Full Space detail: read, open questions, recent events, connected threads.

### PATCH /api/spaces/{id}
Update name, status (archive/restore). System spaces cannot be deleted.

### POST /api/spaces
Create a user Space.
```json
{"name": "Ducati", "domain": null}
```

---

## Threads

### GET /api/threads
```
Query: ?status=active|emerging|resolving|closed
```
```json
[{
  "id": 3,
  "name": "The investment question",
  "core_tension": "Wants to build independently but unsure if capital is blocking progress.",
  "domains": ["projects", "finance", "life"],
  "status": "active",
  "confidence": 0.68,
  "days_active": 18
}]
```

### PATCH /api/threads/{id}
```json
{
  "action": "confirm" | "dismiss" | "rename" | "mark_resolved",
  "name": "new name (if rename)"
}
```

---

## Finance

### POST /api/finance/upload
Multipart form. Accepts Swedbank CSV.

**Response:**
```json
{
  "upload_id": 5,
  "period_start": "2026-05-10",
  "period_end": "2026-06-09",
  "total_transactions": 87,
  "categorized": 82,
  "uncategorized": 5
}
```

Categorization runs async after upload. SSE endpoint for progress:
`GET /api/finance/upload/{id}/status` (SSE)

---

### GET /api/finance/summary
```
Query: ?start=2026-05-10&end=2026-06-09
```
```json
{
  "period_start": "2026-05-10",
  "period_end": "2026-06-09",
  "total_income": 3800.00,
  "other_incoming": 240.00,
  "total_debit": 2940.00,
  "free_capital": 860.00,
  "days_until_salary": 8,
  "categories": [
    {"category": "food_restaurants", "amount": 420.00, "delta_pct": 89},
    {"category": "subscriptions", "amount": 205.00, "delta_pct": 0}
  ]
}
```

### GET /api/finance/transactions
```
Query: ?start=...&end=...&category=...&limit=50&offset=0
```

### PUT /api/finance/transactions/{id}/category
```json
{"category": "food_restaurants"}
```

### GET/POST/PUT/DELETE /api/finance/subscriptions
Standard CRUD. See Data Model for fields.

### GET/POST/PUT/DELETE /api/finance/obligations
Standard CRUD.

### GET /api/finance/cycles
```json
{
  "active": {"start": "2026-06-10", "end": "2026-07-09"},
  "latest_with_data": {"start": "2026-05-10", "end": "2026-06-09"},
  "earliest_with_data": {"start": "2026-01-10", "end": "2026-02-09"}
}
```

---

## Health

### GET /api/health/workouts
```
Query: ?limit=20
```

### POST /api/health/workouts
```json
{
  "date": "2026-06-13",
  "type": "strength",
  "duration": 60,
  "exercises": [{"name": "bench_press", "sets": 3, "reps": 10, "weight": 82.5}],
  "energy_level": 4,
  "notes": ""
}
```

### GET/POST /api/health/metrics
Body weight and height entries.

### GET /api/health/checkups
```
Query: ?marker=LDL|testosterone|...
```
Returns history of a marker across all checkup dates.

### GET /api/health/markers/{name}/history
```json
[
  {"date": "2026-03-12", "value": 2.2, "unit": "mmol/L", "status": "normal"},
  {"date": "2025-11-17", "value": 3.2, "unit": "mmol/L", "status": "high"}
]
```

---

## Projects

### GET/POST /api/projects
### GET /api/projects/{id}
Includes: recent logs, momentum, session time this week.

### POST /api/projects/{id}/logs
```json
{"note": "Finished the Dialogue spec", "log_type": "milestone"}
```

### POST/GET /api/projects/{id}/sessions/start|stop
Session timer. Stop returns duration and prompts for log entry.

---

## Scheduler (internal)

These endpoints are for the scheduler to trigger jobs. Not user-facing.

```
POST /internal/scheduler/morning-run
POST /internal/scheduler/nightly-run
POST /internal/scheduler/post-session   body: {"session_turn_ids": [40, 41, 42]}
POST /internal/scheduler/space-reassess body: {"space_id": 2}
```

All return `{"job_id": ..., "status": "started"}` immediately. Jobs run async.

---

## System

### GET /health
```json
{"status": "ok", "db": "ok", "llm": "ok"}
```

### GET /api/air4/mode
```json
{"mode": "normal"}
```

### PUT /api/air4/mode
```json
{"mode": "quiet" | "normal" | "active" | "jarvis"}
```

---

## Error Format

All errors:
```json
{
  "error": "short_code",
  "message": "Human-readable description",
  "detail": {} 
}
```

Common codes:
```
upload_parse_error      CSV could not be parsed
llm_unavailable         LLM not responding
llm_timeout             Response exceeded timeout
not_found               Resource does not exist
invalid_action          Action not valid for this resource state
```

---

## Streaming (SSE) Pattern

All streaming responses follow this pattern:

```python
@router.post("/api/chat")
async def chat(req: ChatRequest):
    async def generate():
        async for delta in llm_stream(prompt, context):
            yield f"data: {json.dumps({'type': 'delta', 'content': delta})}\n\n"
        yield f"data: {json.dumps({'type': 'done', 'turn_id': saved_id})}\n\n"
    
    # Trigger async post-processing after stream
    background_tasks.add_task(post_session_extract, turn_ids)
    
    return StreamingResponse(generate(), media_type="text/event-stream")
```

Frontend reads SSE, appends deltas to message, handles `done` event to finalize.
