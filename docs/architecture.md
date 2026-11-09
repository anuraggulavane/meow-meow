# Architecture Proposal: Portfolio + AI Digital Twin

## Context

You want a production-quality portfolio with a RAG chatbot that answers questions about you. The main goal is learning to design, build, debug, evaluate and deploy this kind of system yourself. This document is the Phase 1 deliverable. **It contains no application code, and nothing gets built until you approve it.** Once approved, it moves into `docs/architecture.md`.

**Repository state.** `meow-meow` is empty: no commits, `main` only, remote set to `github.com/anuraggulavane/meow-meow`. The GitHub API returns 404 for it, so it is either private or not created yet.

**Local toolchain**

| Tool | State | Action |
|---|---|---|
| Python 3.14.7 | OK. `asyncpg`, `SQLAlchemy`, `fastapi`, `google-genai`, `pgvector` all ship compatible wheels | Pin Docker to 3.14 to match |
| Node 25.6.1 / npm 11.9 | Node 25 is not an LTS line | Fine locally; pin Docker build stage to the current LTS |
| Docker 29.5 | **Daemon not running** | Start Docker Desktop before Phase 2 |
| Supabase CLI | **`supabase --version` printed nothing** | Reinstall or check at the start of Phase 2 |
| `railway`, `gh`, `pnpm` | Not installed | Not needed: Railway deploys from GitHub, and we use npm |

---

## 1. Where I disagree with the brief

You asked me to be critical. These are the changes I recommend, most important first. Each is a default you can overrule.

| # | Brief says | Problem | Recommendation |
|---|---|---|---|
| 1 | Embed the visitor question directly | **Follow-ups break retrieval.** "What projects did he work on during that time?" embeds with no mention of the master's degree, so conversation history reaching Gemini doesn't help: the wrong chunks were already fetched | Add a **query-rewrite step** before embedding, only when history exists. The brief's Pipeline B doesn't have this step |
| 2 | Top-k retrieval for everything | **"List everything" questions fail.** "What projects has he built?" returns k fragments, not the full list | Generate deterministic **overview chunks** per collection (e.g. all project names with one-liners) at ingestion |
| 3 | RAG is the core architecture | Your knowledge base will probably be ~10–30K tokens, which fits in a 1M context window. RAG isn't strictly needed at this size | Keep RAG, because learning it is the point, but **add "whole KB in the prompt" as an eval baseline**. Beating or matching it is how you prove the retrieval works, and it makes a strong interview story |
| 4 | Create vector indexes | With under ~1,000 chunks, an exact scan should take only milliseconds and has perfect recall. HNSW is approximate and adds tuning | **No ANN index at launch.** In Phase 6, measure with `EXPLAIN ANALYZE` and add HNSW only if it earns its place. A measured "no" is a better lesson than cargo-culting an index |
| 5 | `POST /ingestion`, document admin endpoints | Puts write credentials in the public web process and creates the upload attack surface you then have to defend | **Ingestion is a CLI for v1.** The public process stays read-only and there are no uploads to secure. Protected endpoints can come later as thin wrappers |
| 6 | Visibility enforced at query level | A `WHERE` clause is still application code, and one forgotten filter leaks data | **Enforce with Postgres permissions.** The web app logs in as a role that can only `SELECT` from a public-only view (§6) |
| 7 | Private data in the knowledge base | Nothing in v1 is allowed to read it, and a private row that doesn't exist cannot leak | Build the mechanism and test it, but **keep `data/private/` empty** until a real use appears |
| 8 | `pip freeze > requirements.txt` | Conflicts with your own "clean, intentional list": freeze dumps every transitive package and whatever else is in the venv | `requirements.in` (intentional, hand-edited) compiled to a pinned `requirements.txt` with `pip-tools`. Separate dev file so pytest doesn't ship to production. Still pip and `.venv` |
| 9 | Deployment is Phase 10 | Networking, `$PORT` and build problems surface at the very end, when everything is tangled together | **Deploy a hello-world in Phase 2**, then ship continuously |
| 10 | Two deployed apps | Doubles cost and adds CORS for a personal site | **One Railway service**: FastAPI serves the static frontend (§12) |
| 11 | "Digital twin" | Rule 9 says never claim to be Anurag, and first person would imply that | Third person ("Anurag built…") and a visible "AI, may be wrong" label |
| 12 | Contact form | Needs an email provider, spam protection, and a write path | `mailto:` plus links for v1 |
| 13 | Parse PDF in v1 | Your resume duplicates the JSON, so facts appear twice and compete in retrieval | JSON, Markdown and text first. PDF parser later (`pypdf`, pure Python) |
| 14 | `docker-compose.yml` | The Supabase CLI already runs local Postgres in Docker; compose would only wrap one container | Skip for v1. Use `docker build` / `docker run` to test the production image |

### One thing to know about the free tiers

**Supabase pauses free projects after a week of inactivity.** Low traffic is normal for a portfolio, so this will happen. It's a major reason for the static frontend: if the database sleeps, the site stays up and only the chat degrades. Mitigation is a scheduled ping, and we must confirm that a ping counts as activity. GitHub also disables scheduled workflows on inactive repos, so this needs monitoring. The fully reliable fix is Pro at $25/month.

---

## 2. High-level architecture

```
                     Visitor's browser
                            │  HTTPS, same origin (no CORS in prod)
                            ▼
┌──────────────── Railway: ONE container ────────────────┐
│  FastAPI (uvicorn)                                     │
│   /            → pre-rendered static site (Next export)│
│   /api/v1/*    → JSON and SSE                          │
│                                                        │
│  chat pipeline: validate → rate limit → rewrite →      │
│    embed → retrieve → build context → generate         │
└───────┬───────────────────────────────┬────────────────┘
        │ SELECT only (role: chat_reader)│ HTTPS, API key
        │ session pooler, IPv4           │
        ▼                                ▼
┌───────────────────┐           ┌───────────────────┐
│ Supabase Postgres │           │ Gemini API        │
│ + pgvector        │           │ embed + generate  │
└─────────▲─────────┘           └─────────▲─────────┘
          │ read/write (admin URL)        │
┌─────────┴───────────────────────────────┴──────┐
│ Ingestion CLI (your laptop or CI, never web)   │
│ data/*.json → parse → chunk → embed → upsert   │
└─────────▲──────────────────────────────────────┘
          │
      data/*.json ──── also read at build time ───► Next.js site
```

**The key idea: `data/` is the single source of truth.** The site is built from the same files the chatbot ingests, so the two can't contradict each other.

The browser only ever talks to FastAPI. It never sees Gemini, Supabase, or any key. There is no Supabase client library in this system; we talk to Postgres directly, which keeps the SQL visible for you to learn from.

---

## 3. Frontend: Next.js with static export

**Stack:** Next.js (App Router, `output: 'export'`), React, TypeScript, Tailwind, Source Code Pro via `next/font`. Package manager: npm.

| Option | SEO and link previews | Learning and job value | Production footprint | Verdict |
|---|---|---|---|---|
| **Next.js static export** | Real HTML at build time | Strong | Static files, no Node server | **Chosen** |
| Vite + React SPA | Content needs JS to render | Good | Static files | Weaker SEO, your stated requirement |
| Next.js with server | Excellent | Strong | Second always-on service | Paying for SSR we don't use |
| Astro + React island | Excellent | Less React practice | Static files | Technically the closest fit for a content site. If you care more about minimal JS than React experience, take this one |

**Why it fits:** content changes only when you edit `data/`, so build-time rendering gives the fastest, simplest result. Chat is the only dynamic part, and it's a client component calling FastAPI.

**What you give up:** no Next API routes, server actions or rewrites. That's fine, since FastAPI handles all of that. In dev the site runs on `:3000` and the API on `:8000`, so CORS is enabled by env var for local only.

**Structure:** one scrolling home page with all sections, plus a `/projects/[slug]` page for each project's engineering detail. Chat is a persistent panel. A terminal-inspired look that doesn't imitate a terminal: type, whitespace and hairlines, no cards or gradients.

---

## 4. Backend architecture

FastAPI, Pydantic v2, pydantic-settings, SQLAlchemy Core (async) + asyncpg, `google-genai`.

```
HTTP route    parse, validate, return               api/v1/
  ▼
Pipeline      orchestrates the steps                rag/pipeline.py
  ▼
Components    rewrite, retrieve, build context      rag/
  ▼
Adapters      Postgres queries, Gemini calls        db/, providers/
```

**Decisions**

- **SQLAlchemy Core, not the ORM.** With two tables and hand-tuned vector SQL, ORM models would duplicate the migrations and hide the SQL you want to learn. Core gives us pooling and bound parameters, so queries stay injection-safe.
- **Exactly two abstractions:** `EmbeddingProvider` and `LLMProvider` as `typing.Protocol`. You required swappability, and they also let tests run without hitting Gemini. No other interface layers, and no repository pattern beyond plain query functions.
- **Async where it matters.** Chat spends nearly all its time waiting on Gemini and Postgres, which is what async is for. Use `client.aio` from the Gemini SDK, because a sync call inside an `async def` route blocks every other visitor. This is the classic FastAPI bug and worth understanding.
- **Dependency injection** via `Depends`, so tests can swap in fakes.
- **Middleware:** request ID, structured access log, body size limit, security headers.
- **Errors:** typed domain exceptions mapped to one consistent JSON error shape. Internals never reach the client.

**Python environment.** `backend/.venv` per your spec, never committed, never in Docker. Docker runs `pip install` from the pinned `requirements.txt` in its own image.

| Environment | Python comes from | Dependencies | Config |
|---|---|---|---|
| Local | `backend/.venv` | `requirements-dev.txt` | `.env` |
| Docker | `python:3.14-slim` image | `requirements.txt` | `--env-file` |
| Railway | Same Dockerfile | Same | Railway variables |

---

## 5. Component communication

| From → to | Protocol | Auth | Notes |
|---|---|---|---|
| Browser → FastAPI | HTTPS, JSON + SSE | None (public), rate limited | Same origin |
| FastAPI → Postgres | Postgres wire protocol | `chat_reader` | Supavisor **session** pooler |
| FastAPI → Gemini | HTTPS | API key from env | Embed + generate |
| Ingestion CLI → Postgres | Postgres wire protocol | Admin URL | Never set on Railway |

**Why the session pooler:** Supabase's direct host is IPv6 by default, while the pooler is IPv4 on every plan. Reports conflict on Railway's outbound IPv6, so the pooler avoids the question. Session mode also keeps prepared statements, which asyncpg uses by default; transaction mode (port 6543) doesn't support them. Phase 2's deploy tests this path for real.

**Streaming:** SSE over `fetch` + `ReadableStream`. The browser's `EventSource` only supports GET, and we need POST. Once streaming starts the HTTP status is already 200, so failures need an in-band `error` event.

---

## 6. Database schema

Migrations are plain SQL managed by the Supabase CLI, in `supabase/migrations/`. No Alembic, because the CLI already does this and you'll be writing real SQL.

Tables live in a dedicated **`kb` schema, not `public`**. Supabase auto-exposes `public` through its REST API, so keeping tables out of it removes a whole class of accidental exposure. RLS is also enabled with no policies as a second layer.

```sql
create extension if not exists vector;
create schema kb;

create table kb.documents (
  id           uuid primary key default gen_random_uuid(),
  source_key   text not null unique,       -- stable: 'projects/rag-portfolio'
  title        text not null,
  source_type  text not null check (source_type in ('json','markdown','text','pdf')),
  source_path  text not null,
  content_hash text not null,
  visibility   text not null default 'private'     -- fails closed
               check (visibility in ('public','private')),
  metadata     jsonb not null default '{}',
  created_at   timestamptz not null default now(),
  updated_at   timestamptz not null default now()
);

create table kb.document_chunks (
  id              uuid primary key default gen_random_uuid(),
  document_id     uuid not null references kb.documents(id) on delete cascade,
  chunk_index     int  not null,
  content         text not null,
  embedding       vector(768) not null,
  embedding_model text not null,
  metadata        jsonb not null default '{}',
  created_at      timestamptz not null default now(),
  unique (document_id, chunk_index)
);
create index on kb.document_chunks (document_id);

-- The only thing the web app can see
create view kb.public_chunks as
  select c.id, c.content, c.embedding, c.embedding_model, c.metadata,
         d.source_key, d.title
  from kb.document_chunks c
  join kb.documents d on d.id = c.document_id
  where d.visibility = 'public';

create role chat_reader login;      -- password set out-of-band, never in a migration
grant usage on schema kb to chat_reader;
grant select on kb.public_chunks to chat_reader;
```

**Design notes**

- **A "document" is one logical record** (one project, one job), not one file. That gives clean source attribution, and evaluation labels that survive re-chunking.
- **Privacy is testable.** An automated test connects as `chat_reader` and asserts that reading `kb.document_chunks` raises `permission denied`. Even a SQL injection bug couldn't read private rows or write anything.
- **`embedding_model` per chunk** is required because embedding spaces are incompatible across models. Retrieval filters on it, so a half-finished migration can't compare vectors from different spaces.
- **Privacy rule by location:** `data/*.json` is public; gitignored `data/private/` is private. No per-record flags to forget.

**Conversations and messages: deferred.** v1 has no conversation storage, so the web app stays read-only and there's no visitor data to protect. The trade-off is that you can't see what visitors ask. That data is genuinely useful, so it's a good early v2 feature, with a retention policy and a privacy notice.

---

## 7. How pgvector stores and retrieves

Each chunk's 768 floats live in a `vector(768)` column. The query is ordinary SQL:

```sql
select content, source_key, title, metadata,
       1 - (embedding <=> :query_vec) as similarity
from kb.public_chunks
where embedding_model = :model
order by embedding <=> :query_vec
limit :k;
```

`<=>` is cosine **distance**, so similarity is `1 - distance`. The similarity threshold is applied afterwards in Python, so we can log what was rejected. This matters when tuning.

**Strategy:** exact cosine search for v1. HNSW (`vector_cosine_ops`) is the scale-up path when measurements justify it. pgvector's plain `vector` index limit is 2,000 dimensions, another reason not to use the full 3072. Hybrid search (Postgres full-text fused with vectors by reciprocal rank fusion) comes only if the eval shows exact-term misses such as project names or certifications.

---

## 8. Ingestion pipeline

Run with `python -m app.ingestion sync [--dry-run] [--prune]`.

```
data/*.json
  → load & validate (Pydantic: a typo fails loudly here)
  → render records to natural prose, with context header
  → chunk
  → hash(text + chunker version + embedding model)
  → unchanged? skip
  → changed or new? embed (outside any transaction)
  → one transaction per document: upsert doc, replace chunks
  → with --prune: delete documents no longer on disk
```

- **Idempotent:** running twice changes nothing and costs nothing. The hash includes chunker version and model, so changing either correctly triggers re-embedding.
- **Atomic:** a failure leaves the previous version intact. Embedding happens before the transaction, so we never hold a database transaction open across a slow network call.
- **Fact vs opinion:** records carry `kind: fact | opinion`, passed to Gemini as metadata. This is how "distinguish facts from opinions" gets implemented rather than merely requested.
- **Validation in CI** catches malformed data before deployment.

### Chunking strategy

Structure first, size limits second. Your data is already structured, so blind fixed-size splitting would throw that away.

| Source | Rule |
|---|---|
| JSON record | One record → one chunk if it fits. If not, split long fields by paragraph and **repeat the context header on every piece** |
| Markdown | Split on headings; keep the heading path as a prefix |
| Plain text | Paragraphs, then sentences |
| Overview | One generated summary chunk per collection (fix #2 above) |

```
Project: <name> | Period: <dates> | Technologies: <list>
<prose about the project>
```

The header is what keeps "this paragraph about caching" linked to its project and technologies. **Dates must be in every chunk**, or "during that time" questions can't be answered.

**Starting values,** all configurable and tuned by the eval: target 300–500 tokens, hard max 800, no overlap between records, ~1 sentence overlap only when splitting prose.

---

## 9. Embedding model

**`gemini-embedding-2` at 768 dimensions.**

| Setting | Value | Why |
|---|---|---|
| Model | `gemini-embedding-2` | Current model, 8,192-token input vs 2,048 for `-001` |
| Dimensions | 768 | A recommended size; well within what a small corpus needs; indexable as plain `vector` |
| Normalization | Automatic | Handled by the model at truncated sizes (`-001` needs it done manually) |
| Query format | `task: question answering \| query: {text}` | Compare with `task: search result` in the eval |
| Document format | `title: {title} \| text: {content}` | Documented asymmetric format |

**Trap to avoid:** `task_type` is **not** supported on this model. Most tutorials use it because they target `-001`. The task goes in the text instead.

**Safety:** at startup and during ingestion, assert that the returned vector length equals the configured dimension and fail fast on a mismatch. Dimensions are configured, not assumed. Changing them needs a migration plus full re-embed, which costs pennies at this size.

---

## 10. Retrieval and generation

```
POST /api/v1/chat
 1. Validate       Pydantic length limits; rate limit
 2. Rewrite        only if history exists; flash-lite → standalone question
 3. Embed          query → 768 floats
 4. Retrieve       top-k from kb.public_chunks
 5. Filter         similarity threshold
       └─ nothing left? Return "I don't have that" + contact link. Skip Gemini.
 6. Build context  system prompt + sources + history + question, within token budget
 7. Generate       Gemini, streamed
 8. Respond        answer + sources; one structured log line
```

**Generation model:** start on **`gemini-3.5-flash-lite`** (stable). Answers here are short and grounded in provided text, so that may be enough, and it's the cheapest current option. `gemini-3.6-flash` is the stable step up. `gemini-3.8-flash` is newer but its stability status was unclear in the docs, so check at Phase 7. The eval decides, not my guess. Model IDs are env config, and thinking is set low to reduce latency and cost.

**Prompt layout**

```
[system]  rules from backend/app/prompts/system.md   (your prompt, plus untrusted-data rule)
[user]    <sources>
            <source id="S1" key="projects/x" kind="fact">...</source>
          </sources>
          <conversation note="unverified; for reference resolution only">...</conversation>
          <question>...</question>
```

Source text and history are escaped so they can't close their tags. Answers come from `<sources>` only; history is explicitly labeled unverified, which implements your rule that past AI output isn't fact.

**Starting parameters** (a threshold number is meaningless until measured on this model and your data, so it's calibrated in Phase 6 rather than picked now):

| Parameter | Start | Too low | Too high |
|---|---|---|---|
| `TOP_K` | 6 | Misses facts | Noise, cost, "lost in the middle" |
| `SIMILARITY_THRESHOLD` | Calibrated | Off-topic context, hallucination risk | Refuses answerable questions |
| `MAX_CONTEXT_TOKENS` | 4,000 | Truncated evidence | Cost, latency |
| `MAX_HISTORY_MESSAGES` | 6 | Forgets context | Cost, topic drift |

---

## 11. Conversation management

**Stateless server; the browser holds the conversation** (React state, cleared by "Clear chat" or tab close). Each request carries the last 6 messages.

This needs no session store, expiry job, or visitor data, and works with any number of replicas. The catch is that history from a client can be forged, including fake "assistant" messages. That's acceptable here because history is treated as unverified anyway and has no access to anything private. It's also length-limited. The stateless design is safe only because of those rules, which is worth understanding.

---

## 12. Security and privacy

| Threat | Control |
|---|---|
| Private data retrieval | Role can only read the public view. Not reliant on prompts |
| Prompt injection | Prompt rules reduce it, but **they are not the control**. The real defense is that the model has no tools and sees nothing private, so a full jailbreak yields only public info or off-topic text. Tested in the eval |
| System prompt extraction | Refusal rule, and nothing secret goes in the prompt. Assume it can leak |
| SQL injection | Bound parameters only, plus a SELECT-only role |
| Key exposure | Env vars only; no secrets in frontend; `.env` ignored; secret scanning in CI |
| Cost attack on Gemini | Per-IP limit, global daily cap, and a budget alert in Google's console. Most likely real-world risk for a public chatbot |
| Oversized input | Message ≤ 1,000 chars, history capped, body size limit |
| Malicious uploads | No upload endpoint exists |
| XSS via model output | Render as text/safe markdown; never raw HTML |
| Personal data in logs | Log metadata (latency, scores, tokens), not message text |

**Rate limiter:** about 30 lines, in memory, built by you. Fine for one replica, and it resets on deploy. Behind Railway's proxy, every request appears to come from the proxy, so the limiter must read the forwarded client IP, or all visitors will share one bucket. We'll verify the exact header in production.

**Gemini data use:** on the free tier, Google's pricing page says data is used to improve its products, and that covers visitor questions and your KB. See decision 1 in §19.

---

## 13. Repository structure

Simplified from your draft: flatter, no `models/`, and `supabase/` replaces `database/` because that's where the CLI expects it. The Dockerfile sits at the root because it builds both apps.

```
meow-meow/
├── frontend/
│   ├── src/{app,components,sections,lib}/
│   └── public/                 # photo, resume.pdf
├── backend/
│   ├── .venv/                  # gitignored
│   ├── app/
│   │   ├── main.py  config.py  logging.py  errors.py
│   │   ├── api/v1/             # chat.py, health.py
│   │   ├── schemas/            # request/response models
│   │   ├── db/                 # engine, queries
│   │   ├── providers/          # base.py (Protocols), gemini.py
│   │   ├── rag/                # pipeline, rewrite, retrieve, context
│   │   ├── ingestion/          # cli, loaders, render, chunker, sync
│   │   └── prompts/            # system.md, rewrite.md
│   ├── tests/
│   ├── requirements.in  requirements.txt
│   └── requirements-dev.in  requirements-dev.txt
├── data/                       # single source of truth (+ certifications.json)
│   └── private/                # gitignored
├── supabase/migrations/
├── evaluation/{questions.json, run.py, results/}
├── docs/{architecture,database,rag,deployment}.md
├── .github/workflows/ci.yml
├── Dockerfile  .dockerignore  railway.json
├── .gitignore  .env.example  DEVELOPMENT.md  README.md
```

Note: `certifications.json` was in your §5.3 list but not your §8 tree. I've kept it.

---

## 14. API design

| Method | Path | Purpose | Auth | Version |
|---|---|---|---|---|
| POST | `/api/v1/chat` | JSON answer. Used by tests and the eval | Public, limited | v1 |
| POST | `/api/v1/chat/stream` | Same pipeline as SSE. Used by the UI | Public, limited | v1 |
| GET | `/api/v1/health` | Liveness: no dependencies | Public | v1 |
| GET | `/api/v1/health/ready` | Readiness: checks database | Public | v1 |
| GET/DELETE/POST | `/api/v1/documents...`, `/ingestion` | Admin | Key | Later, if ever |

Liveness and readiness are separate deliberately. Railway's health check uses liveness, so a Supabase pause can't fail your deployment.

```jsonc
// POST /api/v1/chat
{ "message": "What did he build at WPI?",
  "history": [{ "role": "user", "content": "..." },
              { "role": "assistant", "content": "..." }] }

// 200
{ "answer": "...",
  "sources": [{ "key": "projects/x", "title": "..." }],
  "grounded": true,
  "request_id": "..." }

// Errors (always this shape): 422 validation, 429 limited, 503 upstream
{ "error": { "code": "rate_limited", "message": "...", "request_id": "..." } }
```

SSE events: `sources` → `token`… → `done` (or `error`).

---

## 15. Testing and evaluation

**Tests**

| Layer | Tool | What | Real or fake |
|---|---|---|---|
| Unit | pytest | Chunker, renderer, hashing, context budget, rate limiter | Pure functions |
| API | pytest + httpx | Contracts, validation, errors, SSE order | Fake providers |
| Database | pytest + real pgvector | Migrations, ranking, sync idempotence, **privacy** | Real Postgres; mocking would test nothing |
| Frontend | Vitest + Testing Library | Chat stream handling, error states | Mock fetch |
| E2E | Playwright | Later | |

CI on every push: ruff, pytest with a Postgres service, `tsc`, Vitest, `next build`, data validation, and Docker build. **No real Gemini calls in CI:** they cost money and are nondeterministic, which makes tests flaky.

**RAG evaluation** runs manually with `python evaluation/run.py --label <experiment>`. It writes results to `evaluation/results/` so you can compare runs.

- **Dataset:** 20+ questions covering every category in your §6, each labeled with expected `source_key`s, not chunk IDs, so labels survive re-chunking. I'll draft the structure, and **you write or verify the expected answers**, since only you know the truth.
- **Retrieval:** Recall@K, Precision@K, MRR, plus similarity distributions for answerable vs unanswerable questions. Those distributions are how the threshold gets set.
- **Generation:** faithfulness, relevance, completeness, correct refusal on unknowns, via Gemini as judge with a fixed rubric. LLM judges are imperfect, so you hand-check a sample. With 20 questions, a difference of one or two isn't meaningful, and that's a lesson in itself.
- **Baselines:** the full-KB-in-prompt comparison from fix #3.
- **Injection set:** attempts to extract the prompt, fabricate credentials, or go off-topic.

**Observability:** stdlib `logging` with a JSON formatter to stdout, which Railway captures. One line per chat request with request ID, latency per stage, token counts, chunk count, top similarity, estimated cost. Sentry is a later option.

---

## 16. Docker and Railway

```
Stage 1: node LTS          npm ci, next build → out/   (needs data/)
Stage 2: python:3.14-slim  pip install → copy backend + out/
                           non-root user; uvicorn on $PORT, 1 worker
```

Build context is the repo root, since the frontend reads `data/`. `.dockerignore` excludes `.venv`, `node_modules`, `.git`, `.env`, `data/private`.

- **Config as code:** `railway.json` sets the Dockerfile and health check path.
- **Deploy flow:** push to `main` → CI → Railway builds → health check → traffic switches.
- **Migrations:** run `supabase db push` as an explicit step before deploying, not at app startup. The app role can't run them anyway, and that's correct.
- **Ingestion:** run the CLI manually after data changes for v1. Weakness: a data change redeploys the site automatically but not the KB, so they can briefly disagree. A CI job can close this gap later.
- **Variables on Railway:** `DATABASE_URL` (`chat_reader`), `GEMINI_API_KEY`, model and retrieval config. **`ADMIN_DATABASE_URL` never goes on Railway.**

**Estimated cost**

| Item | Monthly | Notes |
|---|---|---|
| Railway Hobby | ~$5 | Includes $5 usage; a small service should fit, but confirm on the usage page |
| Supabase Free | $0 | Pauses when inactive; 500 MB limit (third-party sources; verify) |
| Gemini, paid tier | Likely under $1 | About $0.0014 per message on flash-lite, assuming ~3K input + 200 output tokens; 500 messages ≈ $0.70 |
| Custom domain | ~$1 | Optional |

---

## 17. Roadmap

Each phase ends with something working and a checkpoint with you. "You build" items are the exercises where writing it yourself teaches the most; I'll set up scaffolding and review your code.

| # | Phase | Outcome | You build | Concepts |
|---|---|---|---|---|
| 1 | Architecture | This document | Challenge it | System design, trade-offs |
| 2 | Foundation + **first deploy** | Hello-world on Railway reaching Supabase; CI green | `.gitignore`, health route | venv, pinning, Docker layers, env config, CI/CD |
| 3 | Portfolio frontend | Full site from `data/` | One section, then `SanDiegoClock` | Static generation, components, hydration, responsive design, accessibility |
| 4 | Database | Migrations, roles, privacy test | Write the schema from this spec; run `EXPLAIN` | Normalization, keys, roles, pooling, migrations |
| 5 | Ingestion | `sync` populates the KB idempotently | The chunker | Embeddings, tokenization, chunking, hashing, transactions |
| 6 | Retrieval | Search CLI; eval baseline; threshold set | Cosine by hand; Recall@K and MRR | Vector similarity, ANN vs exact, metrics |
| 7 | Generation | `POST /chat` grounded with sources | Prompt iterations | Prompts, context windows, tokens, retries, hallucination |
| 8 | Chat UI | Streaming, follow-ups | SSE parser | Streaming, client state, query rewriting |
| 9 | Hardening | Eval report, injection tests, limits, tuning | Rate limiter | Security, LLM evaluation, cost and latency work |
| 10 | Launch | Domain, monitoring, final docs | Runbook | Production debugging, operations |

**Teaching rhythm:** for each major feature, your 10-point format: problem → concepts → architecture → options → data flow → incremental build → code walkthrough → run and test → common failures. Deep explanations (what embeddings are, how similarity works, how chunk size affects retrieval) come in the phase that uses them.

**v1 scope**

| In v1 | Later, only if justified |
|---|---|
| Site, chat, streaming, sources | Hybrid search, reranking |
| JSON, Markdown, text ingestion | PDF parsing |
| Stateless conversations | Stored conversations and analytics |
| Exact vector search | HNSW |
| CLI ingestion | Admin API and dashboard |
| `mailto:` | Contact form |
| Railway logs | Sentry, dashboards |
| In-memory rate limiting | Shared limiter |

---

## 18. Technical risks

| Risk | Likelihood | Mitigation |
|---|---|---|
| Supabase free project pauses | **High** | Static site survives; scheduled ping; graceful chat message; Pro if needed |
| Custom role fails through Supavisor | Medium | Test in Phase 2/4; fallback is `SET ROLE` after connect |
| Model IDs change (Gemini is deprecating 2.5 models now) | High over time | IDs in env; eval re-run on upgrade |
| Thin knowledge base gives poor answers | **High** | Your content quality matters more than any tuning. Spend real time on it |
| Rewrite step adds latency | Medium | Only with history; cheapest model; measure |
| Over-refusal from a bad threshold | Medium | Calibrate on data; log rejected chunks |
| Streaming buffered by a proxy | Low–medium | Test in production in Phase 8 |
| Local Python/Node differ from Docker | Low | CI builds the real image |

---

## 19. What I need from you

### Decisions

Defaults apply unless you say otherwise.

1. **Gemini tier.** Default: **free tier for development, paid tier with a budget alert before public launch**, so visitor questions aren't used by Google to improve products. Your call, since it costs real money, though likely under $1/month.
2. **Repo visibility.** It appears private or uncreated. Default: **public at launch**, because a portfolio repo is itself a portfolio piece. That makes everything in `data/` public, so review it first.
3. **Supabase plan.** Default: **free**, accepting the pause risk.
4. **Frontend.** Default: **Next.js static export**. Say so if you'd rather try Astro.
5. **Exercises.** Default: the "You build" items in §17. Tell me if you'd like more, fewer, or to change the balance.

### Accounts (needed by Phase 2)

- Start **Docker Desktop** and check the **Supabase CLI**
- **Supabase** project, in a US West region near Railway if possible
- **Railway** account connected to GitHub
- **Google AI Studio** API key; check your actual limits at `aistudio.google.com/rate-limit`
- Confirm the GitHub repo exists

### Personal content (needed by Phases 3 and 5)

I won't invent any of this. I'm also treating the details in your brief as unconfirmed examples until you verify them, including "Computer Science at WPI" and San Diego as your location.

| File | Needed |
|---|---|
| Identity | Name as displayed, tagline, intro, location for the clock, public email, LinkedIn, GitHub |
| Assets | Profile photo; resume PDF **with phone and home address removed** |
| Education | Degrees, schools, dates, GPA only if you want it shown, coursework, achievements |
| Experience | Roles, employers, dates, responsibilities, measurable results |
| Projects | Per project: problem, stack, implementation, key decisions, features, links. **The most valuable content for the chatbot** |
| Skills | By your seven categories, honest about depth |
| Certifications | Name, issuer, date, credential link |
| Research and goals | Interests, target roles, locations, work authorization **only if you want it public** |
| Personal | Hobbies and background in your own voice; opinions you're comfortable publishing, which become `kind: opinion` |
| FAQ | The 10–15 questions you'd want recruiters to get good answers to |
| Boundaries | Topics the bot should never discuss |
| Design | 2–3 sites you admire |

A resume and a rough brain-dump is a good start. In Phase 3 I'll give you a template for each file so you can fill them in directly.

---

## 20. Verification

**This proposal:** approval, or your changes to §1 and §19.

**End-to-end acceptance for v1**

1. Fresh clone: README setup alone gets the site and API running locally.
2. `which python` resolves inside `backend/.venv`; `.venv` and `.env` aren't tracked.
3. CI passes, including the real Docker build, with no live Gemini calls.
4. Running `sync` twice: second run makes zero embedding calls and changes zero rows.
5. Connected as `chat_reader`: reading `kb.document_chunks` and any write fail with `permission denied`.
6. Follow-up test passes: the master's degree question, then "what projects did he work on during that time?" retrieves the right sources.
7. Unknown question gets a refusal with no invented facts.
8. Injection set: no prompt-rule bypass exposes anything non-public.
9. Eval report shows Recall@K, MRR, faithfulness, and the full-context baseline.
10. Production: site loads with Supabase unreachable; chat fails gracefully; rate limit returns 429 per real client IP; streaming arrives incrementally.
11. Browser bundle search finds no keys or DB strings.

**First step after approval:** Phase 2 only, one piece at a time, explaining as we go. I'll ask before committing or pushing.

---

## Sources

- Gemini [embeddings](https://aistudio.google.com/docs/embeddings), [models](https://aistudio.google.com/docs/models), [pricing](https://aistudio.google.com/docs/pricing), [rate limits](https://aistudio.google.com/docs/rate-limits), [3.6 Flash](https://aistudio.google.com/docs/models/gemini-3.6-flash)
- Supabase [connecting to Postgres](https://supabase.com/docs/guides/database/connecting-to-postgres), [HNSW indexes](https://supabase.com/docs/guides/ai/vector-indexes/hnsw-indexes), [pricing](https://Supabase.io/pricing), [free tier summary (third-party)](https://automationatlas.io/answers/supabase-free-tier-limits-2026/), [Railway connection discussion](https://github.com/orgs/supabase/discussions/43111)
- Railway [pricing plans](https://docs.railway.com/reference/pricing/plans), [IPv6 thread](https://station.railway.com/questions/i-pv6-feature-flag-can-t-connect-to-sup-38b41ca1), [Hobby IPv6 thread](https://station.railway.com/questions/do-hobby-plans-support-ipv6-a5a48bbf)
