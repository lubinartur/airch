# Reasoning Engine — Operational Spec

> How AIRCH decides what to surface, when, and in what form.

---

## Role

The Brain forms Beliefs. The Reasoning Engine decides what to do with them.

It runs continuously in the background. Its job is not to generate content — it is to make one decision:

> Is there something worth surfacing right now — and if so, what form and where?

---

## Inputs

The Reasoning Engine reads from:

- **Brain output** — Beliefs with confidence scores and suggested output types
- **Domain signals** — raw signals from Finance, Health, Projects, Calendar that have not yet been processed into Beliefs
- **Active Threads** — cross-domain stories currently in motion
- **Open loops** — unresolved questions from Dialogue
- **User state** — inferred from recent behavior, last interaction time, current page
- **Direction** — who the user is trying to become (used for priority weighting)
- **Cooldown registry** — what has been surfaced recently and when

---

## Priority Order

When multiple signals compete, the Reasoning Engine resolves by:

```
1. Safety         — Is anything at risk? Health emergency, financial crisis, 
                    critical missed commitment?
2. Direction      — Does this connect to who the user is trying to become?
3. Commitments    — What has the user explicitly said they would do?
4. Momentum       — What is gaining or losing energy right now?
5. Convenience    — What would be useful but not urgent?
```

Lower-priority signals are not discarded. They are queued. They wait for a better moment or a higher-confidence threshold.

---

## Output Types

### Morning Brief
**When:** Once per day, on first open of Today.
**Content:** One primary thought. The most important signal across all domains, compressed into 2–3 sentences.
**Source:** Highest-priority active Belief, weighted by Direction.
**Constraint:** Only one. Never a list. If no signal is strong enough — Today opens quietly.

### Recommendation
**When:** Belief confidence ≥ 0.70 and signal has actionable dimension.
**Content:** Specific suggestion with visible reasoning.
**Constraint:** Maximum one recommendation per day in Today. Additional recommendations surface in Dialogue only, triggered by user conversation.

### Question
**When:** Brain has a Pattern or Belief at confidence 0.4–0.55 that needs user input to resolve.
**Content:** A single focused question. Not a clarification request — a question that moves thinking forward.
**Where:** Always in Dialogue. Never in Today unless it is the day's primary signal.
**Constraint:** Maximum one proactive question per day. If the user is already in an active Dialogue, the question waits or surfaces naturally within the conversation.

### Warning
**When:** Urgent signal. Time-sensitive. Cost of ignoring is rising.
**Content:** Direct statement of what is happening and why it matters now.
**Where:** Today, regardless of other content. Warnings can interrupt the normal priority queue.
**Constraint:** Warnings are rare. A signal qualifies as a Warning only if: (a) it is time-sensitive within 48 hours, and (b) the user is unlikely to discover it without AIRCH.

### Silence
**When:** Most of the time.
**Content:** Nothing.
**Effect:** Today opens quietly. Dialogue waits for the user.
**This is the correct output in most cases.**

---

## Surfacing Rules

### Frequency limits
```
Morning Brief:       1 per day maximum
Proactive question:  1 per day maximum
Recommendation:      1 per day in Today; unlimited in Dialogue (user-triggered)
Warning:             No frequency limit, but threshold is strict
Silence:             No limit (encouraged)
```

### Cooldown
After any proactive output, a cooldown applies per signal type:

```
Same signal type:    7 days
Same domain:         3 days
Same topic:          5 days
Warning (resolved):  14 days
```

Cooldown is bypassed only for Warnings where new evidence escalates urgency.

### Where outputs appear

```
Today:      Morning Brief, Warnings, top Recommendation (1 max)
Dialogue:   Questions, Recommendations (user-triggered), Observations surfaced mid-conversation
Spaces:     Domain-specific signals when user is in that Space
Mirror:     Updated silently; user pulls, not pushed
Continuity: Updated silently; not a push surface
```

AIRCH never pushes to Mirror or Continuity. The user goes there; AIRCH does not interrupt with updates.

---

## The Stillness Check

Before any output is produced, the Reasoning Engine runs a final check:

```
1. Is this above the confidence threshold for its output type?
2. Is this within cooldown? → withhold
3. Does the user already know this without AIRCH? → withhold
4. Is the user currently in an active Dialogue about this topic? → withhold; surface naturally in conversation
5. Has the user muted this signal type? → withhold
6. Is there a higher-priority signal that should go first? → queue this one
7. Is this the right moment (time of day, user state)? → consider timing
```

If all checks pass → produce output.
If any check fails → silence.

The Stillness check is not a soft filter. It is a gate. One failure = silence.

---

## Timing

The Reasoning Engine runs:

- **On app open** — evaluate current state, produce Morning Brief if warranted
- **After Dialogue session** — extract new information, update Memory, check for new signals
- **On domain data ingestion** (new upload, new health data) — evaluate immediately
- **Background scan** — once per day, independent of user activity

### Time-of-day sensitivity
The Reasoning Engine is aware of when it is running:

- Early morning (06:00–10:00): eligible for Morning Brief
- Late night (22:00–07:00): no proactive output unless Warning
- Anytime: Warnings can fire; Silence is always available

---

## Signal Routing

How a signal flows through the system to output:

```
New input arrives
  ↓
Brain processes → produces Belief + confidence + suggested output type
  ↓
Reasoning Engine receives Belief
  ↓
Priority check → where does this rank against active signals?
  ↓
Stillness check → does this pass all gates?
  ↓
Output type confirmed → Morning Brief / Recommendation / Question / Warning / Silence
  ↓
Routing → which surface? Today / Dialogue / Space
  ↓
Voice layer → generates actual text (see 07_Voice.md)
  ↓
Delivered or withheld
```

---

## Learning Loop

After every proactive output, the Reasoning Engine tracks response:

| User action | Effect |
|---|---|
| Engages (opens, responds, acts) | confidence +0.10 on source Belief |
| Dismisses without engaging | confidence −0.05; cooldown extended |
| Explicitly rejects ("this isn't relevant") | confidence −0.20; similar signals get penalty |
| No response (ignores) | neutral; logged but no immediate change |

After 10+ outputs of a type, the Reasoning Engine evaluates aggregate response rate. If engagement < 30%, the threshold for that output type is raised automatically.

---

## Edge Cases

**Two equally high-priority signals:**
Surface the one more connected to active Direction. If equal, surface the one with higher recency. Never surface both on the same day in Today — queue the second for tomorrow.

**Signal becomes irrelevant by the time it surfaces:**
E.g. a recommendation about an upcoming meeting — but the meeting already happened. Reasoning Engine checks temporal validity before surfacing. Stale signals are discarded, not surfaced late.

**User opens Today multiple times in a day:**
Morning Brief does not regenerate. Today shows the same primary thought. It does not refresh to generate new content on each open.

**No signal for multiple days:**
Today opens quietly. This is correct behavior. AIRCH does not manufacture content to fill silence. The Reasoning Engine waits for a genuine signal.

**Conflict between domains:**
E.g. Finance says "spend less" but Direction says "invest in the startup now." The Reasoning Engine does not resolve this internally — it surfaces it as a Question in Dialogue, where the user can reason through it.
