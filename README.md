# aeon-skill-pack-mneme

> Mneme as Aeon's persistent memory layer.
> Vector search · graph memory · live Base chain streams · async LLM reflection — all in one Postgres schema, callable from any Aeon skill.

[Aeon](https://github.com/aaronjmars/aeon) ships with persistent memory as `memory/MEMORY.md` + a few JSON files in the repo. This works great until you have months of state — then it doesn't scale, doesn't search semantically, and can't reason across runs.

[Mneme](https://mnemedb.dev) is an agent-native database (real Postgres + pgvector + entities/relations graph + live Base event streams + daily LLM "dream" reflections) designed exactly for this case. One API key, one schema per project, vector + graph + streams + chat built-in.

This skill pack lets any Aeon agent **read and write Mneme** in 8 calls.

---

## What's in the pack

| Skill | What it does |
| --- | --- |
| `mneme-remember`  | Save a thought / observation to your `memories` table (with embedding for later semantic recall). |
| `mneme-recall`    | Vector search across all your memories — find what your past self learned about a topic. |
| `mneme-entity`    | Add an entity (person, token, contract, repo, anything) to your graph. |
| `mneme-relate`    | Add an edge between two entities — `person:vitalik holds token:MNEME`. |
| `mneme-find`      | Hybrid vector+graph retrieval — find entities semantically similar to a query AND structurally connected. |
| `mneme-dream`     | Trigger a dream pass — Claude reflects on your recent data and surfaces patterns / questions / gaps. |
| `mneme-watch`     | Subscribe to any Base contract event — matching logs auto-INSERT into a table in your schema. |
| `mneme-ask`       | Free-form chat against your schema — Claude sees all your tables, entities, relations, streams and dreams in its system prompt. |

---

## Why pair Aeon + Mneme

Aeon is the most autonomous **execution** layer. Mneme is the most agent-native **memory** layer. They're built for the same kind of user — the developer who wants their agent to run unattended, learn over time, and answer questions about its own history.

Concretely, this pack gives Aeon:

- **Semantic memory** across runs (your daily brief from 47 days ago, recallable by topic)
- **Graph-shaped knowledge** (who's connected to what, what wallets touched what contracts)
- **Live chain awareness** (subscribe once, your DB tails Base for you)
- **Self-reflective synthesis** (Mneme Dreams reads your data and tells you what you're missing)

All without leaving Aeon. Skills compose: `mneme-watch` writes to your schema, `mneme-recall` queries it, `mneme-dream` reflects on it, `mneme-ask` ties it together for a daily brief.

---

## Install

```bash
# from your Aeon directory
./install-skill-pack mnemedb/aeon-skill-pack-mneme
```

This drops the 8 skills into `skills/` alongside your existing ones.

---

## Auth — one API key, two minutes

1. **Sign in to Mneme** at [mnemedb.dev](https://mnemedb.dev) (email, Google, X, or your wallet).
2. **Pick a handle** — this becomes your schema name (`agent_<handle>`).
3. **Go to the API Keys tab** in the dashboard → "Create key" → scope `*` (full access) → label "aeon".
4. **Copy the key** (starts with `mneme_sk_…`) — shown once, won't be shown again.
5. **Add to your Aeon environment**:

```bash
# Local
export MNEME_API_KEY=mneme_sk_...

# GitHub Actions (Settings → Secrets and variables → Actions → New repository secret)
MNEME_API_KEY = mneme_sk_...
```

Optional override:

```bash
export MNEME_GATEWAY=https://api.mnemedb.dev   # default
```

Every skill checks `MNEME_API_KEY` first and fails fast with a helpful message if it's missing.

---

## Example chain — daily Base ecosystem brief

Wire these together in `aeon.yml`:

```yaml
- name: morning-base-brief
  schedule: "0 7 * * *"
  steps:
    - mneme-watch:  "Transfer on 0x3FcDbEBD5e7BaB79477cFDcA2CDCF6e904C27b07 into mneme_transfers"
    - mneme-dream:  ""
    - mneme-ask:    "summarize my last 24h: what changed onchain, what did I learn, what's the open question I should chase today?"
    - notify:       slack
```

The first run sets up the stream. Every morning after that:

1. Mneme has been writing Base events to your `mneme_transfers` table all night
2. `mneme-dream` synthesizes patterns across memories + entities + streams
3. `mneme-ask` writes the brief using all of it — schema, dreams, the works
4. Slack gets a coherent daily brief grounded in real data, not a fresh-from-zero LLM hallucination

---

## What gets created in your schema

Mneme provisions a per-project schema with these tables, automatically:

```
memories     (id, text, embedding vector(1536), metadata jsonb, created_at)
documents    (id, title, body, embedding, metadata, created_at)
events       (id, kind, payload jsonb, created_at)
kvs          (key, value jsonb, updated_at)
entities     (id, kind, name, properties jsonb, embedding, created_at)   ← graph
relations    (id, src_id, dst_id, kind, weight, properties, created_at)  ← graph
dreams       (id, kind, title, body, sources jsonb, model, created_at)   ← reflection
+ any custom tables your streams or skills create
```

You can also drop into raw SQL via the Mneme dashboard or `mneme-ask "/sql SELECT ..."`.

---

## Compatibility

- Aeon: any version
- Mneme: gateway v0.4+ (uses `/v1/dreams`, `/v1/graph`, `/v1/streams`)
- Auth: API key (Service Account, scope `*`)
- Bash + `curl` + `jq` (already present in GitHub Actions runners)

---

## Mneme primitives this pack exposes

| Mneme primitive | Skill that uses it |
| --- | --- |
| Memory (pgvector) | `mneme-remember`, `mneme-recall` |
| Graph (entities + relations) | `mneme-entity`, `mneme-relate`, `mneme-find` |
| Live (Base chain streams) | `mneme-watch` |
| Chat (schema-aware Claude) | `mneme-ask` |
| Dreams (async LLM reflection) | `mneme-dream` |

Five primitives, one schema, one wallet, one API key. Built for agents like Aeon.

---

## License

MIT — see [LICENSE](./LICENSE).

## Links

- Mneme: [mnemedb.dev](https://mnemedb.dev) · [docs](https://mnemedb.dev/docs) · [github.com/mnemedb/mnemedb](https://github.com/mnemedb/mnemedb)
- Aeon: [github.com/aaronjmars/aeon](https://github.com/aaronjmars/aeon)

---

> *"Aeon decides when to think. Mneme remembers what it learned."*
