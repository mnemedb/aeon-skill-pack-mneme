---
name: Mneme Ask
description: Free-form question to Claude with your Mneme schema, recent dreams, and active stream tables in its system prompt.
var: "the question to ask"
tags: [chat, mneme]
---

> **${var}** — Any natural-language question or instruction. Examples:
> - `"summarize my last 24h"`
> - `"what's the open question I should chase today"`
> - `"draft a tweet based on dream #14"`
> - `"/sql SELECT count(*) FROM mneme_transfers WHERE block_ts > now() - interval '1 hour'"` (literal SQL)

Read `memory/MEMORY.md` for context.

## Goal

Get a grounded answer that knows about EVERYTHING in your Mneme schema — memories, entities, relations, active chain streams, and the latest dream reflections — without having to load any of that context yourself.

Mneme's `/chat` endpoint auto-injects:

- Your handle + project schema name
- A compact list of every table and its columns
- All active Mneme Live streams (table → contract + event)
- The latest 3 dreams (kind, title, body)

So Claude already knows the shape of your agent's brain before it reads your question.

## Steps

### 1. Auth check

```bash
if [ -z "$MNEME_API_KEY" ]; then
  echo "MNEME_API_KEY is not set."
  exit 1
fi
GATEWAY="${MNEME_GATEWAY:-https://api.mnemedb.dev}"
```

### 2. Ask

```bash
RESPONSE=$(curl -sS -X POST "$GATEWAY/v1/llm/chat" \
  -H "Authorization: ApiKey $MNEME_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg p "${var}" '{prompt: $p}')")

REPLY=$(echo "$RESPONSE" | jq -r '.reply')
MODEL=$(echo "$RESPONSE" | jq -r '.model')
MS=$(echo    "$RESPONSE" | jq -r '.elapsed_ms')
```

### 3. (Optional) Multi-turn

If you need a follow-up in the same skill run, pass history:

```bash
HISTORY='[{"role":"user","content":"...prev question..."},{"role":"assistant","content":"...prev reply..."}]'

curl -sS -X POST "$GATEWAY/v1/llm/chat" \
  -H "Authorization: ApiKey $MNEME_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg p "${var}" --argjson h "$HISTORY" '{prompt: $p, history: $h}')"
```

History is capped at last 6 turns server-side to keep prompts tight.

### 4. Output

```
◆ <reply>

  (model · <elapsed>ms · schema-aware · stream-aware · dream-aware)
```

If the reply is more than 6 lines, also write it to `.outputs/mneme-ask.md` so later skills in the chain can read the full text without re-asking.

## When this skill is your best move

- **Daily brief generation** — "summarize my last 24h" produces a coherent paragraph because Claude already sees your memories + dreams + streams
- **Decision support** — "given my graph, what entity is missing that would close the biggest gap"
- **Self-introspection** — "what's the strongest claim in my schema that I haven't verified"
- **Raw SQL escape hatch** — `mneme-ask "/sql SELECT ..."` runs Claude-translated NL→SQL via the same chat endpoint (works for any read query)

## When to skip this skill

- **Pure lookup** (a known table, a known id) — use `mneme-recall` or direct `/v1/sql` instead, faster and free of LLM cost
- **Live external data** the schema doesn't know about — `mneme-ask` is grounded ONLY in your schema, it won't fetch the web
- **Multi-step planning that needs tools** — use Aeon's Claude Code path; `mneme-ask` is single-turn answer, not agent reasoning

## Notes

- Token cost is on Mneme's gateway-side fal.ai bill, NOT yours — your only knob is rate limit (per-API-key default 1200 rpm).
- Replies are deterministic-ish (temperature 0.6) — for true reproducibility, pass `temperature: 0` (not exposed in v1, but planned).
- This is the single most powerful skill in the pack — most chains end with `mneme-ask` because it ties everything else together.
