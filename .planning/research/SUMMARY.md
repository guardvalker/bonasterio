# Project Research Summary

**Project:** bonasterio
**Domain:** Multi-AI "debate council" internal tool (LangGraph-orchestrated LLM ensemble → budget estimation for electrical contracting), FastAPI on Cloud Run + Supabase-realtime PWA
**Researched:** 2026-08-20
**Confidence:** HIGH

## Executive Summary

bonasterio is a solo-user (eventually small-team) internal decision-support tool, not a commercial product — it runs a structured multi-agent LLM debate (parallel initial proposals → one adjustment round → arbiter verdict) to produce a budget range for electrical jobs, then tracks that verdict against what Bona actually invoices over time. Research across all four domains (stack, features, architecture, pitfalls) converges cleanly and confirms nearly every decision already made in PROJECT.md is correct: fixed 2-4 heterogeneous models, one adjustment round, no WebSocket, no configurable round count, and free-tier-only APIs (Gemini 2.5 Flash/Flash-Lite + OpenRouter `:free` models) are all independently validated by current multi-agent-debate literature and official pricing docs, not just cost-driven guesses.

The recommended approach is: FastAPI + LangGraph on Cloud Run (single service, request-based billing, single Uvicorn process), with the debate graph running **synchronously inside the request** (not a background task) because Cloud Run throttles CPU the instant an HTTP response is sent — this is the single most dangerous mistake this project could make, since the spec's original "return `debate_id` immediately, run in background" plan does not work reliably on Cloud Run's default billing. The fix requires no extra infra: the PWA generates the `debate_id` client-side, opens its Supabase Realtime subscription before firing the POST, and the backend holds the request open for the full 15-30s run while writing each round to Postgres incrementally (which is also what makes Realtime feel live). Architecturally, most of the app (role CRUD, history, config, retroactive invoice annotation) needs **no backend at all** — it's direct PWA-to-Supabase CRUD gated by RLS; the backend's only required responsibility is the one operation needing secrets (`POST /debates`).

Key risks, in priority order: (1) the Cloud Run background-execution trap described above — architectural, must be resolved before any deployment; (2) free-tier LLM rate limits (OpenRouter ~20/min & 50-1000/day, Gemini per-model caps) getting exhausted during iterative development, not production use — needs retry/backoff and quota-awareness from the first real API call; (3) LangGraph's default last-write-wins state merge silently dropping parallel role results unless reducers are explicitly annotated; (4) RLS/security — there is a genuine unresolved tension between ARCHITECTURE.md's "shared team data, `authenticated USING (true))`" recommendation and PITFALLS.md's "always add `owner_id` scoping even for solo users" recommendation, which the roadmap/planning phase must explicitly resolve (see Gaps below) since the anon key is public in the PWA bundle either way and getting this wrong is a real data-exposure risk, not a hypothetical one.

## Key Findings

### Recommended Stack

Python 3.12 + FastAPI 0.141.x + Uvicorn (single process, no Gunicorn) + LangGraph 1.2.x (native `Send()` API for dynamic parallel fan-out) form the backend core, containerized via multi-stage Docker and deployed as a Cloud Run **Service** (not Job) with default request-based billing and `min-instances=0`. `langchain-google-genai` wraps Gemini roles, `langchain-openai` (pointed at OpenRouter's OpenAI-compatible endpoint) wraps OpenRouter roles — both behind one `ainvoke()` interface. Supabase (shared `bonapps` project) provides Postgres + Auth (OTP) + Realtime; `supabase-py` (service-role key) on the backend, `@supabase/supabase-js@2` via CDN (no build step, matching the existing BONA PWA pattern) on the frontend.

**Core technologies:**
- FastAPI + Uvicorn (async-native) — needed because each request fires several concurrent outbound LLM calls
- LangGraph `Send()` API — the correct primitive for "fan out to N runtime-determined roles in parallel, fan back in," which matches this project's DB-editable role count exactly
- Cloud Run Service, 2nd-gen, request-based billing, scale-to-zero — fits "zero cost when idle," but requires the debate to run **inside** the request, not after it
- Supabase Postgres + Auth + Realtime (shared `bonapps` project) — `postgres_changes` on INSERT is exactly the "live rounds" requirement, no polling/WebSocket server needed
- Gemini 2.5 Flash/Flash-Lite + OpenRouter `:free` models — confirmed still free as of research date; explicitly avoid `gemini-2.0-flash` (deprecated/shutdown) and `gemini-2.5-pro` (not on free tier)

### Expected Features

Research (multi-agent-debate literature + LLM-as-judge industry sources) validates essentially the entire Active requirements list in PROJECT.md as correctly scoped — this is a case where research confirms rather than expands the plan.

**Must have (table stakes):**
- Independent parallel first-round proposals (no cross-contamination before round 1)
- Visible disagreement as a first-class field, not buried prose — the arbiter's "dónde hubo desacuerdo" is the core value, not a nice-to-have
- Structured job input (task/materials/zone/complexity)
- Live/streaming progress during the 15-30s run — a blank screen reads as "broken" for anything >5s in 2026
- Full expandable round-by-round history + persisted debate list
- Role/persona CRUD without redeploy (already committed — validated as necessary because AAIERIC's official labor-cost table is periodically revised for inflation)
- OTP login before storing real business pricing data
- Fixed small round count (2-4 agents, 1-2 rounds) — literature shows accuracy plateaus here; more rounds add cost/latency without improving output

**Should have (differentiators):**
- Retroactive real-invoice annotation + accuracy tracking — the feature that turns this from "AI chat with extra steps" into a self-improving system; no commercial estimating tool tracks per-debate, per-role prediction-vs-actual at this granularity
- Heterogeneous model ensemble (different labs, not the same model resampled) — research is explicit that heterogeneity, not just multiplicity, is what makes disagreement meaningful
- Per-role calibration bias tracking ("this role overestimates by 15%") — the eventual payoff of accuracy tracking, but genuinely needs data volume first

**Defer (v2+):**
- Per-role/per-model accuracy stats and bias dashboards — needs ~10-20 annotated real-invoice debates before it's statistically meaningful, not viable at launch
- AAIERIC automatic category mapping — no concrete need yet, defer until accuracy analytics need to be sliced by category
- Configurable round count per individual run, adaptive "debate until convergence" — both explicitly conflict with the free-tier rate-limit safety margin; correctly out of scope
- Anything customer-facing (PDF quotes, e-signature, payment, CRM) — wrong product category entirely; this is internal decision support, not a quoting SaaS

### Architecture Approach

The system splits cleanly into "backend does the one thing that needs secrets" and "PWA talks directly to Supabase for everything else." The PWA is a static, no-build-step site (same pattern as other BONA apps) using the Supabase anon key + RLS for auth, role CRUD, debate history, and retroactive invoice annotation. The backend (FastAPI + LangGraph, single Cloud Run service) exposes essentially one endpoint, `POST /debates`, using the Supabase service-role key to write `debates`/`debate_rounds` rows incrementally as each graph node completes — this is what makes Realtime-driven "live" progress possible without polling or a checkpointer. The critical resolved design nuance: the PWA must generate the `debate_id` client-side (`crypto.randomUUID()`) and open its Realtime subscription *before* firing the POST, because the POST itself blocks for the full run duration (required by Cloud Run's CPU-allocation model) — this avoids needing a task queue while still giving live updates.

**Major components:**
1. **PWA (static, GitHub Pages)** — auth UI, forms, Realtime subscription/render, direct Supabase CRUD for everything except kicking off a debate
2. **Supabase Postgres + Auth + Realtime (shared `bonapps` project)** — system of record for 4 tables (`debate_roles`, `debates`, `debate_rounds`, `debate_config`), RLS gate, live change feed
3. **FastAPI + LangGraph backend (Cloud Run)** — orchestrates the debate graph (`Send()` fan-out for parallel rounds, reducer-accumulated state, arbiter as final plain-edge node), calls external LLM APIs, writes incrementally via service-role key, verifies caller JWT

**Suggested build order** (per ARCHITECTURE.md, reordered from the original spec since most components have no backend dependency): (1) schema + RLS + Realtime publication config, (2) PWA screens that talk directly to Supabase (auth, role CRUD, history, config) — validates schema/RLS with zero backend, (3) LangGraph graph logic standalone/scripted against real roles and real free-tier calls, (4) FastAPI wrapper + JWT middleware + Docker + Cloud Run deploy, (5) PWA "nuevo debate" live view wiring client-generated-UUID + Realtime-before-POST — last because it's the only piece touching every other component.

### Critical Pitfalls

1. **Background execution collides with Cloud Run's CPU throttling** — the spec's original "return `debate_id` immediately, finish in background" design silently fails (debates stuck in `en_curso` forever) because Cloud Run only allocates CPU during active request handling by default. Avoid by running the graph synchronously inside the held-open request (15-30s is well within the 300s default timeout), with client-generated UUID + subscribe-before-POST to still get live updates.
2. **Free-tier LLM pool exhaustion during parallel fan-out** — 8-12+ calls per debate against OpenRouter's ~20/min & 50-1000/day caps and Gemini's per-model limits; iterative dev testing burns the *daily* cap fast. Build retry/backoff and quota tracking in from the first real API call, consider a mock-model dev mode.
3. **LangGraph parallel fan-out with unmanaged state reducers loses data** — default last-write-wins state merge silently drops all but one role's result unless `rondas` is annotated with an explicit reducer (`Annotated[list, operator.add]`). Must be correct before the first real parallel round runs; cheap to unit-test with 3+ mocked roles.
4. **Free-model lineup rotates without notice** — OpenRouter's `:free` catalog changes; a saved `debate_roles.modelo` can silently start erroring or (worse) silently start billing if the `:free` suffix is dropped. Validate model IDs against the live catalog at role-save time, log the exact model ID per round.
5. **RLS skipped or under-scoped for "single user"** — the PWA ships the public anon key in client JS; if RLS is off or too broad, anyone with that key can read/write all debate data. Must ship RLS + realtime-publication config in the same migration as the tables — but see the unresolved `owner_id` question in Gaps below.

## Implications for Roadmap

Based on research, suggested phase structure:

### Phase 1: Schema, RLS, and Realtime plumbing
**Rationale:** Zero dependencies, fast, and validates the RLS/Realtime design (Pitfalls 7 & 8) before anything is built on top of a possibly-wrong security model.
**Delivers:** 4 tables (`debate_roles`, `debates`, `debate_rounds`, `debate_config`) with indexes, RLS policies, and explicit `supabase_realtime` publication membership.
**Addresses:** Foundation for structured job input, role CRUD, history, retroactive annotation (all FEATURES.md table stakes).
**Avoids:** Pitfall 7 (RLS skipped/under-scoped) and Pitfall 8 (Realtime publication forgotten) — both must ship in this same migration, not later.

### Phase 2: PWA — auth, config, role CRUD, history (Supabase-direct, no backend)
**Rationale:** Nothing here depends on the backend at all; building it early validates the Phase 1 schema/RLS against a real OTP-logged-in session and gives Bona a usable tool to hand-manage `debate_roles`/`debate_config` while the graph is still being built.
**Delivers:** OTP login, role CRUD screen, debate history list, retroactive real-invoice annotation UI, config read.
**Uses:** `@supabase/supabase-js@2` via CDN, same no-build-step pattern as other BONA PWAs.
**Implements:** PWA↔Supabase direct-CRUD architectural boundary from ARCHITECTURE.md.

### Phase 3: LangGraph debate engine (standalone, script-driven, no HTTP yet)
**Rationale:** Isolates the hardest, most novel logic (dynamic `Send()` fan-out, state reducers, round loop, incremental persistence) from transport concerns; can be validated against real `debate_roles` rows (created in Phase 2) and real free-tier API calls before wrapping in FastAPI.
**Delivers:** Working `StateGraph` — parallel initial round, one adjustment round, arbiter node, `nodo_persistencia` writing to Supabase incrementally as each node completes.
**Addresses:** Core debate mechanic (parallel proposals, disagreement surfacing, arbiter verdict) — the entire P1 feature set from FEATURES.md.
**Avoids:** Pitfall 5 (unmanaged state reducers — unit-test fan-out with 3+ mocked roles before real calls), Pitfall 6 (unbounded round loop — hard iteration ceiling regardless of config), Pitfall 2 (quota exhaustion — retry/backoff and quota tracking from the first real call), Pitfall 4 (inconsistent JSON support — layered fallback: structured output → delimited text pattern → regex, never JSON-only).

### Phase 4: FastAPI wrapper, JWT auth, Docker, Cloud Run deploy
**Rationale:** Deploy early with just `POST /debates` to confirm cold-start behavior, default timeout headroom, and CORS against the real GitHub Pages origin before the frontend depends on it — deployment-environment surprises are cheaper to catch in isolation.
**Delivers:** Single deployed Cloud Run service exposing `POST /debates`, verifying incoming Supabase JWTs before running any LLM calls, secrets via Secret Manager (never baked into the image).
**Avoids:** Pitfall 1 (background execution vs. CPU throttling — request must stay open for the full graph run; verify by deploying, letting the instance go cold, and confirming a post-cold-start debate still completes) and the security mistakes table (service-role key, API keys never reachable from the PWA).

### Phase 5: Live debate flow (client-generated UUID + Realtime-before-POST)
**Rationale:** This is the piece with the most moving parts — it's the only component touching every other one (auth, Realtime, deployed backend, graph). Sequencing it last means each dependency has already been validated in isolation, so debugging is isolated to the integration itself.
**Delivers:** "Nuevo debate" screen — client generates `debate_id`, opens Realtime subscription, fires POST, renders rounds live as they arrive, shows final arbiter verdict on `debates` UPDATE regardless of whether the POST has resolved yet.
**Uses:** Pattern 4 from ARCHITECTURE.md (blocking POST as trigger, Realtime as the actual progress channel).
**Implements:** The full request/data flow diagrammed in ARCHITECTURE.md's "Request Flow" section.

### Phase Ordering Rationale

- Schema/RLS first because every other phase reads or writes these tables, and RLS mistakes are a real security exposure (public anon key), not a hypothetical one — cheapest to get right before data exists.
- Supabase-direct PWA screens before the backend because FEATURES.md and ARCHITECTURE.md agree most of the app has zero backend dependency — this front-loads low-risk, high-value screens and gives Bona a usable tool early even before the debate engine works.
- Graph logic before HTTP wrapper because it isolates LangGraph-specific pitfalls (reducers, round loop, fan-out) from Cloud Run-specific pitfalls (CPU throttling, cold starts) — debugging either in isolation is far easier than debugging both at once inside a live HTTP request.
- Deploy the backend skeleton before wiring the live frontend flow so infra surprises (CORS, cold start, timeout) are caught against a minimal endpoint, not against the most complex integration in the app.
- Live debate flow last because FEATURES.md's dependency graph confirms "live progress view requires incremental persistence" (Phase 3) and the architecture's resolved client-UUID pattern requires both a working backend (Phase 4) and a working Realtime-consuming PWA (Phase 2) to already exist.
- Per-role accuracy/bias tracking and AAIERIC auto-mapping are correctly excluded from all five phases above — FEATURES.md's dependency graph shows they need 10-20+ annotated real debates first, which cannot exist until the phases above ship and get used for a while.

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 3 (LangGraph engine):** `Send()` API and reducer semantics are MEDIUM confidence (WebSearch/Context7-verified, not hands-on tested in this repo) and LangGraph's API surface moves fast — verify current syntax against docs at implementation time, not against training data.
- **Phase 4 (Cloud Run deploy):** JWT verification method against Supabase's current JWT secret/JWKS approach should be re-confirmed at implementation time (Supabase Auth's key-rotation mechanics have changed across versions).
- **Phase 3 (price extraction sub-piece):** structured-output support is inconsistent per OpenRouter provider-endpoint, not per model — the layered fallback (structured → delimited text → regex) needs its own small research/design pass since no single test will cover all role/model combinations.

Phases with standard patterns (skip research-phase):
- **Phase 1 (schema/RLS):** Supabase RLS + Realtime publication patterns are HIGH confidence, official-docs-verified, and already validated in production by the user's own prior BONA apps.
- **Phase 2 (PWA CRUD screens):** Directly reuses the existing BONA ecosystem pattern (no-build-step CDN client, OTP auth) — already proven across Toolbox/bonapp-gastos/lista-super.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | HIGH | Cloud Run billing model, LangGraph `Send` API, Supabase JS/Py clients, OpenRouter rate limits all verified against official docs/Context7; exact free-model roster and numeric quotas are MEDIUM (rotate frequently, verify at implementation time) |
| Features | MEDIUM-HIGH | Multi-agent-debate mechanics backed by multiple 2025-2026 arXiv sources (HIGH); commercial estimating-tool comparisons are vendor marketing content (MEDIUM), used only for anti-feature contrast, not as a primary recommendation source |
| Architecture | HIGH (core patterns) / MEDIUM (exact LangGraph `Send` semantics) | Cloud Run timeout/CPU behavior and Supabase RLS/Realtime are official-docs-verified; LangGraph graph-shape specifics are WebSearch-verified across multiple sources but not hands-on tested in this repo |
| Pitfalls | MEDIUM-HIGH | Cloud Run/Supabase claims verified against official docs and pricing pages; OpenRouter/Gemini free-tier numeric limits are explicitly flagged as volatile snapshots, not guarantees |

**Overall confidence:** HIGH

### Gaps to Address

- **RLS ownership model conflict (must resolve before Phase 1):** ARCHITECTURE.md recommends `authenticated USING (true)` with **no** `owner_id` column, reasoning that this is shared team data (any BONA ecosystem user should see the same debate history). PITFALLS.md's Pitfall 7 recommends adding `owner_id`-scoped policies "regardless of expected user count" as defense against the public anon key. These are genuinely different designs, not a wording difference — the roadmap/planning phase must pick one explicitly. Given PROJECT.md's own stated intent ("por si en el futuro otra persona del equipo la usa," implying *shared* visibility, not per-account silos), the ARCHITECTURE.md shared-team-data model with default-deny for `anon` and `authenticated USING (true)` appears more aligned with actual intent — but this should be a conscious decision documented in the roadmap or Phase 1 plan, not left ambiguous, since getting it wrong is a real security exposure either direction (over-scoping breaks the "team shares history" goal; under-scoping exposes data to anyone with the public key if `authenticated` policies are ever misconfigured).
- **Exact free-model roster:** Both STACK.md and PITFALLS.md explicitly flag that OpenRouter's `:free` catalog and Gemini's exact quota numbers rotate frequently and should be re-verified at `openrouter.ai/models?max_price=0` and Google's pricing page at implementation time, not trusted from this research snapshot.
- **Debate duration estimate (15-30s):** Both STACK.md and ARCHITECTURE.md treat this as an assumption to validate, not a confirmed number — Phase 3's standalone graph-testing step is explicitly recommended as the place to confirm real free-tier model latency before committing to a synchronous-request architecture (if it balloons past a few minutes, Cloud Run's `--timeout` can be raised toward 3600s without re-architecting, so this is a low-risk gap, but should be measured early).
- **Structured-output fallback chain:** No single research source tests this end-to-end across the actual role roster this project will use; needs a small implementation-time spike (Phase 3) against real model responses, particularly prose-only free models that ignore JSON mode.

## Sources

### Primary (HIGH confidence)
- Context7 `/langchain-ai/langgraph` — `Send` API, map-reduce fan-out, state reducer accumulation
- Context7 `/supabase/supabase-py`, `/supabase/realtime` — OTP client methods, `postgres_changes` shape
- Cloud Run: CPU allocation, request timeout, billing settings, pricing — official docs (cloud.google.com/run/docs)
- Google Gemini API pricing (ai.google.dev/gemini-api/docs/pricing) — official, verified 2026-08-20
- OpenRouter API rate limits, FAQ, Structured Outputs (openrouter.ai/docs) — official
- Supabase Docs: Postgres Changes, Securing your data, service role key troubleshooting (supabase.com/docs) — official
- PyPI JSON API — current version numbers for entire recommended stack, verified 2026-08-20
- Self-Consistency Is Losing Its Edge (arXiv 2511.00751), Rethinking Mixture-of-Agents (arXiv 2502.00674), Understanding Agent Scaling via Diversity (arXiv 2602.03794) — peer-reviewed/preprint-track, inform round-count and heterogeneity findings

### Secondary (MEDIUM confidence)
- LangGraph `Send` API reference, Map-Reduce with Send() (Medium) — corroborating but not hands-on tested here
- Multi-Agent Debate Strategies Survey (arXiv 2607.26212), ARMOR-MAD (arXiv 2606.13197), CortexDebate (arXiv 2507.03928) — survey/preprint-track
- OpenRouter Free Tier 2026 (klymentiev.com), Gemini API Free Tier Guide 2026 (aifreeapi.com) — third-party, numbers should be re-verified at implementation time
- Supabase RLS Best Practices (makerkit.dev), Supabase RLS Guide 2026 (agilesoftlabs.com) — widely cited third-party, cross-checked against official docs
- Vendor marketing content (BuildOps, PataBid, MyQuoteIQ) — used only for anti-feature contrast, not as design guidance

### Tertiary (LOW confidence)
- None flagged as low confidence — all sources cross-referenced or explicitly caveated above where volatility exists (free-model rosters, exact quota numbers)

---
*Research completed: 2026-08-20*
*Ready for roadmap: yes*
