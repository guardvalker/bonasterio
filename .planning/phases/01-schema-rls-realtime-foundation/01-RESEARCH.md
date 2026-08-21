# Phase 1: Schema, RLS & Realtime Foundation - Research

**Researched:** 2026-08-20
**Domain:** Supabase/Postgres schema design, Row Level Security, Realtime publication config, RLS/Realtime verification tooling
**Confidence:** HIGH

## Summary

This phase is pure Postgres DDL + Supabase platform config against the shared `bonapps` project — no application code, no LangGraph, no PWA. The technical shape is well-precedented: two sibling BONA-ecosystem apps in the same Supabase project (`lista-super`, `bona-consumos`) already ship this exact pattern (RLS-gated tables + `supabase_realtime` publication), applied via a hand-run `supabase/schema.sql` rather than the Supabase CLI's `migrations/` + `db push` workflow. That precedent is a stronger signal than `research/ARCHITECTURE.md`'s suggested `supabase/migrations/` folder structure, because it's what's actually running in production for this exact Supabase project today, by the same author, and it sidesteps a real environment gap: the Supabase CLI is not installed locally (only reachable via `npx supabase@latest`, confirmed working, v2.115.0) and no `SUPABASE_ACCESS_TOKEN`/project-linking credentials exist in this environment for `supabase link`.

Two non-obvious, cross-verified findings materially change what "correct" looks like for this phase, beyond what `research/ARCHITECTURE.md` and `research/PITFALLS.md` already cover:

1. **RLS alone is not sufficient — the `authenticated` role also needs an explicit `GRANT`.** Both sibling schema.sql files (`lista-super`, `bona-consumos`) pair every `enable row level security` block with an explicit `grant select, insert, update, delete on <tables> to authenticated;`, with an inline comment explaining they don't want to depend on Supabase's default privilege setup. This is not mentioned in `research/PITFALLS.md`'s Pitfall 7. Carry this pattern forward for D-05.
2. **`SELECT`-vs-write RLS rejection behavior differs**, which matters for how the phase's verification test is written and interpreted against the success criterion "pedidos anónimos son rechazados por RLS": when no RLS policy matches a role, `SELECT` silently returns an **empty result set** (HTTP 200, `[]`), not an error — but `INSERT`/`UPDATE` raise an explicit `42501 new row violates row-level security policy` error. A verification script must assert on these two different shapes, not assume both fail the same way. [CITED: PostgreSQL RLS docs + Supabase/PostgREST community threads, cross-verified]

**Primary recommendation:** Write one idempotent `supabase/schema.sql` (four `create table if not exists` blocks reflecting D-01–D-04, `enable row level security` + explicit `grant ... to authenticated` + `for all to authenticated using (true) with check (true)` policies per D-05, and `alter publication supabase_realtime add table debates, debate_rounds;` per D-06, all in one file/one paste-and-run), apply it by hand via the Supabase Studio SQL Editor against `bonapps` (matching the established ecosystem workflow — no CLI link needed), then verify with a small `supabase-py`-based pytest script that checks: anon SELECT returns `[]`, anon INSERT raises `42501`, two independently-authenticated password-based test users can both read/write all four tables, and a live `postgres_changes` subscription on `debate_rounds` receives an INSERT event within a short timeout.

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Table schema, FKs, indexes (4 tables) | Database / Storage | — | Pure Postgres DDL, no app logic involved |
| RLS policies (shared-team access gate) | Database / Storage | API / Backend | Postgres/PostgREST enforces at the row level; the future backend's `service_role` key is a deliberate bypass, which is a backend-tier trust decision documented here but not implemented until Phase 3/4 |
| `authenticated`/`anon` role grants | Database / Storage | — | Postgres `GRANT` is a prerequisite layer beneath RLS, not app logic |
| Realtime publication membership (`debates`, `debate_rounds`) | Database / Storage | Browser / Client | Enabling publication is DB-level config; the consumer of the resulting events is the future PWA (Phase 2+), not built this phase |
| Verification (anon-reject / authenticated-shared / realtime-event tests) | Database / Storage (via test client) | — | Exercises the above from outside — no production app code, a disposable test harness only |

## User Constraints (from CONTEXT.md)

<user_constraints>

### Locked Decisions

- **D-01:** `debate_roles` gets two columns — `proveedor` (text: `gemini` | `openrouter`) and `modelo_id` (free-text model identifier, e.g. `gemini-2.5-flash`, `meta-llama/llama-3.1-8b-instruct:free`) — instead of the original spec's single `modelo` field (which assumed `claude`/`gpt`/`gemini` values, superseded by the Gemini+OpenRouter decision in PROJECT.md). Splitting into two columns makes backend provider routing explicit and gives the Phase 2 role-CRUD UI a clean provider dropdown + free model-id field, rather than parsing a convention out of one string.
- **D-02:** `debates` gets a nullable `created_by uuid references auth.users` column, **display-only** — "who started this debate" — never used in an RLS policy. Cheap to add now; painful to backfill later if a teammate starts using the tool.
- **D-03:** All budget/money columns (`debates.presupuesto_final_min/max`, `debates.presupuesto_real_facturado`, `debate_rounds.presupuesto_propuesto_min/max`) use `numeric` with no fixed precision/scale — not `integer`.
- **D-04:** `debate_roles.orden` is kept, but reencuadrado as **PWA display order** (which role's card appears first in the UI), not "sequential intervention order." Downstream (Phase 2 UI, Phase 3 engine) should not read `orden` as execution order.
- **D-05 (RLS model, re-confirmed, not re-opened):** Shared-team RLS on all 4 tables: `for all to authenticated using (true) with check (true)`. `anon` role gets **zero** policies (default-deny). No `owner_id` column for isolation (D-02's `created_by` is explicitly display-only, never a policy filter). **Known conflicting advice to ignore:** `research/PITFALLS.md` Pitfall 7's generic `owner_id`-scoping advice is superseded by this project's deliberate shared-data decision — do not apply it.
- **D-06 (Realtime publication):** `debates` and `debate_rounds` must be added to the `supabase_realtime` publication (`ALTER PUBLICATION supabase_realtime ADD TABLE ...`) in the **same migration** as their RLS policies. `debate_roles` and `debate_config` are NOT required in the publication for this phase.

### Claude's Discretion

- Exact index set beyond `research/ARCHITECTURE.md`'s suggestion (`debate_rounds(debate_id)`, `debates(estado)`, `debates(created_at desc)`) — add more only if a concrete need shows up during planning.
- `debate_config` table shape: build columns per the spec (`nombre`, `cantidad_rondas`, `roles_incluidos`, `activo`); only one active row needed for v1; no preset-switching UI in scope.
- `estado` column on `debates`: plain `text` with a `CHECK` constraint (`en_curso`/`completo`/`error`) vs. a Postgres enum type — implementation detail, no user-facing impact.

### Deferred Ideas (OUT OF SCOPE)

None — discussion stayed within phase scope.

</user_constraints>

## Phase Requirements

<phase_requirements>

| ID | Description | Research Support |
|----|-------------|------------------|
| AUTH-03 | Todas las cuentas logueadas comparten el mismo historial de debates y roles configurados (sin aislamiento por cuenta) | D-05's `for all to authenticated using (true) with check (true)` policy (no `owner_id`) directly implements this. Verification section below specifies a two-independent-authenticated-users test to prove shared visibility, not just single-user happy path. |

</phase_requirements>

## Project Constraints (from CLAUDE.md)

- **Database:** Supabase compartido del proyecto `bonapps` (mismo que las demás apps BONA) — not a separate project. This phase's tables must be created in that shared project, using a `debate_` name prefix for isolation-by-naming (matching `ls_`/`gc_` prefixes of sibling apps).
- **Auth:** OTP por email vía Supabase, no magic link (not directly exercised by this backend-only phase, but the `authenticated`/`anon` role split this phase's RLS relies on comes from that same OTP-based session).
- **`supabase-py`** pinned to `2.31.x` in the project's established stack (CLAUDE.md's Recommended Stack table) — use this for any verification script, don't introduce a different Supabase Python client.
- No project skills found in `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` — none apply.
- GSD workflow enforcement: file-changing work in this phase should happen through the `/gsd:execute-phase` (or equivalent) entry point, not ad hoc.

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| Supabase Postgres (hosted, `bonapps` project) | Server version auto-tracked by Supabase (no self-managed pinning) | System of record, RLS engine, Realtime publication source | Already the ecosystem's shared DB; no separate provisioning needed. [VERIFIED: STACK.md, already-established project decision] |
| `pgcrypto` extension | Bundled/pre-enabled on Supabase | `gen_random_uuid()` for all 4 tables' `id` PKs | Both sibling apps (`lista-super`, `bona-consumos`) declare `create extension if not exists "pgcrypto";` at the top of `schema.sql` even though Supabase enables it by default — cheap defensive idempotency. [VERIFIED: codebase, `/home/bona/lista-super/supabase/schema.sql:11`, `/home/bona/bona-consumos/supabase/schema.sql:21`] |
| `supabase-py` | 2.31.x (confirmed current on PyPI: 2.31.0, checked 2026-08-20) | Verification script: anon/authenticated CRUD calls + Realtime `postgres_changes` subscription | Matches the project's already-locked backend stack choice (CLAUDE.md); no reason to introduce a second client library for a throwaway test script. [VERIFIED: PyPI registry `https://pypi.org/pypi/supabase/json`, confirms STACK.md's version claim] |

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `pytest` + `pytest-asyncio` | Already the project's chosen test tooling (CLAUDE.md Development Tools) | Structuring the verification script as automated, repeatable tests rather than a one-off script | Use now, in this phase, even though the rest of the pytest suite (LangGraph nodes, FastAPI) doesn't exist yet — this phase is the natural place for the first `tests/` directory and `pyproject.toml`/`pytest.ini`. |
| Supabase CLI (`supabase`, npm-distributed) | 2.115.0 (confirmed via `npm view supabase version`, checked 2026-08-20) | Optional — only needed if the planner chooses the CLI `migrations/` + `db push` workflow instead of the established hand-run `schema.sql` pattern | Not installed locally; reachable via `npx supabase@latest <cmd>` (confirmed working in this environment). Requires `supabase login` (interactive OAuth) + `supabase link --project-ref <ref>` before `db push` can reach `bonapps` — neither credential is present in this environment. **Recommendation: skip the CLI for this phase**, follow the established `schema.sql` convention instead (see Architecture Patterns). |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| Hand-run `supabase/schema.sql` (idempotent, pasted into Studio SQL Editor) | Supabase CLI `supabase/migrations/<timestamp>_name.sql` + `supabase db push` | The CLI gives proper migration history/versioning and matches `research/ARCHITECTURE.md`'s suggested project structure, but requires CLI auth/project-link credentials not present in this environment, and diverges from what both sibling BONA apps in this same `bonapps` project actually do. Also, `supabase db diff`'s declarative-schema mode explicitly does **not** capture `alter publication ... add table ...` statements [CITED: supabase.com/docs/guides/local-development/declarative-database-schemas] — so even the CLI path would need a hand-written versioned migration file for D-06's publication step regardless, which weakens the CLI's main advantage for this specific phase. If the project later wants CLI-managed migrations for its own sake, that's a reasonable standalone decision, but nothing about *this* phase requires it. |
| `for all ... using (true) with check (true)` (D-05, single policy per table) | Separate `for select`/`for insert`/`for update`/`for delete` policies (the pattern both sibling apps actually use, because their policies aren't all identical) | D-05 is explicit and locked — a single `for all` policy is simpler and correct *because* every operation gets the identical `using(true)`/`with check(true)` condition for this phase's shared-team model. Sibling apps only split policies because their conditions differ per operation (e.g., "anyone in the group can update, only the creator can delete") — not applicable here. |

**Installation:**
```bash
# Verification script env (local, this repo) — pip is not on PATH in this environment;
# bootstrap via venv (confirmed working: python3 -m venv + ensurepip both available)
python3 -m venv .venv
source .venv/bin/activate
pip install supabase pytest pytest-asyncio

# Optional, only if the CLI path is chosen instead of schema.sql:
npx supabase@latest --version   # confirmed reachable, v2.115.0, no local install needed
```

**Version verification:** `npm view supabase version` → `2.115.0` (npm-distributed Supabase CLI, checked 2026-08-20). `curl https://pypi.org/pypi/supabase/json` → `2.31.0` (matches STACK.md's `2.31.x` claim, checked 2026-08-20). Both confirmed directly against their registries in this research session.

## Package Legitimacy Audit

> This phase does not introduce any *new* external package that isn't already part of the project's locked stack (CLAUDE.md). `supabase-py`, `pytest`, `pytest-asyncio` were already vetted with HIGH confidence in prior stack research (verified via PyPI directly). No new attack surface is added here.

**slopcheck unavailable:** `pip`/`pip3`/`pipx` are not on PATH in this research environment (`python3 -m pip` → "No module named pip"), and `pip install slopcheck` could not be attempted. Per the graceful-degradation protocol, all packages below are marked `[ASSUMED]` even though their registry existence and versions were independently confirmed via direct PyPI/npm queries in this session (registry existence alone doesn't upgrade the tag — see provenance rule).

| Package | Registry | Age | Downloads | Source Repo | slopcheck | Disposition |
|---------|----------|-----|-----------|-------------|-----------|-------------|
| `supabase` (Python) | PyPI | Established (multi-year, official Supabase org package) | High (official SDK) | github.com/supabase/supabase-py | not run — unavailable | `[ASSUMED]` — already locked in CLAUDE.md's stack from prior research; keep, no new risk introduced |
| `pytest` | PyPI | Established (multi-year, foundational Python testing tool) | Very high | github.com/pytest-dev/pytest | not run — unavailable | `[ASSUMED]` — de facto standard, already project's chosen tool |
| `pytest-asyncio` | PyPI | Established, official pytest ecosystem plugin | High | github.com/pytest-dev/pytest-asyncio | not run — unavailable | `[ASSUMED]` — already project's chosen tool |
| `supabase` (CLI, npm) | npm | Established (official Supabase org package) | High | github.com/supabase/cli | not run — unavailable | `[ASSUMED]` — optional, only needed if CLI path chosen over `schema.sql` |

**Packages removed due to slopcheck `[SLOP]` verdict:** none (slopcheck did not run).
**Packages flagged as suspicious `[SUS]`:** none.

*slopcheck was unavailable at research time (no `pip`/`pip3`/`pipx` in this environment). The planner must gate the actual `pip install supabase pytest pytest-asyncio` step (if a verification-script task is added) behind a `checkpoint:human-verify` task, even though these are well-established official packages already used elsewhere in the project's locked stack.*

## Architecture Patterns

### System Architecture Diagram

```
                          ┌─────────────────────────────────────────┐
                          │   Supabase "bonapps" project (hosted)    │
                          │                                           │
   [1] SQL Editor paste   │   ┌───────────────────────────────────┐  │
   (Studio dashboard,     │───▶  Postgres: 4 debate_* tables       │  │
   manual, one-time) ─────┼──▶│  RLS enabled + policies + grants   │  │
                          │   │  supabase_realtime publication     │  │
                          │   └───────────────┬───────────────────┘  │
                          │                   │                       │
                          │        PostgREST  │  Realtime server      │
                          │        (REST API) │  (WebSocket)          │
                          └────────┬──────────┴───────────┬───────────┘
                                   │                       │
                    [2a] anon key           [2b] authenticated
                    SELECT/INSERT            session (2 separate
                    (expect: empty           test users) SELECT/
                    result / 42501)          INSERT/UPDATE on all
                                              4 tables (expect: all
                                              succeed, shared visibility)
                                   │                       │
                                   │          [2c] postgres_changes
                                   │          subscription on
                                   │          debate_rounds, expect
                                   │          INSERT event delivered
                                   ▼                       ▼
                          ┌─────────────────────────────────────────┐
                          │   Verification script (pytest, this     │
                          │   phase only — throwaway test harness,  │
                          │   no production code) — supabase-py     │
                          └─────────────────────────────────────────┘
```

A reader can trace: schema/RLS/publication config goes in once via the Studio SQL Editor (step 1), then the verification script exercises it from three angles — anon rejection (2a), cross-account shared authenticated access (2b), and live Realtime delivery (2c) — proving all three success criteria without any production app code existing yet.

### Recommended Project Structure

```
bonasterio/
├── supabase/
│   └── schema.sql              # single idempotent file: tables, RLS, grants, publication
└── verify/                     # phase-1-only, throwaway verification harness
    ├── conftest.py             # loads env vars, builds anon/service_role/test-user clients
    └── test_phase1_rls_realtime.py
```

**Note on divergence from `research/ARCHITECTURE.md`'s suggested structure:** that document proposes `supabase/migrations/` (CLI-style). This research recommends the flat `supabase/schema.sql` file instead, matching the two sibling apps already running in the same `bonapps` project (see Alternatives Considered above) and avoiding a CLI-auth dependency this environment doesn't have configured. If the planner has a reason to prefer CLI-managed migrations anyway (e.g., wanting proper migration history from day one), that's a valid call — just note it's a deliberate departure from existing ecosystem precedent, not a research-recommended default.

### Pattern 1: Idempotent single-file schema, applied by hand via Studio SQL Editor

**What:** One `supabase/schema.sql` with `create table if not exists`, `create index if not exists`, `create extension if not exists`, run once by pasting the whole file into the `bonapps` project's Database → SQL Editor → New query → Run. This is the exact workflow both `lista-super` and `bona-consumos` use today in this same shared project.
**When to use:** Any schema change to a Supabase project where the CLI isn't already wired up with link/auth credentials, and where a single human (Bona) is the one applying changes.
**Caveat — not everything in the file is safely re-runnable:** `create table/index/extension if not exists` are idempotent, but `create policy` has **no** `IF NOT EXISTS` form in Postgres, and `alter publication ... add table ...` has no `IF NOT EXISTS` form either — both will error if the file is pasted a second time after the first successful run. `bona-consumos/supabase/schema.sql` handles this for policies via `drop policy if exists "x" on table; create policy "x" on table ...;` before any policy that might already exist. Apply the same pattern here, and guard the publication statement with a catalog check (Pattern 2 below) if idempotent re-run safety matters for this phase's workflow.

**Example (adapted from `/home/bona/bona-consumos/supabase/schema.sql`):**
```sql
-- Source: established pattern, /home/bona/lista-super/supabase/schema.sql and
-- /home/bona/bona-consumos/supabase/schema.sql (same bonapps project)
create extension if not exists "pgcrypto";

create table if not exists debate_roles (
  id uuid primary key default gen_random_uuid(),
  nombre text not null,
  proveedor text not null check (proveedor in ('gemini', 'openrouter')),
  modelo_id text not null,
  system_prompt text not null,
  es_arbitro boolean not null default false,
  orden int not null default 0,
  activo boolean not null default true,
  created_at timestamptz not null default now()
);
```

### Pattern 2: RLS + explicit GRANT, always paired (not RLS alone)

**What:** `enable row level security` gates *row visibility/mutation*, but the `authenticated` (or `anon`) Postgres role still needs base table-level privileges via `GRANT`, or RLS never even gets evaluated — the query fails on privilege before it reaches the policy layer (or, depending on Supabase project setup, "succeeds" only because Supabase's default privilege template already grants it — don't rely on that being true, be explicit).
**When to use:** Every table this phase touches, every time RLS is enabled.
**Source:** `/home/bona/bona-consumos/supabase/schema.sql:146-151` — explicit inline comment: *"Grants explícitos de nivel tabla, para no depender de que el proyecto Supabase tenga configurados los default privileges esperados — RLS por sí sola no alcanza si el rol `authenticated` no tiene ni el permiso base."*

**Example:**
```sql
alter table debate_roles enable row level security;
alter table debates enable row level security;
alter table debate_rounds enable row level security;
alter table debate_config enable row level security;

-- Explicit, don't rely on Supabase project defaults (verified pattern from
-- two sibling apps in this same bonapps project)
grant select, insert, update, delete on
  debate_roles, debates, debate_rounds, debate_config
to authenticated;

-- D-05: shared-team model, identical for every operation on every table
create policy "debate_roles_authenticated_all" on debate_roles
  for all to authenticated using (true) with check (true);
create policy "debates_authenticated_all" on debates
  for all to authenticated using (true) with check (true);
create policy "debate_rounds_authenticated_all" on debate_rounds
  for all to authenticated using (true) with check (true);
create policy "debate_config_authenticated_all" on debate_config
  for all to authenticated using (true) with check (true);

-- anon gets zero policies on these tables — RLS default-denies. Do not add
-- any policy `to anon` here (that's the entire point of D-05's isolation).
```

### Pattern 3: Realtime publication, guarded against double-run errors

**What:** `alter publication supabase_realtime add table ...` has no `IF NOT EXISTS` clause and will raise `ERROR: relation "debates" is already member of publication "supabase_realtime"` if the schema file is ever re-applied. Guard it with a catalog check against `pg_publication_tables` if the file needs to be safely re-runnable (matching the idempotency of the rest of the file); otherwise a plain statement is fine for a true one-time apply.
**When to use:** D-06's publication step, in the same file/paste as the RLS policies above.
**Example:**
```sql
-- D-06: only debates and debate_rounds — NOT debate_roles or debate_config
-- (no live-editing use case for those tables this phase)
do $$
begin
  if not exists (
    select 1 from pg_publication_tables
    where pubname = 'supabase_realtime' and tablename = 'debates'
  ) then
    alter publication supabase_realtime add table debates;
  end if;
  if not exists (
    select 1 from pg_publication_tables
    where pubname = 'supabase_realtime' and tablename = 'debate_rounds'
  ) then
    alter publication supabase_realtime add table debate_rounds;
  end if;
end $$;
```
[CITED: `pg_publication_tables` is a standard PostgreSQL system catalog view, https://www.postgresql.org/docs/current/view-pg-publication-tables.html]

### Anti-Patterns to Avoid

- **Adding an `owner_id` column "just in case," per `research/PITFALLS.md` Pitfall 7's generic advice.** Explicitly overridden by D-05/CONTEXT.md — do not add it, do not scope any policy to it. `created_by` (D-02) is a separate, display-only column and must never appear in a `using`/`with check` clause.
- **Relying on RLS policies alone without the explicit `GRANT`** (Pattern 2) — works by accident if Supabase's default privilege template happens to already cover it, breaks silently/confusingly if it doesn't. Both sibling apps in this same project chose to be explicit; do the same.
- **Assuming anon rejection means "an error every time."** SELECT with no matching policy returns an empty result set, not an error — see Common Pitfalls below. A verification script that only checks `assert response raises Exception` for the anon SELECT case will falsely fail (or worse, falsely pass if written loosely) — check for `len(data) == 0` on reads, and an explicit `42501`/RLS-violation exception on writes.
- **Testing RLS only against a single authenticated user.** AUTH-03 explicitly requires *shared* visibility across accounts — a test that only proves "the logged-in owner can see their own data" (the Pitfall 7 checklist's minimum bar) does not prove AUTH-03; it must prove a *second, independent* authenticated account sees and can modify the *first* account's rows.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| "Is this table readable/writable by anyone logged in, and rejected otherwise" | A custom middleware/backend check before hitting Supabase | Postgres RLS policies (D-05) + PostgREST's automatic enforcement | This is exactly what RLS + PostgREST are for; no backend exists in this phase, and building an app-layer gate would duplicate what Postgres already does correctly, and would need to run somewhere (defeating "no backend, no PWA" phase scope). |
| Live "did the Realtime event actually arrive" verification | A polling loop that repeatedly queries the table to infer whether Realtime "worked" | A real `postgres_changes` subscription (Pattern in Code Examples below) with an `asyncio.Event`/callback, awaited with a timeout | Polling to verify a push mechanism defeats the purpose of testing it — it would pass even if Realtime were completely broken and the PWA silently fell back to nothing, since the poll would still see the row via plain SELECT. |
| Cross-account RLS testing | Manually crafting/signing a JWT with `role: authenticated` and a fake `sub` | `supabase-py`'s `auth.admin.create_user()` (service_role) + `sign_in_with_password()` for two real test accounts | Produces genuine, correctly-signed session tokens exactly like production OTP-derived sessions do, without needing to touch Supabase's JWT secret/JWKS directly or keep that logic in sync with Supabase Auth's current signing scheme (which `research/ARCHITECTURE.md` already flags as something that "has changed across Supabase Auth versions" — don't add a second place where that assumption could go stale). |

**Key insight:** Everything this phase needs to prove (access control, live push) has a first-class Supabase/Postgres primitive already built for it. The only code this phase should produce is DDL and a disposable verification script — resisting the urge to write any kind of custom auth-checking or polling logic is the main "don't hand-roll" lesson here.

## Common Pitfalls

### Pitfall 1: RLS enabled but no explicit GRANT (new finding, not in `research/PITFALLS.md`)

**What goes wrong:** Policies are written correctly, RLS is enabled, but queries from the `authenticated` role still fail (or — more dangerously — silently succeed only because of an unverified default privilege assumption).
**Why it happens:** RLS and table-level `GRANT` are two independent Postgres permission layers; people assume enabling RLS is the complete access-control story.
**How to avoid:** Always pair `enable row level security` with an explicit `grant select, insert, update, delete on <table> to authenticated;` (Pattern 2). Confirmed as the established convention in two sibling BONA apps already running against `bonapps`.
**Warning signs:** Authenticated requests failing with a generic permission error even though the policy's `using`/`with check` clause looks correct; or, inversely, unexplained access working before RLS was even fully set up (a sign default privileges are doing more than assumed, worth confirming explicitly rather than trusting).

### Pitfall 2: Anon "rejection" isn't always an exception (new finding)

**What goes wrong:** A verification script written to `assert raises Exception` for every anon request against a shared-team-RLS table will incorrectly fail (or pass for the wrong reason) on the SELECT case.
**Why it happens:** Postgres RLS's SELECT behavior when no policy matches is to filter to zero rows (a valid, successful, empty response) — not to error. INSERT/UPDATE, in contrast, must raise an explicit `42501` error because there's no way to "silently discard" a write attempt the way a read can silently return nothing. [CITED: PostgreSQL RLS design rationale, cross-verified via postgresql.org docs and Supabase/PostgREST community discussion, 2026-08-20]
**How to avoid:** Write two distinct assertions in the verification script: anon `SELECT` → `len(response.data) == 0`; anon `INSERT` → exception raised, error code `42501` (or PostgREST's mapped equivalent).
**Warning signs:** A verification script that "passes" the anon-read test simply because it never checked the returned row count, only whether an exception was thrown.

### Pitfall 3: RLS skipped or under-scoped because "it's just for one user" — carried forward from `research/PITFALLS.md` Pitfall 7 [CITED: `.planning/research/PITFALLS.md:132-149`]

**What goes wrong:** The PWA (future phase) ships the public Supabase anon key in client JS. If RLS is off or too broad, anyone with that key can read/write the entire debate history.
**Why it happens:** "Single user" gets conflated with "small security surface," but the actual boundary is "anyone with the public anon key," a much larger set.
**How to avoid:** RLS enabled + default-deny for `anon` on all four tables, per D-05 (already locked). **Important — this project's specific resolution differs from Pitfall 7's generic advice:** Pitfall 7 recommends adding an `owner_id`-scoped policy "even for solo-user apps." That recommendation is explicitly superseded by D-05's shared-team model — do not add `owner_id` scoping. What *does* still apply from Pitfall 7: test RLS by attempting reads/writes as a different (or anonymous) user and confirming the expected behavior, not just the happy path as one logged-in account.
**Warning signs:** Any table showing "RLS disabled" in the Supabase dashboard; a policy with `USING (true)` and no accompanying anon-rejection test.

### Pitfall 4: Realtime silently returns nothing because the table isn't in the publication — carried forward from `research/PITFALLS.md` Pitfall 8 [CITED: `.planning/research/PITFALLS.md:153-168`]

**What goes wrong:** RLS can be perfectly correct and Realtime still delivers nothing, because `postgres_changes` events require the table to be explicitly added to the `supabase_realtime` publication — a separate, easy-to-forget step. Failure mode is silent: no error, subscription just never fires.
**Why it happens:** RLS and Realtime publication are both "permission-adjacent" Supabase concepts that get conflated.
**How to avoid:** D-06 (locked) — add `debates` and `debate_rounds` to the publication in the same schema file as their RLS policies (Pattern 3 above). Verify in the Supabase dashboard's Database → Replication settings after applying, in addition to the scripted verification test.
**Warning signs:** The verification script's Realtime test times out waiting for an event even though a direct SELECT confirms the row was inserted.

### Pitfall 5: Idempotent-DDL habits don't extend to `CREATE POLICY` or `ALTER PUBLICATION ... ADD TABLE`

**What goes wrong:** Following the sibling apps' `create table/index if not exists` convention throughout, then reflexively writing the RLS policies and publication statement the same way, only to have the second application of the file error out.
**Why it happens:** Postgres has no `CREATE POLICY IF NOT EXISTS` and no `ADD TABLE IF NOT EXISTS` form for publications.
**How to avoid:** `drop policy if exists "name" on table; create policy "name" on table ...;` for policies (established sibling-app pattern); a `pg_publication_tables` catalog-existence check (Pattern 3) for the publication statement, if re-run safety matters for this phase's workflow. If the file is truly only ever run once, this is lower priority — but the failure mode (a partial re-paste erroring out partway through) is confusing enough to be worth the small amount of extra SQL.
**Warning signs:** Re-running the schema file after a partial failure produces `policy "x" already exists` or `relation "debates" is already member of publication` errors that look alarming but are actually harmless re-run artifacts, not new problems.

## Code Examples

### Full verification script shape (pytest + supabase-py)

```python
# Source: pattern synthesized from Supabase official docs (auth-admin-createuser,
# auth-signinwithpassword, python/subscribe) + PostgreSQL RLS SELECT-vs-INSERT
# behavior, cross-verified 2026-08-20. This is a Phase-1-only throwaway test
# harness — not production code.
import asyncio
import os
import pytest
from supabase import create_client, acreate_client  # sync + async clients

SUPABASE_URL = os.environ["SUPABASE_URL"]
SUPABASE_ANON_KEY = os.environ["SUPABASE_ANON_KEY"]
SUPABASE_SERVICE_KEY = os.environ["SUPABASE_SERVICE_KEY"]


def test_anon_select_returns_empty_not_error():
    anon = create_client(SUPABASE_URL, SUPABASE_ANON_KEY)
    response = anon.table("debates").select("*").execute()
    assert response.data == []  # RLS default-deny filters to zero rows, no exception


def test_anon_insert_raises_rls_violation():
    anon = create_client(SUPABASE_URL, SUPABASE_ANON_KEY)
    with pytest.raises(Exception) as exc_info:
        anon.table("debates").insert({"descripcion_trabajo": "should fail"}).execute()
    assert "row-level security" in str(exc_info.value).lower()


def test_two_independent_authenticated_users_share_visibility():
    admin = create_client(SUPABASE_URL, SUPABASE_SERVICE_KEY)
    # Create two disposable test accounts via admin API — bypasses OTP email
    # delivery, produces real signed sessions identical in shape to production.
    for email, pw in [("phase1-test-a@example.com", "test-pass-A-123!"),
                       ("phase1-test-b@example.com", "test-pass-B-123!")]:
        admin.auth.admin.create_user({"email": email, "password": pw, "email_confirm": True})

    user_a = create_client(SUPABASE_URL, SUPABASE_ANON_KEY)
    user_a.auth.sign_in_with_password({"email": "phase1-test-a@example.com", "password": "test-pass-A-123!"})
    user_b = create_client(SUPABASE_URL, SUPABASE_ANON_KEY)
    user_b.auth.sign_in_with_password({"email": "phase1-test-b@example.com", "password": "test-pass-B-123!"})

    inserted = user_a.table("debates").insert(
        {"descripcion_trabajo": "cross-account visibility test"}
    ).execute()
    debate_id = inserted.data[0]["id"]

    seen_by_b = user_b.table("debates").select("*").eq("id", debate_id).execute()
    assert len(seen_by_b.data) == 1  # user B sees user A's row — proves AUTH-03, no owner_id isolation

    updated = user_b.table("debates").update({"estado": "completo"}).eq("id", debate_id).execute()
    assert updated.data[0]["estado"] == "completo"  # user B can also write it


@pytest.mark.asyncio
async def test_realtime_delivers_insert_event():
    supabase = await acreate_client(SUPABASE_URL, SUPABASE_SERVICE_KEY)
    event_received = asyncio.Event()

    def on_insert(payload):
        event_received.set()

    channel = supabase.channel("phase1-verify")
    await (
        channel
        .on_postgres_changes("INSERT", schema="public", table="debate_rounds", callback=on_insert)
        .subscribe()
    )
    await asyncio.sleep(1)  # let subscription fully establish before writing

    await supabase.table("debate_rounds").insert({
        "debate_id": "<a real debate_id from a prior test/fixture>",
        "role_id": "<a real role_id from a prior test/fixture>",
        "numero_ronda": 1,
        "contenido": "realtime verification row",
    }).execute()

    await asyncio.wait_for(event_received.wait(), timeout=5)
    assert event_received.is_set()
```

**Open point flagged, not resolved by research (see Open Questions):** whether `supabase-py`'s Realtime `.channel()`/`.on_postgres_changes()` API requires the async client (`acreate_client`) as shown above, or whether the sync client also exposes it — official docs examples are consistently `await`-based, but this should be confirmed directly against the installed `supabase==2.31.x` package's actual API at implementation time rather than assumed from docs alone (docs don't always specify a minimum version for a documented method).

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|---------------|--------|
| `bona-debate-ai-spec.md`'s single `modelo` column (`claude`/`gpt`/`gemini`) | D-01's split `proveedor` + `modelo_id` columns | This phase (D-01, locked in CONTEXT.md) | Schema must not follow the original spec's `debate_roles.modelo` column — use the two-column shape. |
| `bona-debate-ai-spec.md`'s `orden` framed as "sequential intervention order" | D-04's `orden` reframed as PWA display order only | This phase (D-04, locked in CONTEXT.md) | No behavior depends on `orden` for execution sequencing — Phase 3's engine uses `Send()` fan-out (parallel), not `orden`-driven sequencing. |
| Supabase CLI declarative schema diffing (`supabase db diff`) as the primary schema-management approach | Hand-run flat `schema.sql`, applied via Studio SQL Editor | Already the established convention in this specific `bonapps` project (2 sibling apps), not a recent platform change | Affects tooling choice for this phase — see Alternatives Considered. Also, even the CLI's own docs note declarative diffing doesn't capture RLS policy statements or `ALTER PUBLICATION` statements, so CLI users still need a hand-written migration for D-06 regardless of tooling choice. [CITED: supabase.com/docs/guides/local-development/declarative-database-schemas] |

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `supabase-py 2.31.x`'s Realtime channel API requires the async client (`acreate_client`)/`AsyncClient`, not the sync `create_client` | Code Examples, Common Pitfalls | Low — if wrong, the verification script's realtime test needs a small rewrite (swap to sync client's channel API if it exists); doesn't affect the SQL/schema/RLS/publication design itself, only the test harness's exact code shape. |
| A2 | Supabase's hosted `bonapps` project has default privilege grants for `authenticated`/`anon` roles that *might* already cover the four new tables without the explicit `GRANT` statements in Pattern 2 | Architecture Patterns, Pitfall 1 | Low-medium — being explicit (as recommended) makes this moot regardless of whether the assumption is true; the risk is only in *not* following the recommendation and instead assuming defaults will "just work," which the two sibling apps' own inline comments suggest they didn't want to trust either. |
| A3 | No `SUPABASE_URL`/`SUPABASE_SERVICE_KEY`/CLI access token for the `bonapps` project exists anywhere in this local environment (checked via file search only, not by attempting to read `.env` contents) | Environment Availability | Medium — if credentials do exist somewhere not found by this search (e.g., a password manager, a not-yet-cloned secrets repo), the "blocking, no fallback" flag below is overly pessimistic and the planner can proceed without a human-supplied-credentials checkpoint. If credentials truly don't exist yet, the checkpoint is required as flagged. |

**If this table is empty:** N/A — see rows above.

## Open Questions

1. **Does `supabase-py 2.31.x`'s Realtime API require `acreate_client` (async), or does the sync client also support `.channel()`?**
   - What we know: Official docs (`supabase.com/docs/reference/python/subscribe`) show only `await`-based examples.
   - What's unclear: Whether this is a hard API requirement or just how docs choose to demonstrate it, and whether `2.31.x` specifically (vs. some other version) has any relevant difference.
   - Recommendation: Confirm directly by importing the installed package and inspecting `Client`/`AsyncClient` at implementation time (5-minute check) rather than guessing; write the verification script's realtime test with whichever client actually exposes `.channel()`.

2. **Should this phase use the hand-run `schema.sql` convention (recommended here) or introduce the Supabase CLI's `migrations/` workflow, diverging from sibling-app precedent?**
   - What we know: `research/ARCHITECTURE.md` suggested the CLI-style folder; the two actual sibling apps in the same `bonapps` project use flat `schema.sql` instead; the CLI is reachable via `npx` but has no linked credentials in this environment.
   - What's unclear: Whether the user has a project-level preference for versioned CLI migrations going forward (not stated in CONTEXT.md, PROJECT.md, or CLAUDE.md) that would justify the extra setup cost now.
   - Recommendation: Default to `schema.sql` (matches precedent, zero new credentials needed) unless the planner/user has a specific reason to introduce CLI-managed migrations for this project starting now.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Docker | (not needed this phase — Phase 4 only) | ✓ | 29.1.4 | — |
| Node.js / npm / npx | Optional Supabase CLI path | ✓ | node v22.23.2, npm 12.0.2 | — |
| Supabase CLI (local install) | Optional CLI migrations workflow | ✗ | — | `npx supabase@latest <cmd>` confirmed working (v2.115.0) |
| Python 3 | Verification script | ✓ | 3.14.6 (system) — note: project's Docker backend target is 3.12, irrelevant for this throwaway script | — |
| `pip` / `pip3` / `pipx` (on PATH) | Installing `supabase-py`, `pytest`, `pytest-asyncio` | ✗ | — | `python3 -m venv .venv && source .venv/bin/activate` bootstraps a working `pip` via `ensurepip` (confirmed both modules present) |
| `SUPABASE_URL` / `SUPABASE_SERVICE_KEY` / project-ref for `bonapps` | Applying the schema, running any verification test against the real project | ✗ (not found in this environment — search only, see Assumption A3) | — | None — this is a hard blocker for actually running anything against `bonapps`, not just a tooling inconvenience |

**Missing dependencies with no fallback:**
- Credentials for the real `bonapps` Supabase project (URL + service-role key, or dashboard access) — required to actually apply the schema and run any verification test against production data. This phase cannot be executed end-to-end without a human supplying these (Supabase dashboard login or the project's service-role key), since there is no local/self-hosted Supabase instance to fall back to for this shared-project phase. **The planner should insert a `checkpoint:human-verify` (or equivalent) task early in this phase's plan for credential/dashboard access**, before any task that assumes it can reach `bonapps`.

**Missing dependencies with fallback:**
- Supabase CLI (local binary) — `npx supabase@latest` is a fully viable substitute, already confirmed working.
- `pip` on PATH — a local venv resolves this in one command, already confirmed working.

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | `pytest` + `pytest-asyncio` (project's already-chosen tool, per CLAUDE.md Development Tools — not yet installed/configured anywhere in this empty repo) |
| Config file | none yet — Wave 0 gap |
| Quick run command | `pytest verify/test_phase1_rls_realtime.py -x` |
| Full suite command | Same as quick run — this phase has no other tests, and no prior-phase test suite exists to run alongside it (this is Phase 1 of 5, repo was empty before this phase) |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| AUTH-03 | Shared visibility: any authenticated account reads/writes all debate data, no per-account isolation | integration (real Supabase project) | `pytest verify/test_phase1_rls_realtime.py::test_two_independent_authenticated_users_share_visibility -x` | ❌ Wave 0 |
| (success criterion 1) | 4 tables exist with correct schema/FKs/indexes | integration (SQL introspection or manual dashboard check) | `pytest verify/test_phase1_rls_realtime.py::test_schema_shape -x` (schema-introspection test, not shown in Code Examples above — planner should add one querying `information_schema.columns`) | ❌ Wave 0 |
| (success criterion 2, anon half) | Anonymous requests rejected by RLS | integration | `pytest verify/test_phase1_rls_realtime.py::test_anon_select_returns_empty_not_error test_anon_insert_raises_rls_violation -x` | ❌ Wave 0 |
| (success criterion 3) | `debates`/`debate_rounds` inserts publish to `supabase_realtime`, received by a live subscription | integration (real WebSocket round-trip) | `pytest verify/test_phase1_rls_realtime.py::test_realtime_delivers_insert_event -x` | ❌ Wave 0 |

### Sampling Rate

- **Per task commit:** `pytest verify/test_phase1_rls_realtime.py -x` (full file — it's small, no reason to sample a subset)
- **Per wave merge:** same command
- **Phase gate:** Full suite green before `/gsd:verify-work`, plus a manual check of the Supabase dashboard's Database → Replication tab confirming `debates`/`debate_rounds` show as active publication members (belt-and-suspenders for Pitfall 4, since the dashboard is the ground truth Supabase itself uses)

### Wave 0 Gaps

- [ ] `verify/conftest.py` — env var loading (`SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_KEY`), shared client fixtures
- [ ] `verify/test_phase1_rls_realtime.py` — all four tests above, plus a schema-shape introspection test
- [ ] `pytest.ini` or `pyproject.toml` `[tool.pytest.ini_options]` — minimal config, `asyncio_mode = "auto"` for `pytest-asyncio`
- [ ] Python env bootstrap: `python3 -m venv .venv && source .venv/bin/activate && pip install supabase pytest pytest-asyncio` (gate behind `checkpoint:human-verify` per Package Legitimacy Audit)
- [ ] `checkpoint:human-verify` for `bonapps` project credentials (URL, service-role key, or dashboard access) — hard blocker, no fallback (see Environment Availability)

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-------------------|
| V2 Authentication | Indirect (not built this phase) | Supabase Auth (OTP email), already established ecosystem-wide; this phase only consumes the resulting `authenticated`/`anon` role distinction, doesn't implement login |
| V3 Session Management | No | Not touched this phase — no session code written |
| V4 Access Control | **Yes — primary** | Postgres RLS (D-05) + explicit `GRANT` (Pattern 2); default-deny for `anon`; this is the phase's core security deliverable |
| V5 Input Validation | Yes (light) | `CHECK` constraints (`proveedor in ('gemini','openrouter')`, `estado in (...)`) at the DDL level — no app-layer validation exists yet, so the DB constraints are the only validation gate this phase provides |
| V6 Cryptography | No | Not touched this phase — no crypto work, no secret generation/storage |

### Known Threat Patterns for this stack

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|----------------------|
| Public anon key (embedded in future PWA's client JS) used to read/write debate data without a session | Elevation of Privilege / Information Disclosure | D-05's default-deny for `anon` (zero policies) + explicit `GRANT` restricted to `authenticated` only — verified by the anon-rejection tests in this phase's verification script |
| RLS policy correct but Realtime publication forgotten, causing "live" features to silently break (not a data-exposure issue, but a correctness/reliability one) | (not classic STRIDE — availability/correctness gap) | D-06 + Pitfall 4's dashboard-verification step |
| A future authenticated account (any BONA ecosystem user sharing `bonapps`, not just intended `bonasterio` users) technically able to read `debate_*` tables since isolation is by table-name prefix, not schema | Information Disclosure (accepted risk, explicitly documented in `research/ARCHITECTURE.md`) | Not mitigated this phase — accepted trade-off per `research/ARCHITECTURE.md`'s RLS Design section, valid only because all `bonapps`-authenticated users are trusted BONA team members. Re-evaluate if the shared Supabase project ever gains untrusted/external authenticated users. |
| SQL injection via the verification script or future app code | Tampering | N/A this phase — `supabase-py`/PostgREST client libraries parameterize all values; no raw string-interpolated SQL is used anywhere in this phase's design |

## Sources

### Primary (HIGH confidence)

- [PostgreSQL Docs: Row Security Policies](https://www.postgresql.org/docs/current/ddl-rowsecurity.html) — SELECT-vs-INSERT RLS rejection behavior, cross-verified against Supabase/PostgREST community threads
- [PostgreSQL Docs: `pg_publication_tables`](https://www.postgresql.org/docs/current/view-pg-publication-tables.html) — system catalog used for the idempotent publication-guard pattern
- [Supabase Docs: Postgres Changes (Realtime)](https://supabase.com/docs/guides/realtime/postgres-changes) — `ALTER PUBLICATION ... ADD TABLE` syntax, RLS SELECT policy requirement for event delivery, `REPLICA IDENTITY FULL` only needed for filtering DELETE events (not required this phase — only INSERT/UPDATE on `debates`/`debate_rounds` are in scope)
- [Supabase Docs: Database Migrations](https://supabase.com/docs/guides/deployment/database-migrations) — CLI `migration new`/`db push` workflow (evaluated, not chosen — see Alternatives Considered)
- [Supabase Docs: Declarative Database Schemas](https://supabase.com/docs/guides/local-development/declarative-database-schemas) — confirms `db diff` does not capture RLS `alter policy` or `alter publication` statements
- [Supabase Docs: Python — Subscribe to channel](https://supabase.com/docs/reference/python/subscribe) — `on_postgres_changes` API shape (async-only examples, see Open Question 1)
- Codebase: `/home/bona/lista-super/supabase/schema.sql`, `/home/bona/bona-consumos/supabase/schema.sql` — established, production-running RLS + Realtime + explicit-GRANT pattern in the same `bonapps` Supabase project
- `npm view supabase version` (2.115.0) and `curl https://pypi.org/pypi/supabase/json` (2.31.0) — directly verified against registries, 2026-08-20

### Secondary (MEDIUM confidence)

- [GitHub supabase/discussions #36619: RLS check failing on INSERT despite `WITH CHECK (true)`](https://github.com/orgs/supabase/discussions/36619) — corroborates SELECT-vs-INSERT RLS behavior difference
- WebSearch (multiple queries, cross-referenced against official docs above) — Supabase CLI current version confirmation, RLS testing patterns (`SET ROLE authenticated`/`request.jwt.claims` SQL-Editor technique, mentioned as an alternative to the `sign_in_with_password` approach recommended here)

### Tertiary (LOW confidence)

- None used without cross-verification in this research.

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — versions directly confirmed against PyPI/npm registries; tooling choice backed by production precedent in this exact Supabase project
- Architecture: HIGH — RLS/Realtime/publication mechanics confirmed via official Supabase/PostgreSQL docs; schema-file convention confirmed via direct inspection of two sibling production repos
- Pitfalls: HIGH for the two new findings (GRANT requirement, SELECT-vs-INSERT rejection shape) — both cross-verified against official docs and/or production codebase evidence, not training-data-only claims

**Research date:** 2026-08-20
**Valid until:** 30 days (stable domain — Postgres RLS/Realtime mechanics don't change quickly; re-verify Supabase CLI version and `supabase-py` version if this research is reused after that window)
