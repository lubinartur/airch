# Memory — Operational Spec

> How AIRCH stores, connects, and uses what it knows.

---

## Role

Memory is what separates AIRCH from a chatbot.

A chatbot responds to input. AIRCH knows the person behind the input.

Memory is not a log. It is a living model of understanding — built gradually, maintained actively, always in service of better judgment.

---

## Memory Types

### Observation
Raw signal. Has not yet been confirmed. Temporary.
- Lifespan: 7 days without confirmation → expires silently
- Not shown in Mirror
- Not used in outputs

### Fact
Confirmed observation. Permanent until explicitly removed.
- Confidence: 0.1 – 1.0
- Shown in Mirror
- Used in Pattern detection and output generation

### Event
A meaningful moment with context. Stored permanently.
- Has domain, date, significance level, and optional linked Fact
- Examples: "left a job", "launched a product", "ended a relationship", "made a financial decision"
- Shown in Continuity
- High-significance Events can become Facts immediately (see Brain spec)

### Decision
A choice the user made or explicitly avoided. High value for Pattern detection.
- Always linked to a domain
- Tracks outcome if follow-up data arrives
- Shown in Continuity and Mirror

### Preference
How the user likes to work, communicate, or live.
- Requires ≥ 3 supporting observations before storage
- Shown in Mirror
- Updated when contradicting behavior accumulates

### Habit
A recurring behavior — positive or negative.
- Derived from Pattern detection, not direct input
- Never declared from a single observation
- Shown in Mirror with honest framing ("this appears to be a habit")

### Pattern
A recurring structure across Facts. See Brain spec for entry conditions.
- Shown in Mirror
- Cross-domain Patterns flagged separately

### Value
What actually matters to the user — inferred from behavior, not stated.
- The deepest memory layer
- Built from confirmed Patterns and Decisions over time
- Rarely changes — requires contradicting evidence across 60+ days to revise
- Shown in Mirror under Direction
- Used by Reasoning Engine to prioritize competing signals

---

## Memory Lifecycle

```
Observation (temporary, 7 days)
  ↓ [confirmed]
Fact (permanent, confidence-weighted)
  ↓ [recurs across time]
Pattern (permanent until dormant)
  ↓ [interpreted in context of Direction]
Belief (held silently or surfaced)
  ↓ [confirmed over time]
Value (near-permanent, shapes everything)
```

Promotion between stages is never automatic below Fact level — it requires meeting explicit thresholds (see Brain spec).

Demotion is always possible: user rejection, contradicting evidence, or elapsed time reduce confidence and can move memory back down the stack.

---

## Memory Confidence

Every memory item above Observation has a confidence score (0.0 – 1.0).

Confidence determines:
- Whether the item is shown in Mirror (threshold: 0.3)
- Whether it is used in output generation (threshold: 0.5 for Facts, 0.55 for Patterns)
- How it is framed in Voice (language must match confidence level)

### Confidence language mapping

```
< 0.4     → not surfaced; held silently
0.4–0.55  → "I have been noticing..." / "It seems like..."
0.55–0.70 → "This appears to be a pattern..." / "Похоже что..."
0.70–0.85 → "Based on the last N months..." / "Судя по данным..."
> 0.85    → stated directly, no hedging
```

AIRCH never states a Pattern as a fact. It never drops hedging language for low-confidence items regardless of how they are presented internally.

---

## Memory and Mirror

All Facts, Patterns, Habits, Preferences, and Values above confidence 0.3 are visible in Mirror.

The user can take these actions on any memory item:

| Action | Effect |
|---|---|
| Confirm | confidence +0.20 |
| Mark outdated | confidence −0.30; item flagged for re-evaluation |
| Correct | existing item updated or replaced; old version archived |
| Add context | stored as annotation; adjusts how item is used in outputs |
| Delete | item removed; Brain notes the rejection; similar items get confidence penalty |

Mirror is the only place where the user can directly modify memory. All other modifications happen through Dialogue (where the user's response is interpreted and applied).

---

## Memory Sources

### From Dialogue
- Event extraction: meaningful events mentioned in conversation
- Fact extraction: stable truths stated explicitly ("I don't drink", "I work best at night")
- Decision tracking: choices discussed and their outcomes
- Preference signals: patterns in how the user talks about different topics

### From Domain Data
- Finance: spending patterns, anomalies, category shifts
- Health: workout frequency, biomarker trends, energy signals
- Calendar: commitment patterns, how the user allocates time
- Projects: momentum, stall patterns, completion rate

### From Behavior
- What the user opens and when
- What outputs they engage with vs dismiss
- What they return to in Dialogue
- What they never mention (absence as signal — used carefully)

### From Mirror interactions
- Explicit confirmations and corrections
- Deletions (strong negative signal)

---

## Memory Decay

Memory does not expire automatically. Confidence decays.

### Decay rules
- Observation: expires after 7 days without confirmation
- Fact: confidence −0.02 per month without supporting evidence (floor: 0.1)
- Pattern: becomes dormant after 90 days without supporting Facts; dormant patterns can be reactivated
- Preference: confidence −0.01 per month without supporting behavior (slow decay)
- Value: no automatic decay; only revised by contradicting evidence over 60+ days
- Event / Decision: permanent; no decay (these are historical records)

---

## Memory Retrieval

When the Brain evaluates a new input, it retrieves relevant memory by:

1. **Domain match** — find all memory in the same domain as the input
2. **Temporal proximity** — prioritize memory from the last 30 days
3. **Semantic relevance** — find memory that relates to the topic of the input (requires embedding search)
4. **Cross-domain lookup** — for MasterAgent: retrieve memory from adjacent domains that have shown correlation

Retrieval is always filtered by confidence threshold (minimum 0.3 for context, 0.5 for active use in output).

---

## What Memory Must Never Feel Like

Memory must never be announced mechanically.

Prohibited patterns:
- "According to my records, you mentioned X on [date]."
- "Based on stored data, your preference is Y."
- "I found 3 matching memories."

Correct pattern:
- Use what is known naturally and without announcement
- "We have been here before." not "You have said this twice."
- "This fits a pattern I have noticed." not "Pattern #4 is triggered."

The user should feel understood — not catalogued.

---

## Edge Cases

**Conflicting Facts:**
Two contradictory Facts can coexist. The higher-confidence one is used in output. The lower-confidence one is retained but not surfaced. If they remain in conflict for 30+ days, Mirror flags it for user resolution.

**Sensitive content:**
Facts about health conditions, relationships, or identity are stored but treated with extra restraint. They are used to inform outputs, never directly quoted back. They require explicit user confirmation before influencing high-confidence Beliefs.

**Memory about absence:**
"The user never mentions family" is an observation, not a Fact. Absence is a weak signal. It requires corroboration from other sources before being stored as anything above Observation level.

**Old memory and new behavior:**
If a user's behavior changes significantly (e.g. new job, moved city, relationship change), existing Facts and Patterns get a confidence penalty of −0.15 across affected domains. The system re-learns from current behavior rather than clinging to outdated models.
