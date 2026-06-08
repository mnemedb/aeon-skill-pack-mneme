---
name: Mneme Recall
description: Semantic search across your Mneme memories — find what your past self learned about a topic.
var: "the topic to recall"
tags: [memory, mneme]
---

> **${var}** — Topic or question to search for. Free text. e.g. "pgvector performance", "what did I learn about Coinbase Smart Wallet", "everything about gas spikes on Base".

Read `memory/MEMORY.md` for context.

## Goal

Surface the 5–20 most relevant past memories so the current Aeon run grounds itself in what you already know — no rediscovering, no contradicting yesterday's findings.

## Steps

### 1. Auth check

```bash
if [ -z "$MNEME_API_KEY" ]; then
  echo "MNEME_API_KEY is not set. Get one from https://mnemedb.dev → API Keys."
  exit 1
fi
GATEWAY="${MNEME_GATEWAY:-https://api.mnemedb.dev}"
```

### 2. Decide retrieval strategy

If `${var}` is empty, error out: this skill needs a query.

Two strategies, pick based on the query shape:

- **Specific keywords/IDs** (e.g. "0x3FcD…", "EIP-712", "pgvector HNSW") → use SQL `ILIKE` for substring match. Fast, exact.
- **Conceptual / fuzzy** (e.g. "what slowed down our agent", "themes from last week") → use vector search via `mneme-find` instead (better recall for paraphrased ideas).

For specific keywords:

```bash
QUERY=$(printf '%s' "${var}" | sed "s/'/''/g")

curl -sS -X POST "$GATEWAY/v1/sql" \
  -H "Authorization: ApiKey $MNEME_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg q "%${var}%" '{query: "SELECT id, text, created_at, metadata FROM memories WHERE text ILIKE $1 ORDER BY created_at DESC LIMIT 20", params: [$q]}')"
```

For fuzzy queries, use the typed memories list endpoint with a `where` clause OR delegate to `mneme-find` with `kind=memory` (if your memories have been promoted into entities).

### 3. Rank and trim

Pull out 5–10 most useful memories. Prioritize:

1. **Recency** within the same theme (today > 3 days ago > 30 days ago)
2. **Specificity** (memories with numbers, addresses, names beat abstract observations)
3. **Outcomes** (memories that record what HAPPENED beat memories that record intent)

### 4. Output

Print each selected memory as a compact line:

```
[memory:<id>] <date> — <text trimmed to 200 chars>
```

End with a one-line synthesis: "Across these memories, the pattern is X — relevant to today's task because Y."

This synthesis is what the rest of the Aeon chain reads. Don't bury the lede.

## Notes

- If zero results, output `(no prior memories matched "${var}")` — do NOT fabricate. Let the caller fall back to a fresh investigation.
- If the SQL endpoint rejects raw SQL (your key wasn't `*`-scoped), use `GET /v1/rows/memories?where=text.ilike.%${var}%&order=created_at.desc&limit=20` instead.
- Found something worth flagging? Pipe into `mneme-remember` with a note like `"insight: <X> from memory #<id> still holds"` — compound memory beats lonely memory.
