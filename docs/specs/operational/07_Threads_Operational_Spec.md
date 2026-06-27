# Threads — Operational Spec

> How AIRCH detects, tracks, and closes cross-domain stories.

---

## Role

Threads are cross-domain stories.

Spaces are vertical. Threads are horizontal — they connect events, patterns, and decisions across multiple Spaces into a single narrative.

A Thread is not a tag. It is not a category. It is a story that AIRCH has noticed running through the user's life.

---

## Thread Detection

AIRCH detects a Thread when:

**Condition A — Cross-domain temporal overlap:**
- A Pattern or high-confidence Belief exists in Domain X
- A Pattern or high-confidence Belief exists in Domain Y
- Both are active within the same 14-day window
- The two patterns have a plausible causal or correlative relationship

**Condition B — Topic recurrence across Spaces:**
- The same topic, person, or decision appears in Dialogue in relation to ≥ 2 distinct Spaces
- Occurs within a 30-day window
- Appears in ≥ 3 distinct interactions

**Condition C — Decision with multi-domain consequences:**
- A Decision is tracked in one domain
- AIRCH detects ripple effects in ≥ 1 other domain within 21 days

Meeting any one condition triggers Thread evaluation. Thread is created only if AIRCH can name it coherently — if it cannot produce a clear one-sentence description of the story, the Thread is not created.

---

## Thread Properties

Each Thread has:

- **Name** — AIRCH-generated, short, descriptive (e.g. "The investment question", "Rebuilding the routine")
- **Domains involved** — list of Spaces this Thread touches
- **Status** — Emerging / Active / Resolving / Closed
- **Confidence** — how certain AIRCH is that this is a real story (not noise)
- **Core tension** — one sentence describing what is unresolved
- **Evidence** — list of supporting Facts, Patterns, and Events
- **Created date** — when AIRCH first detected it
- **Last activity** — last time new evidence was added

---

## Thread Lifecycle

### Emerging
AIRCH has detected the conditions but confidence is low (< 0.5).

Behavior:
- Thread is created internally but not named to the user
- AIRCH may ask a question in Dialogue to test the hypothesis
- If user response supports the Thread → confidence rises
- If user response contradicts it → Thread is discarded

### Active
Confidence ≥ 0.5. Thread is real enough to surface.

Behavior:
- Thread is referenced in Today if it touches the Primary Thought
- Thread is surfaced in relevant Spaces (as Connected Threads)
- AIRCH may reference it in Dialogue when relevant
- New evidence continuously updates the Thread

### Resolving
The underlying tension shows signs of resolving.

Signs of resolution:
- A key Decision has been made
- The cross-domain correlation weakens
- User explicitly addresses the Thread's core tension
- One of the source domains shows significant change

Behavior:
- AIRCH notes the resolution: "The [Thread name] seems to be resolving."
- Monitoring continues for 14 days to confirm
- If tension re-emerges → Thread returns to Active

### Closed
Thread has resolved. Moved to Continuity as part of the life arc.

The story is preserved — not deleted. It becomes part of the historical record that Continuity draws from.

---

## How Threads Surface

Threads are never a screen or a list.

They appear as connections within existing surfaces:

**In Today:**
If the Primary Thought is part of an active Thread: "Part of a longer story." — one line, tap to explore.

**In Dialogue:**
AIRCH names the Thread when relevant: "This connects to something I have been noticing in Finance as well. The [Thread name] has been running for about three weeks."

**In Spaces:**
Connected Threads appear as a quiet indicator within the Space. Thread name only. Tap to see what connects this Space to the story.

**In Continuity:**
Closed Threads appear as narrative arcs — stories that ran through the user's life and how they resolved.

---

## Thread Naming

AIRCH names Threads, not the user.

Naming rules:
- Short: 2–5 words
- Descriptive of the story, not the domain
- Honest about uncertainty in early stages
- Named after the tension, not the outcome

Good names:
- "The investment question"
- "Rebuilding the routine"
- "What to do with SkipMar"
- "The Mihkel negotiation"
- "October pattern returning"

Bad names:
- "Finance and Health correlation" (domain label, not a story)
- "Stress eating pattern" (judgment, not a name)
- "Thread #4" (meaningless)

---

## User Interaction with Threads

The user can:
- **Name / rename** a Thread: "Call this 'The funding decision'"
- **Dismiss** a Thread: "This isn't a story, it's coincidence." → Thread discarded; confidence penalty on similar future detections
- **Confirm** a Thread: "Yes, these are definitely connected." → confidence boost
- **Mark resolved** manually: "This is done." → Thread moves to Closed

The user cannot create Threads manually. Threads are always AIRCH-detected stories, not user-defined tags.

---

## Edge Cases

**Thread detected but user has not mentioned both domains:**
AIRCH can detect a Thread from data alone (e.g. spending spike + project stall from data sources). The Thread is valid. It can be surfaced without requiring the user to have mentioned both domains explicitly.

**Thread that is really just correlation:**
Not every cross-domain overlap is a story. AIRCH must be able to articulate the core tension. If it cannot — no Thread. Correlation without meaning is noise.

**Thread involving sensitive domains:**
If a Thread involves health conditions or personal relationships, AIRCH surfaces it only in Dialogue — not in Today or Spaces. The user's explicit engagement is required before the Thread becomes Active.

**Multiple active Threads simultaneously:**
Normal. Each Thread is independent. Today surfaces at most one Thread reference per day (the one most relevant to the Primary Thought). Others remain available in Dialogue and Spaces.

**Thread that grows to encompass everything:**
If a Thread starts absorbing too many domains and too much evidence, AIRCH evaluates whether it is actually multiple Threads. A Thread that explains everything explains nothing.
