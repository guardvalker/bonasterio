# Architecture Research

**Domain:** Multi-AI debate/orchestration backend (LangGraph) + shared Postgres (Supabase) + static PWA frontend, deployed on scale-to-zero serverless (Cloud Run)
**Researched:** 2026-08-20
**Confidence:** HIGH (Cloud Run timeout/CPU behavior, Supabase RLS/Realtime — official docs verified) / MEDIUM (LangGraph Send API specifics, exact graph shape — WebSearch verified against multiple sources, not hands-on tested in this repo)

## Standard Architecture

### System Overview

```
┌────────────────────────────────────────────────────────────────────┐
│                          PWA (GitHub Pages)                          │
│  Static HTML/CSS/JS — vanilla, no build step, same as other BONA    │
│  apps. Supabase JS client (anon key) + fetch() to backend.          │
├───────────────────────────┬───────────────────────┬──────────────────┤
│  Direct to Supabase        │  Direct to Supabase    │  To Backend      │
│  (anon key + RLS,          │  (Realtime subscribe,  │  (POST /debates  │
│  authenticated session)    │  read-only, live)      │  only, JWT auth) │
│  - roles CRUD               │  - debate_rounds        │                  │
│  - debates history/list     │    INSERT events        │                  │
│  - PATCH real budget        │    for active debate_id │                  │
│  - debate_config read        │                        │                  │
└───────────────┬─────────────┴───────────┬─────────────┴────────┬─────────┘
                │                          │                      │
                ▼                          ▲                      ▼
┌───────────────────────────────────────────────────┐   ┌─────────────────────┐
│              Supabase (shared `bonapps` project)    │   │  FastAPI + LangGraph │
│  Postgres + Auth (OTP) + Realtime + PostgREST        │◀──│  Cloud Run (Docker)  │
│  RLS: authenticated-only policies on debate_* tables │──▶│  service_role key    │
│  (anon role has no access beyond auth flow)          │   │  (bypasses RLS)      │
└───────────────────────────────────────────────────┘   └──────────┬──────────┘
                                                                     │
                                                                     ▼
                                                      External LLM APIs (free tier)
                                                      Gemini API, OpenRouter
```

**Core boundary rule:** the PWA talks to Supabase directly for anything that is plain data CRUD scoped to the logged-in user's session (RLS-enforced). It talks to the backend **only** for the one operation that requires a secret (LLM API keys) and multi-step server-side orchestration (the LangGraph run): kicking off a new debate. Everything else — history list, role CRUD, config read, retroactive "real budget" edit, and **live progress while a debate runs** — goes through Supabase directly (REST/PostgREST for reads/writes, Realtime for live updates). This cuts the REST surface from the spec's 7 endpoints down to effectively 1 required backend endpoint, which materially simplifies the backend build.

### Component Responsibilities

| Component | Responsibility | Typical Implementation |
|-----------|----------------|-------------------------|
| PWA (static) | Auth UI (OTP), forms, rendering debate state, Realtime subscription, direct Supabase CRUD for config/history | Vanilla JS + `@supabase/supabase-js` loaded via CDN or bundled, no framework needed given existing BONA pattern |
| Supabase Postgres | System of record for roles, debates, rounds, config; RLS gate; Realtime change feed | 4 tables (`debate_roles`, `debates`, `debate_rounds`, `debate_config`) in shared `public` schema of the `bonapps` project, `debate_` prefix avoids collision with other apps' tables |
| Supabase Auth | OTP email login, issues JWT session used by PWA's anon-key client (elevates to `authenticated` role) and validated by backend | Supabase-managed, same pattern as other BONA apps |
| Supabase Realtime | Pushes `debate_rounds` INSERT (and `debates` UPDATE) events to subscribed PWA clients while a debate is running | `postgres_changes` channel filtered by `debate_id=eq.<id>` |
| FastAPI + LangGraph backend | Orchestrates the debate graph, calls external LLM APIs, writes rounds to Postgres incrementally via `service_role` key, validates caller's Supabase JWT | Single Cloud Run service, Docker container, `POST /debates` as the primary/only required endpoint |
| Cloud Run | Hosts the backend, scale-to-zero, request-scoped CPU by default | One service, default CPU allocation (not always-on), default or slightly raised request timeout |
| External LLM APIs | Actual model calls (Gemini official API, OpenRouter free-tier models) | Called only from backend, keys never reach the client |

## Recommended Project Structure

```
bonasterio/
├── supabase/
│   └── migrations/            # SQL migrations: 4 tables + RLS policies + indexes
├── backend/
│   ├── app/
│   │   ├── main.py            # FastAPI app, JWT auth middleware, route registration
│   │   ├── routes/
│   │   │   └── debates.py     # POST /debates (primary), optional GET for debug
│   │   ├── graph/
│   │   │   ├── state.py       # DebateState TypedDict
│   │   │   ├── nodes.py       # initial round (Send fan-out), adjustment round, arbiter, persistence
│   │   │   └── build.py       # StateGraph wiring
│   │   ├── llm/
│   │   │   ├── gemini.py      # Gemini API client wrapper
│   │   │   └── openrouter.py  # OpenRouter client wrapper (model-agnostic, picks model per role)
│   │   ├── db/
│   │   │   └── supabase_client.py  # service_role client, incremental writes
│   │   └── auth.py            # verifies incoming Supabase JWT (Authorization header)
│   ├── Dockerfile
│   └── requirements.txt
└── pwa/
    ├── index.html              # Nuevo debate + live view
    ├── historial.html
    ├── roles.html              # config CRUD
    ├── js/
    │   ├── supabase-client.js  # shared anon-key client init, same pattern as other BONA apps
    │   ├── debates.js          # POST kickoff + Realtime subscription + render
    │   ├── historial.js
    │   └── roles.js
    └── sw.js                   # PWA service worker, cache versioned per BONA convention
```

### Structure Rationale

- **`backend/app/graph/`** isolates the LangGraph state machine from the FastAPI transport layer — the graph itself has no knowledge of HTTP, which is what makes it reusable for other "AI council" use cases later (explicitly named as a future goal in the spec, even though out of scope this milestone).
- **`backend/app/llm/`** separates provider clients so adding/removing models (per `debate_roles.modelo`) doesn't touch graph logic — a role's `modelo` field maps to a provider+model-id lookup here.
- **`pwa/js/supabase-client.js`** mirrors the existing BONA app pattern (same anon key init style as Toolbox/gastos) — keep consistent to avoid re-deriving auth/session handling per app.
- No `backend/app/routes/` sprawl — resist building CRUD routes for roles/config/history in the backend. That data has no secret dependency and belongs directly on Supabase via RLS.

## Architectural Patterns

### Pattern 1: Backend as the only writer with secrets; Supabase RLS gates everything else

**What:** The backend uses the Supabase `service_role` key (bypasses RLS) to write `debates`/`debate_rounds` rows during a run. The PWA uses the `anon` key + a logged-in session (RLS enforces `authenticated` role) for every other read/write.
**When to use:** Any time a subset of operations require a secret (API keys) or multi-step server logic that can't run client-side, while the rest of the data is plain CRUD.
**Trade-offs:** Simpler backend surface and less latency (no backend hop) for CRUD screens; but two different "identities" write to the same tables (service_role vs. authenticated user), so RLS policies must be written assuming service_role bypasses them entirely — don't rely on RLS to validate data shape coming from the backend, do that in the backend itself.

### Pattern 2: Incremental persistence via graph node side-effects, not LangGraph checkpointing

**What:** Each LangGraph node (`nodo_presupuesto_inicial`, `nodo_ronda_ajuste`, `nodo_arbitro`) writes its own results to `debate_rounds` as a side effect immediately after producing them (a dedicated `nodo_persistencia` step, or persistence inlined at the end of each node), rather than relying on LangGraph's built-in checkpointer (which persists *graph execution state* for resumability, not business rows queryable by the frontend).
**When to use:** Whenever a long-running graph needs to expose partial progress to an external consumer (the PWA) in real time.
**Trade-offs:** Slightly more plumbing per node (each node needs Supabase client access), but this is what makes Realtime progress possible at all — the frontend never needs to know about graph internals, only about rows appearing in `debate_rounds`.

**Example:**
```python
async def nodo_presupuesto_inicial(state: DebateState) -> DebateState:
    sends = [
        Send("consultar_rol", {"role": role, "descripcion": state["descripcion_trabajo"]})
        for role in state["roles"] if not role["es_arbitro"]
    ]
    return sends  # LangGraph fans out dynamically, one node invocation per role

async def consultar_rol(payload: dict) -> dict:
    resultado = await llamar_modelo(payload["role"], payload["descripcion"])
    await supabase.table("debate_rounds").insert({
        "debate_id": payload["debate_id"],
        "role_id": payload["role"]["id"],
        "numero_ronda": 1,
        "contenido": resultado.texto,
        "presupuesto_propuesto_min": resultado.min,
        "presupuesto_propuesto_max": resultado.max,
    }).execute()  # write happens immediately, Realtime pushes it to the PWA
    return {"rondas": [resultado]}
```

### Pattern 3: Dynamic fan-out with LangGraph's `Send` API for a variable number of roles

**What:** Because `debate_roles` is a DB-editable table (Bona can add/remove roles from the PWA), the graph cannot hardcode N parallel branches at construction time. LangGraph's `Send` object lets a node dispatch an arbitrary, runtime-determined number of parallel child invocations, joined back via a state reducer (e.g. `operator.add` on a `rondas: list` field).
**When to use:** Any time the "how many things run in parallel" number comes from data, not from the graph's static topology — exactly this project's "N active roles" case.
**Trade-offs:** More correct than trying to statically define N branches; slightly less common pattern in tutorials than the fixed-branch fan-out, so allow extra time to learn it if the team hasn't used `Send` before. Verify current LangGraph version's `Send` semantics against docs at implementation time (training data may be stale on exact API surface).

### Pattern 4: Blocking POST request as the trigger, Realtime as the actual progress channel (not polling)

**What:** `POST /debates` creates the `debates` row (`estado='en_curso'`), runs the LangGraph graph with `await graph.ainvoke(...)`, and returns only when the whole run finishes (updating `estado='completo'` or `'error'`). The HTTP response itself is *not* how the PWA gets live updates — it fires the POST, then immediately (before or without awaiting the POST's resolution) opens a Realtime subscription on `debate_rounds` filtered by the new `debate_id` and renders rows as they arrive. See "Data Flow" below for why this is the right shape for this specific runtime (15-30s, Cloud Run, single low-volume user) instead of the spec's original polling proposal.
**When to use:** Single-tenant/low-volume internal tools on scale-to-zero platforms where the extra infrastructure of a task queue isn't justified.
**Trade-offs:** Holding an HTTP connection open for 15-30s (worst case maybe up to ~60-90s with several models × several rounds) is unusual for public APIs but is fine for Cloud Run's default 300s (5 min) timeout, and fine for a fetch() call from a PWA the owner is actively watching. If a debate ever needs to run longer than a few minutes (more roles, more rounds, slower free-tier models), raise Cloud Run's `--timeout` (configurable up to 3600s) rather than re-architecting — no need to reach for a task queue at this scale.

## Data Flow

### Request Flow: kicking off and watching a debate

```
[Bona fills form, clicks "Iniciar consejo"]
    ↓
PWA:  fetch POST https://<cloud-run-url>/debates
      { descripcion_trabajo, ... }
      Authorization: Bearer <supabase JWT from session>
    ↓ (fire, don't block UI on the response)
PWA:  supabase.channel(`debate-rounds-pending`)  — actually needs debate_id first;
      simplest: POST returns debate_id fast (see below), THEN subscribe
    ↓
Backend: verify JWT → INSERT debates (estado='en_curso') → return debate_id early?
```

**Important nuance to resolve during planning, not left ambiguous:** the PWA needs a `debate_id` to filter the Realtime subscription *before* rounds start arriving, but Pattern 4 above has the POST block until completion — which would mean the PWA doesn't get `debate_id` until the very end, missing all the live rounds.

**Resolution — split the POST into two effective phases without adding a task queue:**
1. Backend creates the `debates` row (`estado='en_curso'`) and returns `{ debate_id }` **immediately** (fast, no LLM calls yet) — this is a normal quick DB insert, not the slow part.
2. In the same request/response cycle, before writing the HTTP response, the backend needs to actually keep running the graph. Two ways to do this on Cloud Run without losing CPU allocation:
   - **(a) Simplest, recommended for this scale:** don't split into two HTTP calls at all — do the `debates` insert **client-side is wrong (needs service key)**, so instead: PWA opens the Realtime subscription using a **client-generated `debate_id` (UUID)** sent *in* the POST body, subscribes to that channel immediately after firing the POST (before awaiting its response), and the backend uses that same client-supplied ID for the `debates` row and every `debate_rounds` insert. The POST can then legitimately block for the full run — the PWA was already listening on the right channel from the moment it clicked "start," so it doesn't matter that the POST response itself arrives last.
   - **(b) Alternative if client-generated IDs feel unclean:** two backend endpoints — `POST /debates` returns `debate_id` instantly (just the insert), then internally `await`s the graph run using Cloud Run's request-scoped CPU by keeping that *same* request open via a second internal call, OR accept a tiny bit of extra infra and use Cloud Run's always-on CPU billing so a `BackgroundTasks`-style fire-and-forget after early response keeps executing. **Not recommended** here — always-on CPU billing changes Cloud Run's cost model (charges for idle time too) purely to avoid client-generated UUIDs, a bad trade for a single-user internal tool.
3. Recommended: **(a)**. This is a small deviation from the spec's schema note ("`id` uuid, PK") — no schema change needed, Postgres `uuid` PKs accept client-supplied values fine; just don't use `gen_random_uuid() ... DEFAULT` as the *only* way IDs are created — accept an explicit `id` in the insert.

```
[Resolved flow]
PWA:  const debate_id = crypto.randomUUID()
PWA:  supabase.channel(`debate:${debate_id}`)
        .on('postgres_changes', { event: 'INSERT', table: 'debate_rounds', filter: `debate_id=eq.${debate_id}` }, renderRound)
        .on('postgres_changes', { event: 'UPDATE', table: 'debates', filter: `id=eq.${debate_id}` }, renderFinalVerdict)
        .subscribe()
    ↓ (small delay for subscription to be ready — see Realtime pitfall below)
PWA:  fetch POST /debates { id: debate_id, descripcion_trabajo, ... }
    ↓
Backend: verify JWT → insert `debates` row with the given id (estado='en_curso')
         → graph.ainvoke(...) — each node writes its own debate_rounds rows as it finishes
         → update `debates` row (estado='completo', presupuesto_final_*, justificacion_final)
         → HTTP 200 response (PWA can largely ignore the body; UI already updated via Realtime)
    ↓
PWA:  rows arrive live via Realtime throughout the run; final `debates` UPDATE triggers
      "show arbiter verdict" UI regardless of whether POST has resolved yet
```

### Data Flow: everything else (no backend involved)

```
[Historial screen]      PWA → supabase.from('debates').select(...).order('created_at', desc) → render
[Anotar real facturado] PWA → supabase.from('debates').update({ presupuesto_real_facturado }) .eq('id', ...) → RLS checks authenticated
[Roles CRUD]             PWA → supabase.from('debate_roles').insert/update/select → RLS checks authenticated
[Config read]             PWA → supabase.from('debate_config').select().eq('activo', true) → RLS checks authenticated
```

### Key Data Flows

1. **New debate:** PWA generates ID client-side → Realtime subscription opens first → POST triggers backend orchestration → backend writes incrementally with service_role → PWA renders live, independent of POST resolution.
2. **Everything else:** PWA ↔ Supabase directly, RLS-gated, no backend involvement, no extra latency hop.
3. **Backend never reads from the PWA session** — it only trusts its own JWT verification of the caller, then acts as a fully privileged writer via service_role. It does not need to proxy any Supabase reads.

## Scaling Considerations

Given this is a single-user (eventually maybe 2-3 person) internal tool, "scale" here mostly means "don't over-build," not "prepare for 10K users."

| Scale | Architecture Adjustments |
|-------|--------------------------|
| 1 user, a few debates/week (actual target) | Everything above as-is. Default Cloud Run timeout (300s) and default (non-always-on) CPU billing are sufficient. |
| A few team members, several debates/day | No architecture change needed — RLS already gates on `authenticated`, not per-owner, so multiple logged-in users naturally share the same debate history (this is desired per the spec's "por si en el futuro otra persona del equipo la usa"). |
| Debate runs start exceeding ~4-5 min (many roles, many rounds, slow free-tier model latency) | Raise Cloud Run `--timeout` toward its 3600s ceiling first — cheapest fix. Only reach for a task queue (Cloud Tasks, Celery/RQ) if runs start regularly exceeding tens of minutes, which is very unlikely at this project's role/round counts. |
| Free-tier LLM rate limits become the bottleneck (not compute) | This is the realistic first bottleneck, not Cloud Run capacity — Gemini free tier and OpenRouter free models have per-minute/per-day request caps. Mitigate with role-level backoff/retry in `llm/`, not with architecture changes. |

### Scaling Priorities

1. **First real bottleneck:** free-tier LLM API rate limits (Gemini free tier, OpenRouter free models), not Cloud Run or Postgres. Design the `llm/` client wrappers with retry/backoff from the start since this is the actual constraint, not traffic volume.
2. **Second (unlikely to matter):** Realtime `postgres_changes` throughput is fine at this scale — Supabase's per-subscriber authorization check overhead only matters with many concurrent subscribers, and this app has effectively one.

## Anti-Patterns

### Anti-Pattern 1: Polling `GET /debates/{id}` every N seconds (as the original spec proposed)

**What people do:** Have the PWA fire `POST /debates`, get an `id` back immediately, then `setInterval` a `GET /debates/{id}` poll every 2-3s until `estado='completo'`.
**Why it's wrong:** The project's own PROJECT.md already rules out WebSocket in favor of Supabase Realtime ("Supabase Realtime alcanza para el volumen de uso") — polling reintroduces the exact complexity Realtime was chosen to avoid (extra endpoint, client-side interval management, wasted requests when nothing changed), and Realtime gives strictly better UX (rounds appear the instant they're written, not up to N seconds later) for zero extra backend code.
**Do this instead:** Subscribe to `postgres_changes` on `debate_rounds`/`debates` as described in Pattern 4 / Data Flow above. Only fall back to a `GET /debates/{id}` debug endpoint for manual troubleshooting, not as the primary UI mechanism.

### Anti-Pattern 2: Fire-and-forget background task after an instant HTTP response, without always-on CPU

**What people do:** `POST /debates` inserts the row, calls `background_tasks.add_task(run_graph, ...)`, and returns `202` immediately, assuming the graph keeps running in the background like it would on a traditional always-on server.
**Why it's wrong:** Cloud Run only allocates CPU to a container instance while it is actively handling a request, by default. The instant the HTTP response is sent, CPU is throttled — the background task effectively freezes (or executes unpredictably slowly) unless "CPU always allocated" / instance-based billing is explicitly configured, which changes Cloud Run's cost model.
**Do this instead:** Either (a) keep the request open for the full graph duration as in Pattern 4 (simplest, no billing changes, fits comfortably in Cloud Run's timeout budget), or (b) explicitly opt into always-on CPU billing if a future need for true fire-and-forget emerges. Don't reach for (b) by default.

### Anti-Pattern 3: Per-user `owner_id` RLS scoping for what is actually shared team data

**What people do:** Default to a multi-tenant SaaS RLS pattern — every row gets an `owner_id` referencing `auth.users`, policies check `auth.uid() = owner_id`.
**Why it's wrong:** This project isn't multi-tenant SaaS with private-per-user data; it's a single internal tool where "por si en el futuro otra persona del equipo la usa" means *shared* visibility across whichever BONA team members are logged in (same shape as the existing shared-expense app), not data siloed per account. Owner-scoping here would actively break the intended future use case (a teammate logging in should see the same debate history, not a fresh empty account).
**Do this instead:** RLS policies gated on `TO authenticated USING (true)` (any logged-in ecosystem user can read/write these tables), with `anon` role having no policy at all (default-deny) beyond the auth flow itself. See RLS section below.

## Integration Points

### External Services

| Service | Integration Pattern | Notes |
|---------|---------------------|-------|
| Gemini API (official free tier) | Backend-only HTTP client, key in Cloud Run env var | Free tier has documented per-minute/per-day quotas — verify current limits at implementation time, they change |
| OpenRouter (free-tier open models) | Backend-only HTTP client, key in Cloud Run env var, model ID selected per `debate_roles.modelo` | Multiple free models available across different labs (Llama, DeepSeek, etc.) satisfying the "distinct architectures" requirement in one API |
| Supabase (Postgres + Auth + Realtime + PostgREST) | PWA: `@supabase/supabase-js` with anon key + session. Backend: same SDK (or REST) with `service_role` key | Same shared `bonapps` project as other BONA apps — table naming (`debate_` prefix) is the isolation mechanism, not separate schemas |

### Internal Boundaries

| Boundary | Communication | Notes |
|----------|---------------|-------|
| PWA ↔ Supabase (CRUD) | Direct PostgREST calls via supabase-js, anon key, RLS-enforced | No backend hop; covers roles, history, config, retroactive budget edit |
| PWA ↔ Supabase (live progress) | Realtime `postgres_changes` WebSocket subscription | Client-generated `debate_id` used to subscribe before the debate exists, per Pattern 4 |
| PWA ↔ Backend | Single `POST /debates`, `Authorization: Bearer <supabase JWT>` | Backend validates the JWT (signature check against Supabase project JWT secret / JWKS) before running any LLM calls, to prevent the public Cloud Run URL from being an open compute/quota-burning endpoint |
| Backend ↔ Supabase | `service_role` key, bypasses RLS entirely | Backend is the trusted writer; RLS policies are irrelevant to it and must not be relied upon to validate backend-authored data — validate in backend code |
| Backend ↔ External LLM APIs | Direct HTTPS calls from `llm/` wrappers, called from within graph nodes | Secrets never leave the backend; retry/backoff lives here since this is the real rate-limit bottleneck |

## RLS Design (single-user-with-login, shared Postgres project)

**Model: "authenticated-only shared tool," not "multi-tenant owner-scoped."** Every `debate_*` table gets the same shape of policy:

```sql
alter table debate_roles enable row level security;
alter table debates enable row level security;
alter table debate_rounds enable row level security;
alter table debate_config enable row level security;

-- Any logged-in BONA ecosystem user can read/write. No anon access at all.
create policy "authenticated full access" on debates
  for all to authenticated using (true) with check (true);
-- repeat per table
```

- **No `owner_id` column needed** on these tables for v1 — see Anti-Pattern 3. If per-user attribution is ever wanted later (e.g. "who created this debate"), add a nullable `created_by uuid references auth.users` column for display purposes only, not as an RLS filter.
- **`anon` role gets zero policies** on these tables (RLS default-denies when no policy matches) — the anon key is only used for the pre-login auth flow (OTP request/verify), never for table access. This is what stops the shared anon key (which is public, embedded in client JS) from being usable to read/write debate data without logging in.
- **Isolation from other BONA apps in the same project:** achieved by table naming (`debate_` prefix), not separate schemas or separate RLS logic. Since this project's tables use `to authenticated using (true)`, any authenticated user of *any* BONA app sharing this Supabase project could technically read `debates` rows if they knew the table existed — this is an acceptable trade-off for an internal single-org tool where all authenticated users are Bona/team members anyway (verify this assumption is still true before reusing this pattern for a project with untrusted or external users).
- **Backend's `service_role` key bypasses all of the above** by design — it is the trusted writer during a debate run. Never expose `service_role` to the PWA/frontend.
- **JWT verification on the backend:** the `POST /debates` endpoint must independently verify the incoming `Authorization: Bearer <token>` against Supabase's JWT secret/JWKS (not just "does a token exist") before running any LLM calls — this is the only gate preventing an unauthenticated caller from hitting the public Cloud Run URL and burning free-tier LLM quota. Supabase Python client / `PyJWT` can validate this; confirm current recommended verification method against Supabase Auth docs at implementation time (JWT signing key rotation/JWKS endpoints have changed across Supabase Auth versions).

## Suggested Build Order

The spec's proposed order (schema → backend/graph → REST endpoints → PWA) is directionally right but can be refined now that the component boundaries above show most of the PWA doesn't depend on the backend at all:

1. **Schema + RLS** (`supabase/migrations/`): the 4 tables, indexes (`debate_rounds(debate_id)`, `debates(estado)`, `debates(created_at desc)`), and the `authenticated`-only RLS policies above. Fast, no dependencies, and validates the RLS design decision early (test with a real OTP-logged-in session before building anything else on top).
2. **PWA: auth + config/historial/roles screens, direct-to-Supabase only.** These need zero backend — just schema + RLS + the existing BONA OTP auth pattern. Building this early validates the schema and RLS design against a real client before the backend exists, and produces something Bona can already use to hand-manage `debate_roles`/`debate_config` rows manually while the graph is still being built. (This reorders ahead of "backend" relative to the original spec, since nothing here depends on it.)
3. **Backend graph logic, standalone (no FastAPI yet).** Build and test the LangGraph `StateGraph` — dynamic `Send` fan-out for the initial round, adjustment round(s), arbiter node, incremental persistence node — as a plain Python script/test harness against real `debate_roles` rows (now that step 2 lets Bona create them) and real free-tier LLM calls. Validate correctness and timing (confirm the 15-30s estimate; check whether free-tier model latency/rate limits push it higher) before wrapping in HTTP.
4. **FastAPI wrapper + JWT auth middleware + Dockerfile + Cloud Run deploy**, exposing the single `POST /debates`. Deploy to Cloud Run early even with just this one endpoint — confirms cold-start behavior, default timeout headroom, and CORS config against the real GitHub Pages origin before wiring the frontend to it.
5. **PWA: "Nuevo debate" + live view**, wiring the client-generated-UUID + Realtime-subscribe-before-POST flow from Pattern 4. This is last because it's the only piece touching every other component (auth, Supabase Realtime, and the deployed backend).

This ordering front-loads the pieces with fewer dependencies (schema, then the Supabase-only PWA screens) and pushes the piece with the most moving parts (live debate flow tying backend + Realtime + PWA together) to the end, once each dependency has already been validated in isolation.

## Sources

- [Cloud Run: Configure request timeout for services](https://docs.cloud.google.com/run/docs/configuring/request-timeout) — default 300s, configurable up to 3600s (HIGH confidence, official docs)
- [Cloud Run: Billing settings for services](https://docs.cloud.google.com/run/docs/configuring/billing-settings) — CPU throttling after response by default; always-on CPU changes billing model (HIGH confidence, official docs)
- [Google Cloud Blog: Use Cloud Run always-CPU allocation for background work](https://cloud.google.com/blog/topics/developers-practitioners/use-cloud-run-always-cpu-allocation-background-work) (MEDIUM confidence, official blog, corroborates default-throttling behavior)
- [Supabase Docs: Postgres Changes (Realtime)](https://supabase.com/docs/guides/realtime/postgres-changes) (HIGH confidence, official docs)
- [Supabase Docs: Subscribing to Database Changes](https://supabase.com/docs/guides/realtime/subscribing-to-database-changes) (HIGH confidence, official docs) — includes noted race condition where `SUBSCRIBED` status can fire slightly before the replication listener is fully ready; relevant to the "subscribe before POST" pattern above, mitigate with a small delay or by having the backend's `debates` insert (not the graph run) happen after a short buffer
- [Supabase RLS Best Practices: Production Patterns (makerkit.dev)](https://makerkit.dev/blog/tutorials/supabase-rls-best-practices) (MEDIUM confidence, third-party but widely cited, cross-checked against Supabase's own owner-pattern guidance)
- [LangGraph `Send` API reference](https://reference.langchain.com/python/langgraph/types/Send) (MEDIUM confidence — WebSearch-verified via LangChain reference docs and multiple tutorial sources; recommend a quick current-docs check at implementation time since LangGraph's API has moved fast)
- [Map-Reduce with the Send() API in LangGraph (Medium)](https://medium.com/ai-engineering-bootcamp/map-reduce-with-the-send-api-in-langgraph-29b92078b47d) (MEDIUM confidence, corroborating tutorial)

---
*Architecture research for: bonasterio (multi-AI debate council backend)*
*Researched: 2026-08-20*
