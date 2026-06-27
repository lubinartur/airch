# Stillness — Operational Spec

> How AIRCH decides not to speak.

---

## Role

Stillness is not a feature. It is not a module. It is not a setting.

Stillness is the operating principle that judges every output in the system.

Its question is simple: **should AIRCH say anything at all?**

Most of the time, the answer is no.

---

## The Stillness Check

Every potential output — Morning Brief, Recommendation, Question, Warning, Space update, proactive Dialogue opening — runs through the Stillness Check before it is produced.

The Stillness Check is a gate. Any single failure = silence.

```
Gate 1: Confidence threshold
  Is this above the minimum confidence for this output type?
  Below threshold → SILENCE

Gate 2: Cooldown
  Was a similar output produced within the cooldown period?
  In cooldown → SILENCE

Gate 3: User already knows
  Would a thoughtful person who knows this user assume they already know this?
  Yes → SILENCE

Gate 4: Active conversation
  Is the user currently in an active Dialogue session about this topic?
  Yes → withhold; surface naturally within the conversation instead

Gate 5: User mute
  Has the user muted this signal type or domain?
  Yes → SILENCE

Gate 6: Priority conflict
  Is there a higher-priority signal that should go first?
  Yes → queue this signal; surface the higher-priority one instead

Gate 7: Temporal validity
  Is this signal still relevant right now?
  Stale or no longer actionable → SILENCE

Gate 8: Actionability
  Does this output have an actionable dimension?
  (Something the user can do, decide, or understand differently)
  No → SILENCE
```

All 8 gates must pass. One failure = silence.

---

## Gate 3 in Detail — The Hardest Gate

Gate 3 is the most important and the hardest to implement correctly.

The test: **would a thoughtful person who knows this user say this right now — or would they assume the user already knows?**

### Things the user almost certainly already knows
- That they haven't worked out in several days (they know their own body)
- That a project is stalled (they are working on it or avoiding it — either way, they know)
- That they spent a lot on restaurants last week (they were there)

### Things the user might not have noticed
- That the restaurant spending correlates with the project stall — for the third time
- That their energy pattern in workouts degrades consistently in weeks when a specific type of meeting is frequent
- That a subscription they forgot about has been billing for 47 days

The difference: AIRCH adds value when it **connects** things the user knows separately into a pattern they have not seen. It adds no value by reporting things back to the user that they experienced directly.

Gate 3 asks: is AIRCH adding a connection — or just echoing?

If echoing → SILENCE.

---

## Cooldown Registry

The system maintains a cooldown registry per signal type and per domain.

### Default cooldown periods

```
Same signal type:              7 days
Same domain:                   3 days
Same topic (specific):         5 days
Cross-domain Pattern:          10 days
Warning (after resolution):    14 days
Question (same open loop):     3 days
```

### Cooldown exceptions
- Warnings with escalating urgency bypass cooldown
- User explicitly asks AIRCH about a topic → relevant signals surface regardless of cooldown
- User opens a Space → domain-specific signals may surface within that Space regardless of general cooldown (user is actively seeking that domain context)

---

## Frequency Limits

Beyond cooldown, hard frequency limits apply:

```
Proactive Today content:       1 per day
Proactive Dialogue opening:    1 per day
Proactive Space update:        1 per Space per 14 days (except data-triggered)
Warnings:                      no limit (but threshold is strict)
Questions in Dialogue:         1 proactive per day
```

These limits apply to proactive outputs only. User-triggered responses (the user asks something) have no frequency limit.

---

## Silence as a Signal

When AIRCH is quiet, the silence carries meaning:

- Today is empty → nothing requires your attention right now
- Dialogue does not open proactively → no open loops or new Beliefs worth surfacing
- A Space shows the same Read as last time → nothing meaningful has changed

Users who understand AIRCH learn to read silence correctly. A system that fills every moment trains users to ignore it. A system that speaks rarely trains users to listen.

Silence is calibrated, not random. It is a decision, not an absence.

---

## What Stillness Prevents

### Observation inflation
The most common failure mode. AIRCH has something to say every day. Users start ignoring it within two weeks.

Prevention: frequency limits + cooldown registry + Gate 3.

### Echoing
AIRCH reports back to the user what the user already knows. Creates the feeling of surveillance, not intelligence.

Prevention: Gate 3.

### Low-confidence guessing
AIRCH makes claims based on weak evidence because it feels the need to say something.

Prevention: Gate 1 (confidence threshold).

### Timing failure
A correct observation delivered at the wrong moment. Right after a decision is made. In the middle of a focused work session. At 23:00.

Prevention: Gate 7 (temporal validity) + time-of-day awareness in Reasoning Engine.

### Topic flooding
The same underlying concern expressed in multiple ways across Today, Dialogue, and a Space on the same day.

Prevention: cross-surface deduplication. Once a topic surfaces in one place, it does not surface in another on the same day.

---

## Stillness and Trust

Trust is built through restraint.

Every time AIRCH says something unnecessary, it costs a small amount of trust. This cost is not immediately visible — but it compounds.

A user who has dismissed 20 irrelevant AIRCH outputs will unconsciously begin dismissing everything, including the important things.

A user who has seen AIRCH be quiet for days and then say something — and found that thing to be exactly right — will pay full attention every time.

The goal is not to minimize outputs. The goal is to maximize the value of every output.

Stillness is how that happens.

---

## Monitoring Stillness Quality

The system tracks:

- **Output engagement rate** — what % of proactive outputs are engaged with
- **Dismissal rate** — what % are explicitly dismissed
- **Mute rate** — what signal types have been muted by the user
- **"Already knew this" signals** — user responses that indicate the output was obvious

If engagement rate falls below 40% over 30 days → Reasoning Engine automatically raises confidence thresholds by 0.05 across all output types.

If mute rate for a signal type exceeds 30% → that signal type is suspended for 30 days and re-evaluated.

Stillness quality is measurable. It should be measured.
