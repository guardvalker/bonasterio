# Phase 1: Schema, RLS & Realtime Foundation - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-08-20
**Phase:** 1-Schema, RLS & Realtime Foundation
**Areas discussed:** Model identifier format, Creator attribution, Budget number precision, Role display order

---

## Model Identifier Format

| Option | Description | Selected |
|--------|-------------|----------|
| Single `modelo` text field with a convention | e.g. `"openrouter:llama-3.1-8b"` — matches original spec's single-column shape | |
| Split `proveedor` + `modelo_id` columns | `proveedor` (`gemini`\|`openrouter`) + free-text `modelo_id` — explicit provider routing, cleaner Phase 2 UI | ✓ |

**User's choice:** Deferred to Claude ("elegi vos como empezar" / "segui nomas") — Claude selected split columns.
**Notes:** Original spec assumed `claude`/`gpt`/`gemini` as the `modelo` values; that roster is already superseded by PROJECT.md's Gemini+OpenRouter decision, so the column design needed to change regardless.

---

## Creator Attribution

| Option | Description | Selected |
|--------|-------------|----------|
| Add `created_by uuid references auth.users` (nullable, display-only) | Track who started each debate, for future multi-person use | ✓ |
| Skip it for v1 | Single user right now, add later if a teammate joins | |

**User's choice:** Deferred to Claude — Claude selected adding the column.
**Notes:** Explicitly display-only, never used in an RLS policy (RLS model stays `authenticated USING (true)` per the already-locked roadmap decision).

---

## Budget Number Precision

| Option | Description | Selected |
|--------|-------------|----------|
| `integer` (whole pesos) | Simpler, matches how quotes are usually given verbally | |
| `numeric` (no fixed scale) | Allows decimals without forcing them, no future migration risk | ✓ |

**User's choice:** Deferred to Claude — Claude selected unconstrained `numeric`.
**Notes:** Applies to all four money columns across `debates` and `debate_rounds`.

---

## Role Display Order

| Option | Description | Selected |
|--------|-------------|----------|
| Drop `orden` column | No longer functional since roles run in parallel via LangGraph `Send()`, not sequentially | |
| Keep `orden`, reencuadrado as display order | Repurpose as "which role card shows first in the PWA," not execution order | ✓ |

**User's choice:** Deferred to Claude — Claude selected keeping the column with the reframed purpose.
**Notes:** Original spec's "orden de intervención dentro de una ronda (para debates no simultáneos)" framing is stale — superseded by the parallel-execution architecture decision in `research/ARCHITECTURE.md`.

---

## Claude's Discretion

- Exact index set beyond `research/ARCHITECTURE.md`'s baseline suggestion (`debate_rounds(debate_id)`, `debates(estado)`, `debates(created_at desc)`).
- `debate_config` single-row vs. multi-preset table shape (columns support presets per spec; only one active row needed for v1).
- `estado` column implementation: `text` + `CHECK` constraint vs. Postgres enum type.

## Deferred Ideas

None — discussion stayed within phase scope.
