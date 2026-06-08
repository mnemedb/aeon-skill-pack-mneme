---
name: Mneme Find
description: Hybrid vector + graph retrieval — find entities semantically similar to a query AND structurally connected via 1–4 hops.
var: "query [hops=N] [seed_k=K] [kind=K]"
tags: [graph, retrieval, mneme]
---

> **${var}** — Free-text query plus optional knobs. Examples: `"who's connected to MNEME"` · `"base ecosystem founders" hops=2 seed_k=10` · `"smart wallet protocols" hops=1 kind=protocol`.

Read `memory/MEMORY.md` for context.

## Goal

Answer "what do I know that relates to X" in a way pure vector search can't:

- **Vector seeds** the search (finds entities embedding-similar to the query)
- **Graph walks N hops out** from each seed (finds related entities even if they don't embed similarly)
- **Scores** each reached entity by `MAX(cosine_sim × decay^hops)`

This is what pgvector alone gets wrong when keyword similarity overlaps but meaning diverges.

## Steps

### 1. Auth check

```bash
if [ -z "$MNEME_API_KEY" ]; then
  echo "MNEME_API_KEY is not set."
  exit 1
fi
GATEWAY="${MNEME_GATEWAY:-https://gateway.mnemedb.dev}"
```

### 2. Parse the var

Extract:

- `QUERY` — everything before the first `key=value` pair
- `HOPS` — from `hops=N` (default 2, max 4)
- `SEED_K` — from `seed_k=K` (default 10, max 50)
- `KIND` — from `kind=K` (optional final-result entity-kind filter)

If `${var}` is empty, error.

### 3. Embed the query

Mneme's gateway expects a 1536-dim embedding for `semantic-neighbors`. Generate one with whatever embedding model Aeon uses:

```bash
# Option A — OpenAI
EMBED=$(curl -sS https://api.openai.com/v1/embeddings \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg q "$QUERY" '{model: "text-embedding-3-small", input: $q, dimensions: 1536}')" \
  | jq '.data[0].embedding')

# Option B — Bankr / OpenRouter / any compatible endpoint
# (substitute your embedding provider here)
```

### 4. Query

```bash
curl -sS -X POST "$GATEWAY/v1/graph/semantic-neighbors" \
  -H "Authorization: ApiKey $MNEME_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n \
        --argjson e "$EMBED" \
        --argjson sk "$SEED_K" \
        --argjson h  "$HOPS" \
        --arg k     "$KIND" \
        '{embedding: $e, seed_k: $sk, hops: $h, decay: 0.5, limit: 30}
         + (if $k == "" then {} else {kind: $k} end)')"
```

### 5. Output

For each match, print:

```
[entity:<id>] <kind>:<name>  score=<0.XX>  props=<compact>
```

Sorted by score descending. Top 5–10 typically have score > 0.4; below that is graph noise from far-out hops.

Append a one-line synthesis: "The strongest cluster around '${QUERY}' is X, with Y as the unexpected bridge node — worth investigating in this run."

## Notes

- If `mneme-find` returns 0 results, the graph is empty / sparse for this topic. Run `mneme-entity` + `mneme-relate` to seed it, then retry.
- For pure-graph traversal (no embedding needed), use `GET /v1/graph/neighbors/<id>?hops=N` instead — faster, no LLM cost.
- For pure-vector recall (no graph hops), pass `hops=0` — degrades gracefully to plain pgvector nearest-neighbor.
- The `decay=0.5` knob is good default. Use 0.7 if your graph is shallow (only 1–2 hops matter), 0.3 if it's deep (you want to penalize far hops harder).
