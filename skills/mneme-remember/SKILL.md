---
name: Mneme Remember
description: Save a thought or observation to your Mneme schema's memories table (with embedding) for later semantic recall.
var: "the thought to save"
tags: [memory, mneme]
---

> **${var}** — Free-form text to remember. If empty, ask the agent to summarize what it just learned in 1–3 sentences and use that.

Read `memory/MEMORY.md` for context.

## Goal

Persist a single durable memory in Mneme so future runs (and `mneme-recall`, `mneme-find`, `mneme-dream`, `mneme-ask`) can find it semantically.

## Steps

### 1. Auth check

```bash
if [ -z "$MNEME_API_KEY" ]; then
  echo "MNEME_API_KEY is not set. Get one from https://mnemedb.dev → API Keys (scope *)."
  exit 1
fi
GATEWAY="${MNEME_GATEWAY:-https://gateway.mnemedb.dev}"
```

### 2. Build the memory text

If `${var}` is non-empty, use it verbatim as the memory text.

If `${var}` is empty, generate a 1–3 sentence summary of what was learned in this Aeon run by reading the latest entry in `memory/logs/` (or, if running mid-skill-chain, by summarizing the relevant `.outputs/` files).

The memory should be:

- **Self-contained** — readable in 6 months without surrounding context
- **Specific** — names, numbers, contract addresses, not "the tool"
- **Falsifiable when possible** — "X claims Y" beats "X is great"

### 3. Insert

```bash
TEXT=$(cat <<'EOF'
${memory_text}
EOF
)

curl -sS -X POST "$GATEWAY/v1/sql" \
  -H "Authorization: ApiKey $MNEME_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg t "$TEXT" '{query: "INSERT INTO memories (text, metadata) VALUES ($1, $2::jsonb) RETURNING id, created_at", params: [$t, "{\"source\":\"aeon\",\"skill\":\"mneme-remember\"}"]}')"
```

Note: The raw SQL endpoint accepts API-key auth when the key's scope is `*`. If you scoped your key to a sub-namespace, use the typed insert endpoint instead:

```bash
curl -sS -X POST "$GATEWAY/v1/rows/memories" \
  -H "Authorization: ApiKey $MNEME_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg t "$TEXT" '{text: $t, metadata: {source: "aeon", skill: "mneme-remember"}}')"
```

### 4. Confirm

Parse the response, log the returned `id` and `created_at` to `memory/logs/`, and print a one-line confirmation:

```
✓ remembered #<id> at <created_at> · "<first 80 chars of text>…"
```

## Notes

- This skill does **not** generate embeddings client-side. The Mneme dashboard / SDK can backfill embeddings asynchronously, or you can pre-embed and pass an `embedding` array if you need immediate vector recall.
- For high-frequency saves (e.g. every loop iteration), batch into a single INSERT with multiple `(text, metadata)` rows.
- If the API call fails, append the unsent memory to `memory/MEMORY.md` as fallback so nothing is lost.
