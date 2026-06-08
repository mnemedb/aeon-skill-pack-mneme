---
name: Mneme Relate
description: Add an edge between two entities in your Mneme graph — "person:vitalik holds token:MNEME".
var: "src_ref kind dst_ref [json props]"
tags: [graph, mneme]
---

> **${var}** — `<src> <edge_kind> <dst> [json]`. Refs can be numeric entity ids (`42`) OR `"kind:name"` strings (`"person:vitalik"`). Example: `person:vitalik holds token:MNEME {"since":"2026-05-30"}` or `repo:mneme depends_on protocol:base`.

Read `memory/MEMORY.md` for context.

## Goal

Make a relationship between two entities explicit and queryable. Edges power `mneme-find` (semantic + graph walk), shortest-path queries, and the "neighbors of" lookups Aeon needs for context-rich briefs.

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

Split into:

- `SRC_REF` — first token. Numeric → entity id. Contains `:` → `kind:name` lookup.
- `KIND` — second token. The edge type. Lowercase snake_case (e.g. `holds`, `created`, `replied_to`, `cites`, `bought_from`).
- `DST_REF` — third token. Same format as SRC_REF.
- `PROPS` — remaining JSON object (default `{}`).
- Optional `WEIGHT` from props.weight, defaults to 1.0.

If either entity doesn't exist, the API returns a clear error. Don't auto-create — that masks typos. Tell the user to run `mneme-entity` first.

### 3. Add the edge

```bash
curl -sS -X POST "$GATEWAY/v1/graph/relations" \
  -H "Authorization: ApiKey $MNEME_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n \
        --arg s "$SRC_REF" \
        --arg d "$DST_REF" \
        --arg k "$KIND" \
        --argjson p "$PROPS" \
        '{src: $s, dst: $d, kind: $k, properties: $p}')"
```

Note: `src` and `dst` accept either numbers OR strings via the API — pass them as strings and the gateway resolves `kind:name` references. If you have raw ids, pass them as JSON numbers.

Edges are upserted by `(src_id, dst_id, kind)` — running the same skill twice with identical refs just updates the properties. Safe to retry.

### 4. Output

```
✓ edge #<id> <src> ─[<kind>]→ <dst>  (props: ...)
```

## Common edge kinds

| Kind | Use |
| --- | --- |
| `holds` | wallet ↔ token, person ↔ token |
| `created` | person ↔ thing (token, repo, tweet) |
| `deployed` | wallet/person ↔ contract |
| `member_of` | person ↔ team/org |
| `replied_to` / `cites` | tweet ↔ tweet, doc ↔ doc |
| `depends_on` | repo ↔ repo, contract ↔ protocol |
| `mentioned_in` | entity ↔ memory/doc |
| `interacted_with` | wallet ↔ contract |
| `transferred_to` | wallet ↔ wallet |
| `follows` | person ↔ person |

Pick consistent kinds across your project — that's what makes `mneme-find` good. Sprawling vocabulary = noisy graph.

## Notes

- Use `properties.confidence` (0..1) when the edge is inferred rather than observed — `mneme-find` can later weight by it.
- Don't add an edge to record causation unless you actually have evidence; graphs lie quietly when overconfident.
- For bulk relations, batch via `mneme-ask "/sql"` with a multi-row INSERT.
