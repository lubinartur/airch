# Scheduler — Operational Spec

> What AIRCH recalculates, when, and why.

---

## Role

The Scheduler runs background jobs that keep AIRCH's understanding current between user interactions.

Most of AIRCH's intelligence is built not during conversations — but between them.

The Scheduler is what makes AIRCH feel like it has been paying attention even when the user was not there.

---

## Job Types

### 1. Morning Run
**When:** 05:00–06:00 local time, or on first app open after 06:00 if the 05:00 run was missed.
**Trigger:** Time-based (daily).
**Purpose:** Build Today for the day.

```
Steps:
  1. Collect all active Beliefs from Brain (confidence ≥ 0.45)
  2. Collect all domain signals from last 24 hours
  3. Collect active Threads and open loops
  4. Run Reasoning Engine — priority ordering
  5. Run Stillness check on top signal
  6. If passes: generate Morning Brief text (full LLM call, Sonnet)
  7. If fails: prepare quiet Today state
  8. Cache Today result with timestamp
  9. Log: what was considered, what was selected, why
```

**Output:** Today content (Primary Thought or quiet state), cached until midnight.

**Failure handling:** If Morning Run fails or times out → Today shows yesterday's quiet state with a soft refresh prompt. Does not show an error to the user.

---

### 2. Nightly Run
**When:** 01:00–02:00 local time.
**Trigger:** Time-based (daily).
**Purpose:** Memory consolidation, Pattern detection, confidence updates.

```
Steps:
  1. Collect all Observations from the last 7 days
     → Evaluate: has any Observation been confirmed? → promote to Fact
     → Evaluate: has any Observation expired? → discard
  
  2. Run Pattern detection across all Facts
     → For each domain: look for Facts that share a recurring structure
     → For cross-domain: look for temporal overlaps (see Brain spec thresholds)
     → New Patterns created or existing Pattern confidence updated
  
  3. Thread evaluation
     → Check all active Threads: any new evidence? Confidence update.
     → Check all Emerging Threads: enough evidence to promote to Active?
     → Check all Resolving Threads: confirmed resolved? → Close and archive to Continuity.
  
  4. Confidence decay
     → Apply decay rules to all Facts, Patterns, Preferences
     → (see Memory spec for decay rates)
     → Items that drop below 0.1 → archived, not deleted
  
  5. Mirror consistency check
     → Any conflicting memory items that have been unresolved for 30+ days?
     → Flag for user review (surfaced in Mirror on next open, not pushed)
  
  6. Direction update
     → Has enough new evidence accumulated to revise the Direction interpretation?
     → If yes: update Direction; log what changed and why
     → If significant change: flag for user confirmation in Mirror
  
  7. Continuity update
     → Any Events or closed Threads to add to the life arc?
     → Update Continuity narrative (Haiku-level call, no full LLM)
  
  8. Cooldown registry cleanup
     → Remove cooldown entries older than their window
  
  9. Log summary: what changed, what was created, what was archived
```

**Output:** Updated Memory state, Pattern updates, Continuity updates. No user-facing output unless flagged items surface in Mirror.

**Duration target:** < 5 minutes total. Runs locally, async.

---

### 3. Post-Session Run
**When:** Within 60 seconds after a Dialogue session ends (user stops sending messages for > 5 minutes, or explicitly closes Dialogue).
**Trigger:** Session end event.
**Purpose:** Extract and store everything from the conversation.

```
Steps:
  1. Memory extraction (single unified LLM call, Haiku)
     Input: full conversation turns from this session
     Extract:
       - Events: meaningful things that happened
       - Facts: stable truths stated explicitly
       - Decisions: choices made or discussed
       - Preferences: signals about how the user thinks/works/lives
       - Open loops: unresolved questions or topics
     
     All extractions are candidates, not confirmed.
     → Events and Decisions: stored immediately as confirmed (user stated them)
     → Facts: stored as Observations first; promoted to Facts per Brain spec
     → Preferences: stored as Observations
     → Open loops: stored with domain + priority estimate
  
  2. Embed new items
     → Generate embeddings for all newly created memory items
     → Update embedding index
  
  3. Check for Thread evidence
     → Did this session add evidence to an active Thread?
     → Did this session create conditions for a new Thread? (run Thread detection)
  
  4. Update open loop registry
     → Were any existing open loops addressed? Mark resolved.
     → Were new open loops created? Add to registry.
  
  5. Update Space states
     → If session was Space-scoped: update that Space's recent events
     → If session surfaced domain signals: queue Space update
  
  6. Log: extraction summary (counts only, no content)
```

**Output:** New memory items, updated open loop registry, embedding updates, potential Thread changes.

**Failure handling:** If extraction LLM call fails → log the session for retry in next Nightly Run. No data is lost (conversation history is always stored).

---

### 4. Domain Data Ingestion Run
**When:** Immediately on domain data arrival (finance upload, health sync).
**Trigger:** Event-based.
**Purpose:** Process new domain data into signals and update relevant subsystems.

```
Steps:
  1. Parse and validate incoming data
     → Finance: parse CSV, deduplicate against existing transactions
     → Health: parse metrics, check for anomalies vs history
     → Calendar: parse events, check for commitment signals
  
  2. Generate domain signals
     → Finance: category totals, anomaly flags, spending delta vs last cycle
     → Health: workout streak/break, biomarker changes, trend direction
     → Calendar: upcoming commitments, conflicts, available focus time
  
  3. Route signals to Brain
     → Each signal: does it confirm an existing Observation? Create a new one?
     → Update Fact confidence based on new data
     → Check for Pattern triggers
  
  4. Run Reasoning Engine on new signals
     → Does anything warrant surfacing now?
     → Run Stillness check
     → If a Warning: surface immediately (Today update + Dialogue notification)
     → If a normal signal: queue for next Morning Run
  
  5. Update relevant Space
     → Recalculate Space's AIRCH Read
     → Update momentum signal
     → Add to recent events
  
  6. Log: data ingested, signals generated, any surfacing decisions
```

**Output:** Updated domain data, new signals in Brain, potentially updated Space, potentially a Warning surfaced immediately.

---

### 5. Space Reassessment Run
**When:** Every 14 days per Space (if no data-triggered update has occurred).
**Trigger:** Time-based, per-Space.
**Purpose:** Keep AIRCH's Read fresh even when no new data has arrived.

```
Steps:
  1. Collect all active memory for this Space's domain
  2. Check: has anything changed in 14 days?
     → If yes: recalculate AIRCH's Read (Haiku-level call)
     → If no: extend current Read's validity without regenerating
  3. Check dormancy threshold
     → Has this Space been inactive for 60 days?
     → If yes: flag for dormancy question in Dialogue
```

**Output:** Updated Space Read or dormancy flag.

---

## Scheduler Priorities

When multiple jobs are scheduled simultaneously (rare but possible):

```
Priority order:
1. Domain Data Ingestion  — event-driven, time-sensitive
2. Post-Session Run       — user just finished a conversation
3. Morning Run            — day is starting
4. Space Reassessment     — low urgency
5. Nightly Run            — background, tolerance for delay
```

No two heavy jobs (Morning Run, Nightly Run, Post-Session Run) should overlap. If a conflict occurs: queue the lower-priority job.

---

## Logging

Every Scheduler job logs:

```
job_type, start_time, end_time, duration_ms
items_processed, items_created, items_updated, items_archived
errors (count + type, no content)
llm_calls_made, tokens_used (approximate)
```

Logs are local only. No content is logged — only counts and metadata.

Logs are retained for 30 days then deleted automatically.

---

## What the Scheduler Does Not Do

- Make decisions about what to surface (that is the Reasoning Engine)
- Generate user-facing content (except Morning Brief, which is a special case)
- Delete memory (it archives; the user deletes through Mirror)
- Run while the user is actively in a Dialogue session (waits for session end)
- Send notifications (a surfacing decision by the Reasoning Engine triggers notifications; the Scheduler only feeds that decision)
