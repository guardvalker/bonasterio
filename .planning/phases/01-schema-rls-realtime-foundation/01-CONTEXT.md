# Phase 1: Schema, RLS & Realtime Foundation - Context

**Gathered:** 2026-08-20
**Status:** Ready for planning

<domain>
## Phase Boundary

Supabase (shared `bonapps` project) gets a secure, realtime-ready data foundation for `bonasterio`: four tables (`debate_roles`, `debates`, `debate_rounds`, `debate_config`), the shared-team RLS model (no `owner_id` isolation) implemented and verified, and `debates`/`debate_rounds` publishing to `supabase_realtime`. No backend, no PWA — schema, RLS, and Realtime plumbing only, validated with test queries/subscriptions, not a real logged-in UI session (that's Phase 2).

</domain>

<decisions>
## Implementation Decisions

### Model Identifier Format
- **D-01:** `debate_roles` gets two columns — `proveedor` (text: `gemini` | `openrouter`) and `modelo_id` (free-text model identifier, e.g. `gemini-2.5-flash`, `meta-llama/llama-3.1-8b-instruct:free`) — instead of the original spec's single `modelo` field (which assumed `claude`/`gpt`/`gemini` values, superseded by the Gemini+OpenRouter decision in PROJECT.md). Splitting into two columns makes backend provider routing explicit and gives the Phase 2 role-CRUD UI a clean provider dropdown + free model-id field, rather than parsing a convention out of one string.

### Creator Attribution
- **D-02:** `debates` gets a nullable `created_by uuid references auth.users` column, **display-only** — "who started this debate" — never used in an RLS policy. Cheap to add now; painful to backfill later if a teammate starts using the tool (per PROJECT.md's "por si en el futuro otra persona del equipo la usa").

### Budget Number Precision
- **D-03:** All budget/money columns (`debates.presupuesto_final_min/max`, `debates.presupuesto_real_facturado`, `debate_rounds.presupuesto_propuesto_min/max`) use `numeric` with no fixed precision/scale — not `integer`. These are estimate ranges, not invoices, but unconstrained `numeric` costs nothing and avoids ever needing a migration if a decimal shows up.

### Role Display Order
- **D-04:** `debate_roles.orden` is kept, but reencuadrado as **PWA display order** (which role's card appears first in the UI), not "sequential intervention order" as the original spec framed it — that framing assumed sequential rounds, which is superseded by the parallel `Send()` fan-out execution model in `research/ARCHITECTURE.md`. Downstream (Phase 2 UI, Phase 3 engine) should not read `orden` as execution order.

### RLS Model (carried forward from roadmap — re-confirmed, not re-opened)
- **D-05:** Shared-team RLS on all 4 tables: `for all to authenticated using (true) with check (true)`. `anon` role gets **zero** policies (default-deny) — the anon key is only ever used for the pre-login OTP flow. No `owner_id` column for isolation (D-02's `created_by` is explicitly display-only, never a policy filter). This was resolved during roadmap creation per `research/ARCHITECTURE.md`'s Anti-Pattern 3 and PROJECT.md's stated "shared team tool" intent (AUTH-03) — **not a live gray area**, included here so downstream agents don't second-guess it.
  - **Known conflicting advice to ignore:** `research/PITFALLS.md` Pitfall 7 gives generic advice to add `owner_id`-scoped policies "even for solo-user apps." That advice is superseded by this project's specific, deliberate shared-data decision. Do not apply Pitfall 7's `owner_id` recommendation.

### Realtime Publication (carried forward from roadmap)
- **D-06:** `debates` and `debate_rounds` must be added to the `supabase_realtime` publication (`ALTER PUBLICATION supabase_realtime ADD TABLE ...`) in the **same migration** as their RLS policies (`research/PITFALLS.md` Pitfall 8 — silent-failure risk if done separately or forgotten). `debate_roles` and `debate_config` are not required in the publication for this phase's success criteria — no live-editing use case for those tables yet.

### Claude's Discretion
- Exact index set beyond `research/ARCHITECTURE.md`'s suggestion (`debate_rounds(debate_id)`, `debates(estado)`, `debates(created_at desc)`) — add more only if a concrete need shows up during planning.
- `debate_config` table shape: support multiple named presets (spec shows `"default"`/`"rapido"`/`"profundo"` rows with an `activo` flag) vs. just one row — build the columns per the spec (`nombre`, `cantidad_rondas`, `roles_incluidos`, `activo`), but only one active row is needed for v1; no preset-switching UI is in scope this phase or Phase 2.
- `estado` column on `debates`: plain `text` with a `CHECK` constraint (`en_curso`/`completo`/`error`) vs. a Postgres enum type — implementation detail, no user-facing impact.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Schema baseline (superseded in parts — see Implementation Decisions above)
- `bona-debate-ai-spec.md` §"Componente 1: Base de datos" — original 4-table column list. `modelo` (debate_roles), sequential-round framing of `orden`, and the model roster (Claude/GPT/Gemini) are all superseded by D-01, D-04, and PROJECT.md's Gemini+OpenRouter decision, respectively. Everything else in this section (table set, `debate_rounds`/`debate_config` shape, suggested indices) still applies.

### RLS & Realtime design
- `.planning/research/ARCHITECTURE.md` §"RLS Design (single-user-with-login, shared Postgres project)" — the exact policy SQL shape to implement (D-05).
- `.planning/research/ARCHITECTURE.md` §"Anti-Pattern 3: Per-user owner_id RLS scoping for what is actually shared team data" — why no `owner_id`.
- `.planning/research/PITFALLS.md` Pitfall 7 (RLS under-scoping) and Pitfall 8 (Realtime publication forgotten) — read both; Pitfall 7's `owner_id` recommendation is explicitly overridden by D-05, but its RLS-testing checklist (verify anon rejection, verify a different authenticated user) still applies. Pitfall 8's guidance (publication in the same migration as RLS) is D-06.

### Project-level constraints
- `.planning/PROJECT.md` §"Constraints" and §"Key Decisions" — cost/infra/DB/auth constraints; the shared-`bonapps`-project and no-owner-id decisions.
- `.planning/REQUIREMENTS.md` — AUTH-03 ("todas las cuentas comparten el mismo historial... sin aislamiento por cuenta") is the requirement this phase's RLS design satisfies.
- `.planning/ROADMAP.md` §"Phase 1" — the three success criteria this phase must satisfy verbatim.

### Stack versions (for the migration tooling, not app code)
- `.planning/research/STACK.md` — `supabase-py`/`@supabase/supabase-js` versions if a test script/harness is written to verify RLS or Realtime during this phase.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
None — `bonasterio` repo currently has no code, only planning docs (`.planning/`, `CLAUDE.md`, `bona-debate-ai-spec.md`). This phase is the first thing built.

### Established Patterns
- Other BONA ecosystem apps (lista-super, bonapp-gastos, Toolbox) share the same `bonapps` Supabase project and use OTP email login (not magic link — magic link is known broken in installed iOS PWAs). Not directly relevant to schema/RLS work in this phase, but the shared-project constraint (table naming via `debate_` prefix, not a separate schema) is why isolation is by name, not by Postgres schema.

### Integration Points
- Migrations will live under `supabase/migrations/` per `research/ARCHITECTURE.md`'s suggested project structure — this phase produces the first migration(s) in an otherwise-empty repo.

</code_context>

<specifics>
## Specific Ideas

No additional specific references beyond the four schema decisions (D-01 through D-04) above — those are the concrete "how" choices for this phase.

</specifics>

<deferred>
## Deferred Ideas

None — discussion stayed within phase scope. (User deferred all four implementation questions to Claude's judgment rather than raising new scope.)

</deferred>

---

*Phase: 1-Schema, RLS & Realtime Foundation*
*Context gathered: 2026-08-20*
