---
name: Mneme Entity
description: Add (or upsert) an entity in your Mneme graph — a person, token, contract, repo, anything you want to track.
var: "kind name [json properties]"
tags: [graph, mneme]
---

> **${var}** — `<kind> <name> [json]`. Example: `person vitalik {"wallet":"0xd8da6..."}` or `token MNEME` or `contract 0x3FcD...`. Kind ≤ 64 chars, name ≤ 256 chars. Both lowercase by convention but any casing works.

Read `memory/MEMORY.md` for context.

## Goal

Make an entity addressable in your graph so it can be linked to other entities via `mneme-relate` and traversed via `mneme-find`.

## Steps

### 1. Auth check

```bash
if [ -z "$MNEME_API_KEY" ]; then
  echo "MNEME_API_KEY is not set."
  exit 1
fi
GATEWAY="${MNEME_GATEWAY:-https://api.mnemedb.dev}"
```

### 2. Parse the var

Split `${var}` into:

- `KIND` — first token
- `NAME` — second token (or quoted multi-word phrase, e.g. `"Coinbase Smart Wallet"`)
- `PROPS` — remaining `{...}` JSON object, or `{}` if absent

If kind or name missing, error: `usage: mneme-entity <kind> "<name>" [{json props}]`.

### 3. Upsert

Mneme upserts by `(kind, name)` — running this skill twice with the same kind+name is safe. Properties merge; embedding overwrites if you pass one.

```bash
curl -sS -X POST "$GATEWAY/v1/graph/entities" \
  -H "Authorization: ApiKey $MNEME_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n \
        --arg k "$KIND" \
        --arg n "$NAME" \
        --argjson p "$PROPS" \
        '{kind: $k, name: $n, properties: $p}')"
```

If you want this entity to be findable via `mneme-find` with no graph hops (pure vector seed), also include an `embedding: [...]` field — but for most Aeon use cases, plain entities without embeddings work fine since the graph walk does the heavy lifting.

### 4. Output

Print the returned entity id and a one-liner:

```
✓ entity #<id> <kind>:<name>  (props: <key1>, <key2>, ...)
```

If the entity already existed (upsert), Mneme returns the existing id — same line, but you can note `(updated)` instead of `(created)` by checking if the returned `created_at` matches now-ish.

## Common kinds

You're free to invent kinds. These are common ones agents converge on:

- `person`, `team`, `org`
- `token`, `contract`, `protocol`
- `repo`, `pr`, `issue`
- `tweet`, `cast`, `thread`
- `wallet`, `tx`
- `event` (when you want graph reachability for a chain event row)
- `topic`, `theme`, `concept`

Stick to lowercase snake-case for kind to keep queries clean. Name can be anything.

## Notes

- Mneme provisions the `entities` table on first call — no setup needed.
- For bulk import (>10 entities), prefer a single SQL INSERT via `mneme-ask "/sql"` to save round-trips.
- Pair with `mneme-relate` immediately — a node without edges is invisible to graph queries.
