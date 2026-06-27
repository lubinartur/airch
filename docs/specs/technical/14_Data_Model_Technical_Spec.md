# Data Model — Technical Spec

> SQLite. Local only. One file: airch.db
> All personal data stays on device.

---

## Pragmas (on every connection)

```sql
PRAGMA journal_mode=WAL;
PRAGMA foreign_keys=ON;
PRAGMA synchronous=NORMAL;
PRAGMA temp_store=MEMORY;
PRAGMA cache_size=-64000;
```

---

## Schema

### user_profile
Single row. Core identity. Always in context.

```sql
CREATE TABLE user_profile (
    id                  INTEGER PRIMARY KEY CHECK (id = 1),
    name                TEXT,
    city                TEXT,
    profession          TEXT,
    monthly_income      REAL,
    direction_summary   TEXT,        -- 2–3 sentences, updated by nightly run
    key_context         TEXT,        -- free text, max 500 chars, manually editable
    air4_mode           TEXT DEFAULT 'normal',  -- quiet / normal / active / jarvis
    timezone            TEXT DEFAULT 'UTC',
    created_at          TEXT DEFAULT (datetime('now')),
    updated_at          TEXT DEFAULT (datetime('now'))
);

CREATE UNIQUE INDEX idx_user_profile_single ON user_profile(id);
```

---

### memory_items
Central memory table. All Facts, Patterns, Habits, Preferences, Values, Events, Decisions.

```sql
CREATE TABLE memory_items (
    id              INTEGER PRIMARY KEY,

    -- Type
    item_type       TEXT NOT NULL,
    -- 'observation' | 'fact' | 'event' | 'decision' | 'preference'
    -- 'habit' | 'pattern' | 'value'

    -- Content
    title           TEXT NOT NULL,       -- short label, used in Mirror
    body            TEXT,                -- full description
    domain          TEXT,
    -- 'finance' | 'health' | 'projects' | 'life' | 'personal' | 'cross_domain'

    -- Confidence
    confidence      REAL DEFAULT 0.5,    -- 0.0–1.0
    confirmed_by    TEXT DEFAULT 'inferred',  -- 'inferred' | 'user' | 'data'

    -- For patterns and cross-domain items
    domains_involved TEXT,               -- JSON array: ["finance", "projects"]
    evidence_refs    TEXT,               -- JSON array of memory_item ids

    -- Lifecycle
    status          TEXT DEFAULT 'active',
    -- 'active' | 'archived' | 'dormant' | 'deleted'

    -- Sensitivity flag
    is_sensitive    INTEGER DEFAULT 0,   -- 1 = only shown if user raises topic

    -- Embedding
    embedding_id    INTEGER,             -- FK to embeddings table

    -- Source
    source          TEXT DEFAULT 'inferred',
    -- 'inferred' | 'user_stated' | 'user_confirmed' | 'data_derived'

    -- Timestamps
    first_observed  TEXT DEFAULT (datetime('now')),
    last_confirmed  TEXT,
    expires_at      TEXT,                -- for observations: +7 days
    created_at      TEXT DEFAULT (datetime('now')),
    updated_at      TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_memory_type ON memory_items(item_type);
CREATE INDEX idx_memory_domain ON memory_items(domain);
CREATE INDEX idx_memory_confidence ON memory_items(confidence);
CREATE INDEX idx_memory_status ON memory_items(status);
CREATE INDEX idx_memory_sensitive ON memory_items(is_sensitive);
```

---

### memory_corrections
Log of all user actions on memory items in Mirror.

```sql
CREATE TABLE memory_corrections (
    id              INTEGER PRIMARY KEY,
    memory_item_id  INTEGER NOT NULL REFERENCES memory_items(id),
    action          TEXT NOT NULL,
    -- 'confirm' | 'mark_outdated' | 'correct' | 'add_context' | 'delete' | 'dismiss'
    old_value       TEXT,                -- JSON snapshot before change
    new_value       TEXT,               -- JSON snapshot after change (if applicable)
    user_note       TEXT,               -- annotation added by user
    created_at      TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_corrections_item ON memory_corrections(memory_item_id);
```

---

### embeddings
Vector representations for semantic search.

```sql
CREATE TABLE embeddings (
    id              INTEGER PRIMARY KEY,
    source_type     TEXT NOT NULL,       -- 'memory_item' | 'dialogue_turn' | 'event'
    source_id       INTEGER NOT NULL,
    content_preview TEXT,                -- first 200 chars for debug
    vector          BLOB NOT NULL,       -- serialized float32 array
    model           TEXT NOT NULL,       -- embedding model name
    dimensions      INTEGER NOT NULL,
    created_at      TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_embeddings_source ON embeddings(source_type, source_id);
```

---

### open_loops
Unresolved questions and threads from Dialogue.

```sql
CREATE TABLE open_loops (
    id              INTEGER PRIMARY KEY,
    topic           TEXT NOT NULL,
    description     TEXT,
    domain          TEXT,
    priority        INTEGER DEFAULT 2,   -- 1=low, 2=medium, 3=high
    status          TEXT DEFAULT 'open', -- 'open' | 'resolved' | 'archived'
    source_turn_id  INTEGER,             -- dialogue_turns FK
    resolved_at     TEXT,
    expires_at      TEXT,                -- +30 days from creation
    created_at      TEXT DEFAULT (datetime('now')),
    updated_at      TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_loops_status ON open_loops(status);
CREATE INDEX idx_loops_priority ON open_loops(priority);
CREATE INDEX idx_loops_domain ON open_loops(domain);
```

---

### threads
Cross-domain stories.

```sql
CREATE TABLE threads (
    id              INTEGER PRIMARY KEY,
    name            TEXT NOT NULL,
    core_tension    TEXT NOT NULL,       -- one sentence
    domains         TEXT NOT NULL,       -- JSON array
    status          TEXT DEFAULT 'emerging',
    -- 'emerging' | 'active' | 'resolving' | 'closed'
    confidence      REAL DEFAULT 0.4,
    evidence_refs   TEXT,                -- JSON array of memory_item ids
    created_at      TEXT DEFAULT (datetime('now')),
    updated_at      TEXT DEFAULT (datetime('now')),
    resolved_at     TEXT
);

CREATE INDEX idx_threads_status ON threads(status);
```

---

### dialogue_turns
Full conversation history.

```sql
CREATE TABLE dialogue_turns (
    id              INTEGER PRIMARY KEY,
    role            TEXT NOT NULL,       -- 'user' | 'assistant'
    content         TEXT NOT NULL,
    surface         TEXT,                -- 'dialogue' | 'space:finance' | 'space:health' etc.
    agent           TEXT,                -- 'master' | 'finance' | 'health' | 'projects' | 'life'
    embedding_id    INTEGER,
    created_at      TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_turns_created ON dialogue_turns(created_at);
CREATE INDEX idx_turns_surface ON dialogue_turns(surface);
```

---

### today_cache
Cached Today content. One active row.

```sql
CREATE TABLE today_cache (
    id              INTEGER PRIMARY KEY,
    date            TEXT NOT NULL UNIQUE,    -- YYYY-MM-DD
    state           TEXT NOT NULL,           -- 'normal' | 'quiet' | 'warning'
    primary_thought TEXT,
    focus_domain    TEXT,
    supporting      TEXT,                    -- JSON array, max 2 items
    thread_ref      INTEGER REFERENCES threads(id),
    belief_ref      INTEGER REFERENCES memory_items(id),
    generated_at    TEXT DEFAULT (datetime('now'))
);
```

---

### spaces
Domain rooms.

```sql
CREATE TABLE spaces (
    id              INTEGER PRIMARY KEY,
    name            TEXT NOT NULL,
    space_type      TEXT DEFAULT 'user',     -- 'system' | 'user' | 'airch_proposed'
    domain          TEXT,
    -- 'finance' | 'health' | 'projects' | 'life' | 'personal' | null (custom)
    airch_read      TEXT,                    -- current one-sentence assessment
    momentum        TEXT DEFAULT 'steady',   -- 'moving' | 'steady' | 'stalling' | 'dormant'
    status          TEXT DEFAULT 'active',   -- 'active' | 'archived' | 'dormant'
    last_meaningful_activity TEXT,
    reassess_after  TEXT,                    -- date for next Space Reassessment Run
    created_at      TEXT DEFAULT (datetime('now')),
    updated_at      TEXT DEFAULT (datetime('now'))
);
```

---

### Finance tables

```sql
CREATE TABLE uploads (
    id                  INTEGER PRIMARY KEY,
    filename            TEXT NOT NULL,
    account_iban        TEXT,
    period_start        TEXT,
    period_end          TEXT,
    total_transactions  INTEGER,
    created_at          TEXT DEFAULT (datetime('now'))
);

CREATE TABLE transactions (
    id                    INTEGER PRIMARY KEY,
    upload_id             INTEGER REFERENCES uploads(id),
    transaction_hash      TEXT UNIQUE,
    date                  TEXT NOT NULL,
    description           TEXT,
    amount                REAL NOT NULL,
    currency              TEXT DEFAULT 'EUR',
    category              TEXT,
    category_confirmed    INTEGER DEFAULT 0,
    account_iban          TEXT,
    is_debit              INTEGER DEFAULT 1,
    is_internal_transfer  INTEGER DEFAULT 0,
    raw_description       TEXT,
    created_at            TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_tx_date ON transactions(date);
CREATE INDEX idx_tx_category ON transactions(category);
CREATE INDEX idx_tx_hash ON transactions(transaction_hash);
CREATE INDEX idx_tx_debit ON transactions(is_debit, is_internal_transfer);

CREATE TABLE subscriptions (
    id              INTEGER PRIMARY KEY,
    name            TEXT NOT NULL,
    amount          REAL NOT NULL,
    currency        TEXT DEFAULT 'EUR',
    billing_day     INTEGER,
    is_active       INTEGER DEFAULT 1,
    last_charged    TEXT,
    created_at      TEXT DEFAULT (datetime('now')),
    updated_at      TEXT DEFAULT (datetime('now'))
);

CREATE TABLE obligations (
    id              INTEGER PRIMARY KEY,
    name            TEXT NOT NULL,
    amount          REAL NOT NULL,
    currency        TEXT DEFAULT 'EUR',
    billing_day     INTEGER,
    total_debt      REAL,
    interest_rate   REAL,
    is_active       INTEGER DEFAULT 1,
    created_at      TEXT DEFAULT (datetime('now')),
    updated_at      TEXT DEFAULT (datetime('now'))
);
```

---

### Health tables

```sql
CREATE TABLE workouts (
    id              INTEGER PRIMARY KEY,
    date            TEXT NOT NULL,
    type            TEXT,                -- 'strength' | 'cardio' | 'flexibility' | 'other'
    duration        INTEGER,             -- minutes
    exercises       TEXT,                -- JSON array
    energy_level    INTEGER,             -- 1–5
    notes           TEXT,
    source          TEXT DEFAULT 'chat', -- 'chat' | 'manual' | 'import'
    created_at      TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_workouts_date ON workouts(date);

CREATE TABLE body_metrics (
    id          INTEGER PRIMARY KEY,
    date        TEXT NOT NULL,
    weight      REAL,
    height      REAL,
    body_fat    REAL,
    notes       TEXT,
    source      TEXT DEFAULT 'manual',
    created_at  TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_metrics_date ON body_metrics(date);

CREATE TABLE health_checkups (
    id              INTEGER PRIMARY KEY,
    date            TEXT NOT NULL,
    marker_name     TEXT NOT NULL,
    value           REAL NOT NULL,
    unit            TEXT,
    reference_min   REAL,
    reference_max   REAL,
    status          TEXT,  -- 'normal' | 'high' | 'low'
    notes           TEXT,
    created_at      TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_checkups_date ON health_checkups(date);
CREATE INDEX idx_checkups_marker ON health_checkups(marker_name);
```

---

### Projects tables

```sql
CREATE TABLE projects (
    id              INTEGER PRIMARY KEY,
    name            TEXT NOT NULL,
    description     TEXT,
    status          TEXT DEFAULT 'active',
    -- 'active' | 'stalled' | 'completed' | 'archived'
    priority        INTEGER DEFAULT 2,
    space_id        INTEGER REFERENCES spaces(id),
    started_at      TEXT,
    created_at      TEXT DEFAULT (datetime('now')),
    updated_at      TEXT DEFAULT (datetime('now'))
);

CREATE TABLE project_logs (
    id          INTEGER PRIMARY KEY,
    project_id  INTEGER REFERENCES projects(id) ON DELETE CASCADE,
    note        TEXT NOT NULL,
    log_type    TEXT DEFAULT 'update',
    -- 'update' | 'milestone' | 'blocker' | 'decision'
    source      TEXT DEFAULT 'manual',
    created_at  TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_logs_project ON project_logs(project_id);
CREATE INDEX idx_logs_date ON project_logs(created_at);
```

---

### Scheduler state

```sql
CREATE TABLE scheduler_jobs (
    id              INTEGER PRIMARY KEY,
    job_type        TEXT NOT NULL,
    -- 'morning_run' | 'nightly_run' | 'post_session' | 'domain_ingestion' | 'space_reassessment'
    status          TEXT DEFAULT 'pending',
    -- 'pending' | 'running' | 'completed' | 'failed'
    started_at      TEXT,
    completed_at    TEXT,
    duration_ms     INTEGER,
    items_processed INTEGER DEFAULT 0,
    error_message   TEXT,
    created_at      TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_jobs_type ON scheduler_jobs(job_type);
CREATE INDEX idx_jobs_status ON scheduler_jobs(status);
CREATE INDEX idx_jobs_created ON scheduler_jobs(created_at);
```

---

### App meta (migration gating)

```sql
CREATE TABLE _app_meta (
    key     TEXT PRIMARY KEY,
    value   TEXT NOT NULL
);
```

---

## Entity Relationships

```
user_profile         ← single row, referenced everywhere via context
memory_items         ← central; referenced by threads, open_loops
memory_corrections   → memory_items
embeddings           → memory_items | dialogue_turns
open_loops           → dialogue_turns (source_turn_id)
threads              → memory_items (evidence_refs, JSON)
dialogue_turns       ← full history
today_cache          → threads, memory_items
spaces               ← rooms; projects reference spaces
transactions         → uploads
project_logs         → projects
health_checkups      ← standalone (marker history)
scheduler_jobs       ← standalone (job log)
```

---

## Migration Pattern

Every schema change is gated through `_app_meta`:

```python
def run_migration(db, key: str, sql: str):
    if not db.execute(
        "SELECT 1 FROM _app_meta WHERE key = ?", (key,)
    ).fetchone():
        db.executescript(sql)
        db.execute(
            "INSERT INTO _app_meta (key, value) VALUES (?, ?)",
            (key, datetime.utcnow().isoformat())
        )
        db.commit()
```

Migrations run on startup. Idempotent. Never destructive without explicit backup step.
