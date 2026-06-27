# Reasoning Engine

## Role

The Reasoning Engine is the decision layer above the Brain.

The Brain forms beliefs. The Reasoning Engine decides what to do with them.

Its core question at any moment:

> Is there something worth surfacing right now — and if so, what form should it take?

---

## Inputs

The Reasoning Engine reads from all available domains:

- **Calendar** — commitments, time available, upcoming events
- **Finance** — spending patterns, obligations, balance, anomalies
- **Health** — workouts, recovery, biomarkers, energy signals
- **Projects** — momentum, stalled work, open decisions
- **Memory** — patterns, facts, values, direction
- **Dialogue** — open loops, unresolved questions, recent conversations
- **Threads** — cross-domain stories currently active
- **User state** — inferred from recent behavior and context

---

## Outputs

The Reasoning Engine produces one of:

**Morning Brief**
One primary thought for the day. Drawn from the highest-priority signal across all domains.

**Recommendation**
A specific, concrete suggestion — with reasoning. Only produced when confidence is sufficient.

**Question**
When AIRCH detects something important but lacks enough data to recommend. A question is more honest than a low-confidence recommendation.

**Warning**
An urgent signal. Something is off course and the cost of ignoring it is rising.

**Silence**
The most common output. Nothing worth surfacing. AIRCH waits.

---

## Priority Order

When multiple signals compete, the Reasoning Engine resolves by:

1. **Safety** — Is anything at risk? Health, finance, a critical commitment?
2. **Long-term direction** — Does this connect to who the user is trying to become?
3. **Commitments** — What has the user said they would do?
4. **Momentum** — What is gaining or losing energy?
5. **Convenience** — What would be nice to know?

Lower-priority signals are withheld, not discarded. They wait for a better moment.

---

## Frequency Rules

- At most one proactive surface per day on the main screen.
- Warnings can break this limit — but only for genuine urgency.
- Questions are surfaced in Dialogue, not on Today, unless they are the day's primary thought.
- The Reasoning Engine never fills silence with noise.

---

## The Stillness Check

Before any output, the Reasoning Engine runs a final check:

> Would a thoughtful person who knows this user well say this right now?

If the answer is uncertain — withhold.

Stillness is not the absence of good ideas. It is the judgment to deliver them at the right moment.

---

## Learning Loop

After every output, the Reasoning Engine tracks:

- Did the user engage with it?
- Did the user dismiss it?
- Did the user act on it?
- Did the user correct it?

This feedback updates Memory confidence and refines future outputs.

A Reasoning Engine that never learns from response is not reasoning. It is broadcasting.
