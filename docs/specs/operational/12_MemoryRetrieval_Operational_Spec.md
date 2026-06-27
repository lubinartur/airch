# Memory Retrieval — Operational Spec

> How AIRCH finds the right facts at the right moment.

---

## Role

Memory Retrieval is the bridge between stored memory and the Context Builder.

Given a query (a user message, a scheduled job, a domain signal), Memory Retrieval finds the memory items most relevant to answering it well.

Its job is not to return everything that might be related. Its job is to return the 3–5 items that would most improve the LLM's response.

---

## Retrieval Types

Memory Retrieval uses three methods, run in parallel and merged by relevance score.

### 1. Semantic Retrieval (embedding-based)

Finds memory items semantically similar to the query.

```
Input:  query text (user message, or generated query string for scheduled jobs)
Method: embed query → cosine similarity against memory embedding index
Filter: confidence ≥ 0.30, not archived
Return: top 10 candidates by similarity score
```

Semantic retrieval is the primary method. It finds things that are related in meaning even if they do not share keywords.

### 2. Keyword / Structural Retrieval (fast filter)

Finds memory items that match explicit signals in the query.

```
Triggers:
  - Named entity in query (person, project, place) → retrieve Facts mentioning that entity
  - Domain keyword (finance, health, project name) → retrieve top Facts + Patterns from that domain
  - Temporal reference ("last time", "in February") → retrieve Events from that period
  - Decision reference ("what I decided about X") → retrieve Decisions related to X

Method: structured query against memory store (no embedding needed)
Return: all matching items, no limit (passed to merge step)
```

### 3. Recency Retrieval (always-on)

Returns the most recent significant memory items regardless of query content.

```
Return:
  - Last 3 Events with importance ≥ high
  - Last 2 confirmed Facts (any domain)
  - All open loops (regardless of query)

Rationale: recent context is almost always relevant. Open loops always need to be available.
```

---

## Merge and Rank

After the three retrieval methods run, their results are merged and ranked.

### Scoring

Each candidate memory item receives a composite score:

```
score = (semantic_similarity × 0.45)
      + (recency_weight × 0.25)
      + (confidence × 0.20)
      + (domain_match × 0.10)

semantic_similarity:  0.0–1.0 (from embedding cosine similarity; 0 if not in semantic results)
recency_weight:       1.0 if < 7 days, 0.8 if < 30 days, 0.5 if < 90 days, 0.2 if older
confidence:           memory item's confidence score (0.0–1.0)
domain_match:         1.0 if item domain matches query domain, 0.5 if cross-domain, 0 if no match
```

### Deduplication
If the same memory item appears in multiple retrieval methods: keep once, take the highest semantic_similarity score.

### Final selection
Top 5 items by composite score.

Hard filters applied after scoring (these override score):
- Exclude items with confidence < 0.30
- Exclude archived items
- Exclude items the user has deleted
- Exclude sensitive items (health conditions, relationship details) unless the user raised the topic in this session

---

## Retrieval for Specific Call Types

### Dialogue response
Query: the user's message text.
All three retrieval methods run.
Output: top 5 results → Layer 2 of context package.

### Morning Brief / Today assembly
Query: generated from current highest-priority Belief + domain signals.
Example generated query: "project stall stress restaurant spending pattern"
Semantic + structural retrieval run. Recency always included.
Output: top 3 results → Layer 2 of context package.

### Space update (AIRCH's Read)
Query: domain name + recent domain events.
Structural retrieval only (domain-filtered).
Recency retrieval domain-filtered.
Output: top 3 domain-specific results.

### Pattern detection (nightly)
Query: not a text query. Run structural retrieval for each domain.
Return all Facts and Events from last 30 days per domain.
Used by Brain to detect new Patterns, not for LLM context.

### Stillness fast check
No memory retrieval. Uses only the candidate output and the cooldown registry.

---

## Retrieval Freshness Rules

Some memory is excluded from retrieval based on age, regardless of relevance score:

```
Observations:      never retrieved (too raw)
Facts:             retrievable at any age, but recency_weight decays
Events:            retrievable at any age, recency_weight decays
Patterns:          retrievable at any age; dormant Patterns get recency_weight = 0.1
Habits:            retrievable at any age
Preferences:       retrievable at any age
Values:            always retrievable (no recency decay — Values are structural)
Decisions:         retrievable at any age; old Decisions useful for Continuity context
```

---

## What Good Retrieval Looks Like

Given the user message: "I keep putting off the investor call."

Good retrieval returns:
- Pattern: "User avoids direct confrontation in high-stakes conversations" (confidence 0.65)
- Decision: "Decided against external investment in March" (3 months ago)
- Fact: "User works through avoidance by talking through the fear first" (confidence 0.72)
- Event: "Delayed the Mihkel meeting for 3 days before it went well" (2 weeks ago)
- Open loop: "What to do about the investment question" (open 18 days)

Bad retrieval returns:
- All transactions from February
- A workout log from last week
- A fact about preferred working hours
- The user's city of residence

Bad retrieval fails because it is recent (recency_weight) or domain-matched (domain_match) without being semantically relevant. The scoring formula must prevent this by weighting semantic_similarity highest.

---

## Retrieval Failure Cases

### Nothing relevant found
All three methods return results, but composite scores are all < 0.3.
Action: skip Layer 2 entirely. Do not inject low-relevance items to fill the slot.
The LLM works better with no retrieved memory than with irrelevant retrieved memory.

### Retrieval timeout
Embedding search or structural query takes too long.
Action: fall back to recency retrieval only (fastest). Log the timeout.

### Conflicting memories retrieved
Two retrieved items directly contradict each other.
Action: include both, flagged as conflicting in the context: "Note: conflicting memory — [A] and [B]. User has not resolved this."
The LLM handles contradiction more gracefully when it is made explicit.

### High volume of relevant items
More than 10 items score above 0.5.
Action: apply stricter deduplication (merge items from the same Pattern), then take top 5.
Do not expand the retrieval budget because there is a lot of good content — maintain the limit.

---

## Embedding Index

The embedding index must support:
- Incremental updates (new memory items added without full reindex)
- Metadata filtering (confidence, domain, date, archived status)
- Fast approximate nearest neighbor search (< 100ms for top-10 results)

All memory items above Observation level are embedded at creation time and re-embedded if their content changes significantly (e.g. after a user correction in Mirror).

Embeddings are generated using the same model consistently. If the embedding model changes, the full index is rebuilt.
