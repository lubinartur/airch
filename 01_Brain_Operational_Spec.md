# Brain — Operational Spec

> How AIRCH processes information into judgment.
> This is the core pipeline. All other specs depend on it.

---

## Role

The Brain is the reasoning layer of AIRCH. It is never visible to the user.

Its job is not to process data. Its job is to form judgment — and decide when that judgment is worth acting on.

---

## Processing Pipeline

Every piece of input enters the Brain and moves through stages. Each stage has an entry condition. Nothing skips a stage.

```
Input
  ↓
Observation       — something was detected
  ↓ [confirmation threshold]
Fact              — it is confirmed and stored
  ↓ [recurrence threshold]
Pattern           — it repeats across time or domains
  ↓ [significance threshold]
Belief            — AIRCH forms a view about what it means
  ↓ [confidence threshold]
Output            — recommendation / question / warning / silence
```

---

## Stage Definitions

### Observation

**What it is:** A raw signal. Something happened, was said, or was detected.

**Sources:**
- User message in Dialogue
- Data from a connected domain (finance, health, calendar)
- Absence of expected behavior (no workout in N days, no upload in N days)
- Explicit user statement

**Properties:**
- Has a source, timestamp, and domain
- Has no confidence score yet
- Is not stored permanently — it waits for confirmation
- Expires if not confirmed within 7 days

**Entry condition:** Any input qualifies as an Observation. No threshold.

**Exit condition:** Confirmed by recurrence, user acknowledgment, or cross-domain corroboration.

---

### Fact

**What it is:** A confirmed observation. Stored permanently until explicitly removed.

**Properties:**
- Has source, domain, timestamp, and confidence score (initial: 0.5)
- Confidence increases with each supporting observation
- Confidence decreases if user rejects a related output
- Is editable through Mirror

**Entry conditions (any one of):**
- Same observation occurs ≥ 2 times within 30 days
- User explicitly confirms it in Dialogue
- Cross-domain corroboration (same signal appears in 2+ domains simultaneously)

**Exit conditions:**
- User deletes it in Mirror
- Contradicted by strong evidence over time (confidence drops below 0.1)
- User marks it as outdated

**Edge cases:**
- A single high-importance event (e.g. "I quit my job") can become a Fact immediately — no recurrence required. Importance is determined by the presence of strong explicit language and domain significance.
- Facts about preferences are never inferred from a single data point. Preferences require ≥ 3 observations.

---

### Pattern

**What it is:** A recurring structure detected across Facts over time or across domains.

**Properties:**
- Links ≥ 2 Facts
- Has a domain scope (single-domain or cross-domain)
- Has a time scope (weekly, monthly, situational)
- Has a confidence score (initial: 0.4)
- Is visible in Mirror

**Entry conditions (all required):**
- ≥ 3 supporting Facts
- Recurrence across ≥ 2 distinct time periods
- Minimum time span of 14 days between first and last supporting Fact

**Cross-domain Pattern entry conditions (stricter):**
- ≥ 2 supporting Facts per domain involved
- Temporal overlap: the Facts from different domains must overlap within a 7-day window
- Minimum 2 occurrences of the overlap

**Exit conditions:**
- User rejects it in Mirror (confidence drops to 0, pattern is archived)
- No supporting Facts for 90 days (pattern becomes dormant, not deleted)
- User explicitly marks it resolved

**Edge cases:**
- A Pattern with confidence < 0.4 is never surfaced directly. It may generate a Question in Dialogue.
- A dormant Pattern can be reactivated if new supporting Facts appear.

---

### Belief

**What it is:** AIRCH's interpretation of what a Pattern means — in the context of this user's Direction and current state.

**Properties:**
- Is derived from one or more Patterns
- Incorporates Direction (who the user is becoming) as context
- Has a confidence score inherited and adjusted from its source Patterns
- Can be held silently or surfaced as output
- Is visible in Mirror at high confidence

**Entry conditions (all required):**
- At least one Pattern with confidence ≥ 0.5
- Belief is interpretable in terms of user's current state or Direction
- Belief has an actionable dimension (something can be done, asked, or understood)

**Exit conditions:**
- Source Pattern is rejected or becomes dormant
- User explicitly contradicts the Belief in Dialogue
- Evidence accumulates against it over time

**Edge cases:**
- A Belief without an actionable dimension is held silently. It is never surfaced.
- Contradictory Beliefs (two valid but opposing interpretations) are both held. Neither is surfaced until one gains significantly higher confidence.

---

## Output Decision

When a Belief reaches sufficient confidence, the Brain evaluates what output to produce.

### Output types (in order of restraint)

**Silence** — most common. No output. The Brain continues observing.

**Question** — when the Brain has a Pattern or emerging Belief but insufficient confidence to recommend. The question surfaces in Dialogue, not in Today.

**Observation** — a named pattern surfaced to the user. Framed honestly as a pattern, not a fact. Appears in Today or Dialogue depending on urgency.

**Recommendation** — a concrete suggestion. Requires highest confidence. Always includes the reasoning behind it.

**Warning** — urgent signal. Something is off course and the cost of ignoring it is rising. Can interrupt normal surfacing priority.

### Output selection rules

```
Confidence < 0.4           → Silence
Confidence 0.4–0.55        → Question only
Confidence 0.55–0.70       → Observation (with honest uncertainty language)
Confidence 0.70–0.85       → Recommendation (with reasoning)
Confidence > 0.85          → Strong recommendation or Warning if time-sensitive
```

---

## Confidence Scoring

Confidence is a float between 0.0 and 1.0.

### Factors that increase confidence
- Each additional supporting observation: +0.08
- User confirms related output: +0.15
- Cross-domain corroboration: +0.12
- Recurrence across 3+ distinct time periods: +0.10
- Matches a known user Pattern: +0.08

### Factors that decrease confidence
- User rejects related output: −0.20
- Contradicting observation appears: −0.10
- Data is older than 30 days: −0.05 per week beyond 30 days
- Single-domain only (for cross-domain Beliefs): −0.10

### Confidence floors
- Facts: minimum 0.1 (never fully discarded until user deletes)
- Patterns: minimum 0.0 (can become dormant)
- Beliefs: minimum 0.0 (can be silently archived)

---

## When the Brain Must Stay Silent

The Brain is obligated to produce no output when any of the following are true:

1. Confidence is below 0.4
2. A similar output was produced within the last 7 days (cooldown)
3. The output has no actionable dimension
4. The user has muted this signal type
5. The user is in an active Dialogue about the same topic (let the conversation resolve it)
6. The Belief is contradicted by a more recent higher-confidence Belief
7. The output would be something the user already knows without AIRCH

Rule 7 is the hardest to implement and the most important. The test: would a thoughtful person who knows this user say this right now, or would they assume the user already knows?

---

## Decision Loop

```
Observe
  → Does this confirm an existing Observation? → update Fact confidence
  → Does this contradict an existing Fact? → decrease confidence
  → Does this create a new Observation? → store temporarily

Remember
  → Connect to existing Facts, Patterns, Beliefs
  → Update Mirror if significant

Understand
  → Does a new Pattern emerge?
  → Does an existing Belief gain or lose confidence?
  → Does Direction provide context that changes interpretation?

Output decision
  → Run output selection rules
  → Run Stillness check (see 09_Stillness_Operational_Spec.md)
  → Produce output or stay silent

Learn
  → Track user response to output
  → Update confidence accordingly
  → If rejected: review source Pattern and Belief
```

---

## What the Brain Is Not Responsible For

- Deciding where output appears (Today vs Dialogue vs Space) — that is the Reasoning Engine
- Storing raw data (transactions, workouts) — that is domain-specific pipelines
- Generating the text of output — that is the Voice layer
- Deciding timing of output — that is the Reasoning Engine

The Brain produces: a Belief with a confidence score and an output type recommendation.
Everything else is downstream.
