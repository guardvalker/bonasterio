---
phase: 1
slug: schema-rls-realtime-foundation
status: draft
nyquist_compliant: false
wave_0_complete: false
created: 2026-08-20
---

# Phase 1 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | pytest + pytest-asyncio (project's chosen tool per CLAUDE.md — not yet installed/configured in this empty repo) |
| **Config file** | none yet — Wave 0 gap |
| **Quick run command** | `pytest verify/test_phase1_rls_realtime.py -x` |
| **Full suite command** | `pytest verify/test_phase1_rls_realtime.py -x` (same — no other tests exist yet, this is Phase 1 of a fresh repo) |
| **Estimated runtime** | ~15 seconds (integration tests against real Supabase project) |

---

## Sampling Rate

- **After every task commit:** Run `pytest verify/test_phase1_rls_realtime.py -x`
- **After every plan wave:** Run `pytest verify/test_phase1_rls_realtime.py -x`
- **Before `/gsd:verify-work`:** Full suite must be green, plus a manual check of the Supabase dashboard's Database → Replication tab confirming `debates`/`debate_rounds` show as active publication members (belt-and-suspenders — the dashboard is Supabase's own ground truth)
- **Max feedback latency:** 15 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 01-01-XX | 01 | 0 | — | — | Test infra bootstrap (venv, deps, conftest) | infra | `pytest --collect-only verify/` | ❌ W0 | ⬜ pending |
| 01-XX-XX | TBD | TBD | success criterion 1 | — | 4 tables exist with correct schema/FKs/indexes | integration | `pytest verify/test_phase1_rls_realtime.py::test_schema_shape -x` | ❌ W0 | ⬜ pending |
| 01-XX-XX | TBD | TBD | AUTH-03 | T-1-01 | Any authenticated account reads/writes all debate data, no per-account isolation | integration | `pytest verify/test_phase1_rls_realtime.py::test_two_independent_authenticated_users_share_visibility -x` | ❌ W0 | ⬜ pending |
| 01-XX-XX | TBD | TBD | success criterion 2 | T-1-01 | Anonymous SELECT returns empty (not error); anonymous INSERT raises RLS violation (42501) | integration | `pytest verify/test_phase1_rls_realtime.py::test_anon_select_returns_empty_not_error verify/test_phase1_rls_realtime.py::test_anon_insert_raises_rls_violation -x` | ❌ W0 | ⬜ pending |
| 01-XX-XX | TBD | TBD | success criterion 3 | — | `debates`/`debate_rounds` inserts publish to `supabase_realtime`, received by a live subscription | integration (real WebSocket round-trip) | `pytest verify/test_phase1_rls_realtime.py::test_realtime_delivers_insert_event -x` | ❌ W0 | ⬜ pending |

*Task IDs are TBD — planner assigns final IDs when PLAN.md is written; this map records the requirement→test binding the plan must satisfy.*

---

## Wave 0 Requirements

- [ ] `verify/conftest.py` — env var loading (`SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_KEY`), shared client fixtures
- [ ] `verify/test_phase1_rls_realtime.py` — all four tests above, plus the schema-shape introspection test
- [ ] `pytest.ini` or `pyproject.toml` `[tool.pytest.ini_options]` — minimal config, `asyncio_mode = "auto"` for `pytest-asyncio`
- [ ] Python env bootstrap: `python3 -m venv .venv && source .venv/bin/activate && pip install supabase pytest pytest-asyncio`
- [ ] `checkpoint:human-verify` for `bonapps` project credentials (URL, service-role key, or dashboard access) — hard blocker, no fallback exists in this environment; the human must supply these before Wave 0's integration tests can run at all

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| `debates`/`debate_rounds` show as active members of the `supabase_realtime` publication | success criterion 3 | Belt-and-suspenders cross-check against Supabase's own dashboard ground truth, catches silent publication-config drift that a client-side subscription test could miss | Supabase Studio → Database → Replication tab → confirm both tables listed under `supabase_realtime` |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 15s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
