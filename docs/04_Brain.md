# AIRCH Brain

## What the Brain Does

The Brain is the reasoning layer of AIRCH.

It is not visible to the user. It runs continuously in the background, watching patterns, forming beliefs, and deciding when something is worth surfacing.

Its job is not to process data. Its job is to form judgment.

---

## Processing Pipeline

```
Observation          — something happened
↓
Fact                 — it is confirmed and stored
↓
Memory               — it is connected to other facts
↓
Pattern              — it is repeating across time or domains
↓
Belief               — AIRCH forms a view about what it means
↓
Recommendation       — only if confidence is sufficient
```

Each step requires evidence. The Brain does not skip steps.

A single observation never becomes a recommendation.

---

## Decision Loop

```
Observe → Remember → Understand → Recommend → Plan → Learn
```

**Observe:** What is happening across all inputs — finance, health, projects, conversations, calendar.

**Remember:** Connect this to what AIRCH already knows about the user.

**Understand:** What does this mean in the context of this person's patterns, direction, and current state?

**Recommend:** Surface an insight, ask a question, or stay silent. Never recommend without sufficient confidence.

**Plan:** If the user acts, help them think through the next step.

**Learn:** Update Memory and Mirror based on what happened and how the user responded.

---

## Confidence Threshold

AIRCH should recommend only when confidence is sufficient.

Otherwise it should ask — or remain silent.

Confidence is built from:
- Number of supporting observations
- Recency of data
- Consistency across domains
- Whether the user has confirmed similar patterns before
- Whether the user has rejected similar patterns before

Low confidence → ask a question, not make a claim.
No confidence → say nothing.

---

## What the Brain Is Not

The Brain is not an analytics engine.

It does not produce reports. It does not surface metrics. It does not summarize data.

It forms judgment — and only speaks when that judgment is worth hearing.

---

## Silence as Output

Silence is a valid and frequent output of the Brain.

Most of the time, AIRCH has nothing important enough to say.

The Brain's ability to withhold is as important as its ability to recommend.

A Brain that always produces output is a Brain that has stopped thinking.
