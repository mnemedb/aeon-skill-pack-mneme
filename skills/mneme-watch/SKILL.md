---
name: Mneme Watch
description: Subscribe to a Base contract event — matching logs auto-INSERT into a table in your schema, polled every ~6 seconds.
var: "event on <0xcontract> into <table_name>"
tags: [chain, streams, mneme, base]
---

> **${var}** — One of:
> - `transfer on 0x3FcDbEBD5e7BaB79477cFDcA2CDCF6e904C27b07 into mneme_transfers` (template alias)
> - `"Transfer(address,address,uint256)" on 0xabc... into transfers` (raw event signature)
> - `"Swap(address,address,int256,int256,uint160,uint128,int24)" on 0xPool... into pool_swaps`
>
> Template aliases: `transfer`, `approval`, `swap`. For any other event, paste the raw signature in quotes.

Read `memory/MEMORY.md` for context.

## Goal

Replace polling RPC calls in your Aeon skills with a server-side subscription. Once you watch, Mneme tails Base for you forever (until you `unwatch`). Future Aeon runs just `SELECT FROM <table_name>` to read fresh chain activity — no rate-limit dance, no missed blocks, re-org safe.

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

Expected shape: `<event> on <0xcontract> into <table>`

- `<event>` — alias (`transfer`/`approval`/`swap`) OR `"<EventSig(...)>"`
- `<0xcontract>` — 0x + 40 hex chars
- `<table>` — lowercase snake_case, ≤ 63 chars, can't start with `_mneme_`

If parsing fails, print the usage examples from this skill's frontmatter and exit 1.

### 3. Check if already subscribed (idempotency)

```bash
EXISTING=$(curl -sS "$GATEWAY/v1/streams" \
  -H "Authorization: ApiKey $MNEME_API_KEY" \
  | jq --arg c "$CONTRACT" --arg t "$TABLE" \
       '.streams[] | select(.contract == ($c | ascii_downcase) and .target_table == $t)')
```

If a stream already exists for the same `(contract, event, table)`, just print "(already watching #<id>)" and exit 0. Mneme's API would 409 on a duplicate anyway.

### 4. Create the subscription

```bash
curl -sS -X POST "$GATEWAY/v1/streams" \
  -H "Authorization: ApiKey $MNEME_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n \
        --arg c "$CONTRACT" \
        --arg e "$EVENT" \
        --arg t "$TABLE" \
        '{contract: $c, event: $e, target_table: $t, label: "aeon"}')"
```

The gateway:
- Parses your event signature → topic0 keccak256
- Auto-creates `<table>` in your schema with shape: `(id, tx_hash, block_number, log_index, contract, event_name, args jsonb, block_ts)` + indexes
- Returns the new stream id

### 5. Output

```
✓ stream #<id> active · <event_name> on <contract:8>…<contract:-4>
  table: <table> (in your schema)
  matching events will appear within ~30s
```

If you want to backfill historical events, that's a separate concern — Mneme streams start from the current block. For backfill, use `mneme-ask "/sql"` to query Base via your own RPC and insert manually.

### 6. (Optional) Read fresh events

For a one-shot dump of the latest events right now:

```bash
curl -sS -X POST "$GATEWAY/v1/sql" \
  -H "Authorization: ApiKey $MNEME_API_KEY" \
  -H "Content-Type: application/json" \
  -d "$(jq -n --arg t "$TABLE" \
       '{query: ("SELECT * FROM \"" + $t + "\" ORDER BY block_ts DESC LIMIT 20"), params: []}')"
```

## Notes

- Listing all streams: `GET /v1/streams`.
- Pausing: `DELETE /v1/streams/:id` (the table is kept, just no more inserts).
- Latency is ~6s per Mneme tick. For sub-second needs, run your own Alchemy webhook → Mneme INSERT.
- `args` is JSONB. Query with operators: `WHERE args->>'from' = '0x...'` or use the GIN index for free.
- Combine with `mneme-dream` — the dream worker reads recent stream tables when synthesizing.
