# Mirror — Operational Spec

> How AIRCH builds, maintains, and exposes its understanding of the user.

---

## Role

Mirror is what AIRCH believes about the user — made visible and editable.

It is the trust contract. Without Mirror, memory is surveillance. With Mirror, the user can verify and correct what the system believes.

---

## Mirror Content

Mirror contains all memory above confidence 0.3, organized by type:

### Facts
Confirmed observations about the user. Stable truths.
- Examples: "Works best between 22:00 and 01:00", "Does not want external investment", "Allergic to X"
- Confidence shown: high-confidence Facts are stated plainly; lower-confidence ones flagged as "likely" or "appears to be"

### Patterns
Recurring structures AIRCH has detected.
- Examples: "Restaurant spending rises during high-stress project periods", "Project activity drops when more than 3 are active simultaneously"
- Always framed as patterns, never as facts: "This appears to be a pattern..."
- Confidence shown explicitly

### Habits
Recurring behaviors — positive or negative.
- Examples: "Works in long uninterrupted sessions", "Avoids financial review for weeks at a time"
- Derived from Patterns; never stated as personality traits

### Preferences
How the user likes to work, communicate, and live.
- Examples: "Prefers direct feedback", "Works better alone than in teams", "Avoids confrontation in negotiations"
- Derived from behavior, confirmed through Dialogue

### Values
What actually matters to the user, inferred from choices over time.
- Examples: "Autonomy over income", "Depth over breadth", "Long-term over short-term"
- The deepest layer. Shown with highest confidence threshold (≥ 0.7 to appear in Mirror).

### Direction
Who the user is trying to become.
- Derived from Values, Decisions, and long-term Patterns
- May be shown as a statement or as a heading: "Based on your choices over the last year..."
- Editable — the user can refine or correct AIRCH's interpretation

### Gaps
What AIRCH does not know or is uncertain about.
- Shown honestly: "I have limited data about [domain]. My understanding here may be incomplete."
- Not a list of missing data — a qualitative acknowledgment of uncertainty in key areas

---

## Mirror Display Rules

### What appears
- All memory items with confidence ≥ 0.3
- Organized by type, not by date
- Confidence level shown alongside each item (not as a number — as framing language)

### What does not appear
- Observations (too raw)
- Items with confidence < 0.3 (held silently)
- Closed Threads (shown in Continuity instead)
- Raw domain data (transactions, workout logs) — these are in Spaces, not Mirror

### Framing by confidence level

```
≥ 0.85   → stated plainly, no qualifier
0.70–0.85 → "Based on the last [period]..."
0.55–0.70 → "This appears to be..."
0.40–0.55 → "I have been noticing..."
0.30–0.40 → shown with "Uncertain" label; user prompted to confirm or dismiss
```

---

## User Actions in Mirror

For every item in Mirror, the user can:

### Confirm
"Yes, this is true."
Effect: confidence +0.20; item promoted toward higher confidence tier.

### Mark outdated
"This was true, but not anymore."
Effect: confidence −0.30; item flagged as historical; no longer used in active outputs. Retained for Continuity.

### Correct
"This is wrong / incomplete. Here is what is actually true."
Effect: existing item updated or replaced; old version archived with note "user corrected"; Brain notes the correction for similar future inferences.

### Add context
"This is true, but here is what you are missing."
Effect: annotation stored alongside the item; used to qualify future outputs based on this belief.

### Delete
"Remove this entirely."
Effect: item removed from Mirror; Brain applies confidence penalty to similar future inferences; deletion is logged (AIRCH learns what kinds of things to not infer).

### Dismiss (for uncertain items)
"I don't think this is accurate."
Effect: confidence drops to 0; item archived; similar signals get −0.15 penalty.

---

## Mirror Updates

Mirror is updated silently and continuously. The user is not notified of every update.

### What triggers a Mirror update
- New high-confidence Fact extracted from Dialogue
- Pattern confidence crosses a threshold (0.3 → visible; 0.7 → stated plainly)
- User action in Mirror (confirm, correct, delete)
- Value-level inference formed
- Direction interpretation updated

### What does not trigger a Mirror update
- Low-confidence observations
- Routine data with no inference
- Information the user explicitly asked AIRCH to ignore

### AIRCH's notification policy for Mirror
Mirror is a pull surface. AIRCH does not push Mirror updates.

Exception: if AIRCH has updated a high-confidence item based on new contradicting evidence, it may note this in Dialogue: "I have revised what I understand about [topic]. You may want to check Mirror." — used sparingly, once per significant revision.

---

## Sensitive Content in Mirror

Some Facts and Patterns involve sensitive domains: health conditions, relationships, finances, identity.

Rules for sensitive content:
- Stored at any confidence level (sensitivity does not prevent storage)
- Displayed in Mirror only at confidence ≥ 0.5
- Referenced in outputs only if the user has raised the topic in Dialogue first
- Never quoted back verbatim — paraphrased and contextualized
- User can mark any item as "private" — it remains in Mirror but is never referenced in outputs or Dialogue

---

## Mirror and Trust

The existence of Mirror changes the entire relationship between the user and AIRCH.

Without Mirror: "AIRCH knows things about me. I don't know what."
With Mirror: "AIRCH knows things about me. I can see exactly what, and I can correct it."

This is why Mirror must be honest about uncertainty. An overconfident Mirror that shows wrong things as facts destroys trust faster than a cautious Mirror that shows uncertainty.

Mirror should sometimes feel incomplete. That is accurate. AIRCH does not fully understand anyone.

---

## Edge Cases

**User deletes most of Mirror:**
AIRCH does not rebuild the same inferences immediately. It starts re-learning from current behavior. The deletion is a signal that the previous understanding was significantly wrong.

**User adds a Fact that AIRCH has not inferred:**
"I want you to know that I have two kids."
This is stored as a user-added Fact, confidence 1.0 (user-stated). It does not need to be inferred. AIRCH uses it going forward.

**Conflicting user statements:**
User confirmed X in Mirror, then said the opposite in Dialogue.
AIRCH flags the conflict: "You confirmed [X] in Mirror, but you just said [Y]. Which is still true?" — one direct question, resolved in Dialogue. Mirror updated based on the resolution.

**User asks to see everything AIRCH knows:**
Mirror is already that. If the user is asking something more specific ("what do you know about my health?"), AIRCH filters Mirror to that domain and presents it directly.
