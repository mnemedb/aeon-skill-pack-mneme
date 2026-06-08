---
name: Mneme Dream
description: Trigger a dream pass — Claude reflects on your recent memories / entities / relations and surfaces patterns, questions, gaps, synthesis.
var: "optional focus hint"
tags: [reflection, mneme]
---

> **${var}** — Optional hint biasing the LLM toward a topic. Examples: `"focus on what's connected to MNEME"` · `"what's missing from my graph"` · `"only synthesize, no questions"`. Leave empty for a free-form pass.

Read `memory/MEMORY.md` for context.

## Goal

Surface non-obvious observations from your Mneme schema that the current Aeon run might have missed:

- **pattern** — a co-occurrence / cluster you didn't notice
- **question** — something the data implies, worth investigating
- **gap** — data missing but likely useful given what IS there
- **synthesis** — narrative paragraph across the recent window

Mneme runs a background dream worker daily on its own. This skill **forces a fresh pass** right now — useful after a big ingest or when the agent needs the most current reflection.

## Steps

### 1. Auth check

```bash
if [ -z "$MNEME_API_KEY" ]; then
  echo "MNEME_API_KEY is not set."
  exit 1
fi
GATEWAY="${MNEME_GATEWAY:-https://api.mnemedb.dev}"
```

### 2. Generate dreams

```bash
HINT="${var}"

RESPONSE=$(curl -sS -X POST "$GATEWAY/v1/dreams/generate" \
  -H "Authorization: ApiKey $MNEME_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg h "$HINT" '{hint: ($h // null), max_dreams: 3}')")
```

The server reads up to: 15 recent memories + 8 documents + 25 entities + 25 relations, packs them into a prompt, calls claude-sonnet-4.5, and INSERTs 1–3 dreams into your `dreams` table.

If `RESPONSE.ok == false`, the schema is empty — output `(no data to dream about — add memories/entities first)` and exit 0.

### 3. Read back the just-inserted dreams

The `generate` response already includes the new dreams in `response.dreams[]`, but if you want them re-fetched with full metadata (sources, model):

```bash
curl -sS "$GATEWAY/v1/dreams?limit=3" \
  -H "Authorization: ApiKey $MNEME_API_KEY"
```

### 4. Output

For each dream, print:

```
[<kind>] <title>
  <body> (wrap at ~72 cols)
  sources: <id1>, <id2>, ...
```

End with a one-line action recommendation: pick the dream that's most actionable for the current run ("the gap dream is closable in 3 mneme-entity calls — do that first").

## Notes

- Dreams are **cheap to ignore** — they're persistent in your schema, so a future `mneme-recall "what did mneme dream about last week"` will find them.
- For long-running agents, schedule `mneme-dream` once daily and feed the output into a Slack notification — your DB does the homework while you sleep.
- The hint is powerful: `mneme-dream "what would a skeptic say about my last 24h"` produces very different output than `mneme-dream "what are the strongest signals"`.
- Output of this skill chains beautifully into `mneme-ask`: "given these dreams, what's the single best next action?"
