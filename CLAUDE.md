<!-- GSD:project-start source:PROJECT.md -->
## Project

**bonasterio**

Herramienta interna de BONA que arma un "consejo de IAs" para generar presupuestos de trabajos eléctricos. En vez de consultar varios modelos por separado y comparar a mano, arma un debate estructurado entre varios roles/personalidades de IA: cada uno propone un presupuesto, ve las respuestas de los demás, tiene una ronda para ajustar o defender su número, y un rol Árbitro da la conclusión final (rango de precio, justificación, y dónde hubo desacuerdo). Todo el debate queda guardado para comparar después contra lo que Bona terminó facturando realmente.

**Core Value:** Que el rango de presupuesto final que da el Árbitro sea confiable y quede registrado, para poder medir con el tiempo qué tan preciso es el consejo comparado contra lo realmente facturado.

### Constraints

- **Costo**: Cero costo de API permanente — todo modelo de IA usado debe tener free tier oficial documentado; nada de scraping de cuentas consumer.
- **Infraestructura backend**: Google Cloud Run (Docker) — no depende de hardware propio siempre encendido.
- **Base de datos**: Supabase compartido del proyecto `bonapps` (mismo que las demás apps BONA), no un proyecto separado.
- **Frontend**: PWA estática en GitHub Pages, mismo patrón que el resto del ecosistema.
- **Auth**: OTP por email vía Supabase, no magic link — lección ya aprendida en lista-super: magic link no funciona en PWAs instaladas en iOS (storage aislado del navegador que abre el mail).
<!-- GSD:project-end -->

<!-- GSD:stack-start source:research/STACK.md -->
## Technology Stack

## Recommended Stack
### Core Technologies
| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| Python | 3.12 (slim Docker base) | Runtime | Broadest current compatibility across `langgraph`/`langchain-*`/`supabase-py` wheels. 3.13 works too but some ML/AI-adjacent packages still lag on day-1 wheels for new Python minors — 3.12 is the safer default for a project that shouldn't need debugging time on infra. MEDIUM confidence (general ecosystem pattern, not project-specific verification). |
| FastAPI | 0.141.x | HTTP API layer | Current stable (verified via PyPI, 2026-08-20). Async-native, which matters here because each request does several concurrent outbound calls to LLM APIs — you want `async def` handlers + `httpx`/async SDKs end to end, not sync blocking calls. HIGH confidence. |
| Uvicorn | 0.52.x (`uvicorn[standard]`) | ASGI server | Current stable. Run as a **single process**, no Gunicorn (see "What NOT to Use"). HIGH confidence. |
| LangGraph | 1.2.x | Debate orchestration graph | Current stable major (1.x line, verified via PyPI — up from the 0.x versions common in older tutorials/training data). Native `Send()` API is the correct primitive for "fan out to N roles in parallel, fan back in" — exactly this project's round-1 pattern. HIGH confidence (verified via Context7 `/langchain-ai/langgraph`). |
| langchain-core | 1.6.x | Shared message/model abstractions used by LangGraph + the model integration packages below | Pulled in transitively; pin explicitly to avoid resolver drift between `langgraph`, `langchain-google-genai`, and `langchain-openai`. HIGH confidence. |
| Docker (multi-stage build) | — | Packaging for Cloud Run | Cloud Run requires a container image; multi-stage keeps the final image slim (faster cold starts — see Cloud Run section). HIGH confidence. |
| Google Cloud Run (2nd gen execution environment, Service — not Job) | — | Backend hosting | Scale-to-zero fits "no cost when idle" and "no dependency on a home machine always on." Must be a **Service** (request/response), not a Job, because the PWA calls it synchronously per debate. Use the 2nd gen execution environment (default today) for better outbound network throughput to the LLM APIs. HIGH confidence. |
| Supabase (shared `bonapps` project) — Postgres + Auth + Realtime | — | Database, auth, live updates | Already the ecosystem standard (Toolbox, bonapp-gastos, lista-super). Realtime's `postgres_changes` on `INSERT` is exactly the "show rounds as they're written" requirement — no polling, no WebSocket server to run yourself. HIGH confidence. |
| `@supabase/supabase-js` v2 (via CDN, no bundler) | pin `@supabase/supabase-js@2` (jsDelivr/unpkg) | Frontend DB/auth/realtime client | Matches the existing no-build-step PWA pattern used across the BONA ecosystem. `import { createClient } from 'https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/+esm'` (ESM CDN build) works directly in a `<script type="module">`, no npm install needed. HIGH confidence. |
| `supabase-py` | 2.31.x | Backend DB writes (service-role key) | Backend writes to `debate_rounds`/`debates` progressively as each LangGraph node finishes — this is the client that does it, using the **service role key** (bypasses RLS; the anon key + RLS is for the PWA only). HIGH confidence. |
### Supporting Libraries
| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `langchain-google-genai` | 4.3.x | Gemini role(s) via `ChatGoogleGenerativeAI` | Requires Python ≥3.10 (verified via PyPI metadata). Wraps Google's new unified `google-genai` SDK under the hood — do not add `google-generativeai` (legacy, superseded) as a separate direct dependency. |
| `langchain-openai` | 1.6.x | OpenRouter roles, via `ChatOpenAI` pointed at OpenRouter's OpenAI-compatible endpoint | OpenRouter is OpenAI-API-compatible by design. Configure `ChatOpenAI(model="<org>/<model>:free", base_url="https://openrouter.ai/api/v1", api_key=OPENROUTER_API_KEY, default_headers={"HTTP-Referer": "<your-app-url>", "X-Title": "bonasterio"})`. This is the long-established, high-adoption integration path — prefer it over the newer `langchain-openrouter` package (see Alternatives). |
| `pydantic` | 2.13.x | Request/response models, `DebateState` typing support | Already a FastAPI dependency; use `BaseModel`/`TypedDict` for the LangGraph state as shown in the spec. |
| `google-genai` | latest 1.x (transitive via `langchain-google-genai`) | Underlying Gemini SDK | Don't call it directly unless you need a Gemini-specific feature LangChain doesn't expose yet (e.g. some grounding params) — for this project's plain chat-completion use case, the LangChain wrapper is sufficient and keeps all three model providers behind one `ainvoke()` interface. |
| Built-in LangChain retry (`Runnable.with_retry()`) | — | Backoff on OpenRouter 429s / transient Gemini errors | Prefer this over adding a separate `tenacity` dependency — it's already available on every `ChatOpenAI`/`ChatGoogleGenerativeAI` instance and integrates with LangGraph node execution without extra wiring. Configure `stop_after_attempt` low (2-3) since a full debate round budget is only 15-30s total. |
| `httpx` | 0.28.x | Transitive dependency of FastAPI/LangChain/Supabase clients | Don't add manual `requests`-based calls; everything in this stack is already async-`httpx`-based — stay consistent so nothing blocks the event loop. |
| `python-dotenv` | 1.2.x | Local dev env var loading | Dev-only. On Cloud Run, inject secrets via **Secret Manager + `--set-secrets`**, not plain `--set-env-vars`, for the API keys (`OPENROUTER_API_KEY`, `GOOGLE_API_KEY`, `SUPABASE_SERVICE_KEY`). |
### Development Tools
| Tool | Purpose | Notes |
|------|---------|-------|
| `uvicorn --reload` | Local dev server | No Docker needed for day-to-day iteration; point `SUPABASE_URL`/keys at the shared `bonapps` project via `.env` (gitignored). |
| `ruff` | Lint/format | Single tool replacing flake8+black+isort, fast, standard 2025-2026 Python choice. |
| `pytest` + `pytest-asyncio` | Testing | Needed to test async LangGraph nodes and FastAPI async endpoints without real API calls — mock the `ChatOpenAI`/`ChatGoogleGenerativeAI` `.ainvoke()` calls. |
| Google Cloud CLI (`gcloud run deploy`) | Deploy | Simplest path for a single-service, low-traffic app — no need for Terraform/Cloud Build pipelines at this scale; a `gcloud run deploy --source .` from the repo root (Cloud Build auto-builds the Dockerfile) is enough. |
## Installation
# Core
# Model providers
# Supabase (backend)
# Config
# Dev dependencies
## Alternatives Considered
| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|--------------------------|
| Run the LangGraph debate **synchronously inside** the `POST /debates` request handler (client holds the connection open 15-30s) | `FastAPI BackgroundTasks` / `asyncio.create_task` returning immediately, processing "in the background" | Never on Cloud Run's default billing (see What NOT to Use below) — only if you switch the service to `--no-cpu-throttling` (instance-based billing), which costs continuously and conflicts with the zero-cost constraint. |
| `ChatOpenAI` (langchain-openai) pointed at OpenRouter's base URL | `langchain-openrouter` (dedicated integration package, PyPI v0.2.8) | If you specifically need OpenRouter-only features LangChain's generic OpenAI wrapper doesn't expose (e.g. provider-routing metadata in responses). It's real and functional, but very new/low-adoption (single-digit minor version) — extra dependency-freshness risk for a "set it up once, don't come back" internal tool. |
| Single Uvicorn process per Cloud Run container | Gunicorn + multiple Uvicorn workers (`(2*CPU)+1` formula) | At meaningful concurrent traffic (hundreds of req/min). This is a single-user internal tool — Cloud Run's horizontal instance autoscaling is the scaling axis, not in-container multi-processing. |
| Backend persists progress directly to `debates`/`debate_rounds` tables (source of truth) | LangGraph's built-in Postgres checkpointer (`langgraph-checkpoint-postgres`) | If you need to **resume/replay** interrupted graph runs across process restarts (human-in-the-loop, pause/resume). Not needed here — a debate is a single 15-30s synchronous execution; the domain tables already are the persisted record. |
| Cloud Run Service with `min-instances=0` | Cloud Run Jobs, or Cloud Tasks + a worker service | If the debate execution needs to be decoupled from the HTTP request (e.g. very long multi-minute debates, or explicit queueing/retry semantics). Cloud Tasks is the GCP-idiomatic way to do "fire and forget" — not `asyncio` background tasks (see below). |
| Configurable model IDs per role (`debate_roles.modelo`), including `"openrouter/free"` auto-router as a safe default/fallback | Hardcoding specific free model IDs (e.g. a specific Llama/DeepSeek slug) in code | Never as the only option — OpenRouter's free-model roster rotates (DeepSeek and Mistral free variants that existed earlier in 2026 are gone as of mid-2026, replaced by others). Store the model ID in the DB (already the plan) so a rate-limited or deprecated model is a data edit, not a redeploy. |
## What NOT to Use
| Avoid | Why | Use Instead |
|-------|-----|--------------|
| `FastAPI BackgroundTasks` / detached `asyncio.create_task()` to run the LangGraph debate after returning `debate_id` immediately | **Critical, verified pitfall.** Cloud Run's default billing is *request-based*: "CPU is only allocated during request processing." Per Google's own docs, work that continues after the HTTP response has been sent is not guaranteed CPU — background tasks. can stall or silently never finish, leaving debates stuck in `en_curso` forever, and it's hard to detect because it "works" in local dev (where CPU isn't throttled) and breaks intermittently in prod. | Run the graph **inside** the awaited request handler (client holds the connection for the full 15-30s). Since the PWA needs a `debate_id` to subscribe to Realtime *before* rows exist, have the **PWA generate the UUID client-side** (`crypto.randomUUID()`), POST it as part of the debate payload, and subscribe to `postgres_changes` filtered on that id in parallel with firing the POST — no early-return trick needed, and everything stays inside one CPU-allocated request lifecycle. If you truly need decoupled background execution later, use Cloud Tasks or `--no-cpu-throttling`, not in-process async tasks. |
| Gunicorn + N Uvicorn workers as the default Cloud Run entrypoint | Adds process-management complexity (worker restarts, memory ×N, duplicate connection pools) for a workload where Cloud Run itself is already the horizontal scaler; at this traffic volume it's pure overhead, and multi-process containers complicate Cloud Run's SIGTERM graceful-shutdown contract. | Single `uvicorn app.main:app --host 0.0.0.0 --port 8080` process per container; let Cloud Run's instance autoscaling (`min-instances=0`, small `max-instances`) handle load. |
| Anthropic/OpenAI paid chat APIs | Explicit project constraint: zero ongoing API cost, no trial-credit-then-billing traps. | Gemini official free tier + OpenRouter `:free` models, exactly as decided. |
| `gemini-2.0-flash` as a "free" model choice | Per Google's own pricing page, `gemini-2.0-flash` is deprecated and being shut down (June 1, 2026 — already past as of this research date). It will hard-fail role calls once removed. | `gemini-2.5-flash` or `gemini-2.5-flash-lite` — both confirmed "free of charge" on the current official pricing page. |
| `gemini-2.5-pro` as a "free" model choice | Per Google's own pricing page, 2.5 Pro is explicitly **not available on the free tier** (paid-only). Using it will incur cost or hard-fail with no billing account attached — directly violates the zero-cost constraint. | Stick to Flash-tier Gemini models for all free-tier roles. |
| Magic-link email auth | Already-learned lesson (lista-super): breaks in installed iOS PWAs because the link opens in Safari, which has isolated storage from the installed PWA's context, so the session never reaches the app. | Supabase email OTP **code** flow: `signInWithOtp({ email })` then `verifyOtp({ email, token, type: 'email' })`, with the Supabase email template using `{{ .Token }}` instead of `{{ .ConfirmationURL }}`. |
| Manual polling loop (`setInterval` + `GET /debates/{id}`) for live updates | Explicitly out of scope per PROJECT.md — Realtime already does this better (push, not poll) and the original spec's polling suggestion predates the Realtime decision. | Supabase Realtime `postgres_changes` subscription on `debate_rounds` filtered by `debate_id=eq.<id>`, `event: 'INSERT'`. |
| A message queue / worker system (Celery, RQ + Redis) | Massive over-infrastructure for a single-user internal tool running a handful of debates a day; adds a stateful service that breaks the "scale to zero, pay nothing when idle" property Cloud Run gives you for free. | Synchronous in-request execution (see above), or Cloud Tasks if/when true decoupling is needed. |
## Stack Patterns by Variant
- Keep Cloud Run's default request-based billing (cheapest, fits free tier).
- Cloud Run's default request timeout is 300s — well above the 15-30s debate budget; no need to raise it.
- Set container `concurrency` low-to-moderate (e.g. 8-10) — each in-flight request is mostly waiting on external LLM API I/O, not CPU, so Cloud Run can multiplex several concurrent debates per instance without contention; but this is a single-user tool, so this is a non-issue in practice, just don't leave it at the default 80 blindly.
- `min-instances=0` for true scale-to-zero; accept one cold start (a few seconds) on the first debate after idle — acceptable for an internal tool used sporadically.
- Use **Cloud Tasks** to enqueue the debate job to a second endpoint, or switch the service to `--no-cpu-throttling` (instance-based billing). Both cost more than the request-based default — budget accordingly, this breaks the current zero-cost assumption if left running continuously.
- Round 1 (initial budgets): conditional edge from `START` returning a list of `Send("nodo_presupuesto_inicial", {...})`, one per active non-arbiter role — this is LangGraph's native map-reduce fan-out, exactly matching "ask all roles in parallel."
- Accumulate results with an `Annotated[list[RondaResult], operator.add]` reducer field on `DebateState` — required because multiple parallel node instances write to the same state key in the same superstep; without a reducer LangGraph will raise on the second concurrent write (`LastValue` channels reject >1 write per step).
- Round 2+ (adjustment rounds): same `Send()` fan-out pattern, reading `rondas` accumulated so far as context for each role's next call.
- Final node (`nodo_arbitro`): a plain edge (not `Send`) after the last adjustment round, since it's a single node reading the full accumulated state — no fan-out needed there.
- `nodo_persistencia` writes to Supabase as each node completes rather than batching at the end (already in the spec) — this is also what makes Realtime feel "live" to the PWA, since each insert commits and pushes independently mid-execution.
## Version Compatibility
| Package A | Compatible With | Notes |
|-----------|------------------|-------|
| `langgraph==1.2.x` | `langchain-core==1.6.x` | Both on the current 1.x majors as of 2026-08-20; pin both explicitly, don't let pip resolve loosely — the 0.x→1.x jump across the LangChain/LangGraph ecosystem (common in older tutorials/training data) had breaking API changes (e.g. `StateGraph` construction, `Send` import path). |
| `langchain-google-genai==4.3.x` | Python `>=3.10,<4.0` | Verified via PyPI metadata — compatible with the recommended Python 3.12 base image. |
| `langchain-openai==1.6.x` | `langchain-core==1.6.x` | Same major-line requirement as above. |
| `supabase==2.31.x` (Python) | PostgREST/GoTrue/Realtime server versions on Supabase's hosted platform | The hosted `bonapps` project auto-tracks current server versions; no manual server-side version pinning needed since it's not self-hosted. |
| `@supabase/supabase-js@2` (CDN) | Same hosted `bonapps` project | Pin to `@2` (major) via the CDN URL, not to an exact patch, so the PWA picks up compatible patch fixes automatically without a redeploy step (there is no build step to redeploy through). |
## Sources
- Context7 `/langchain-ai/langgraph` — `Send` API, map-reduce fan-out, state reducer accumulation (`apply_writes`), conditional edges. HIGH confidence.
- Context7 `/supabase/supabase-py`, `/supabase/realtime` — OTP client methods, `postgres_changes` channel/event/filter shape. HIGH confidence.
- [Cloud Run CPU allocation docs](https://cloud.google.com/run/docs/configuring/cpu-allocation) — request-based vs instance-based billing, background task behavior after response. HIGH confidence, directly verified via WebFetch.
- [OpenRouter API rate limits](https://openrouter.ai/docs/api-reference/limits) — 20 req/min, 50/day (<$10 purchased) vs 1000/day (≥$10 purchased) for `:free` models. HIGH confidence, official docs.
- [Google Gemini API pricing](https://ai.google.dev/gemini-api/docs/pricing) — 2.5 Flash / Flash-Lite free, 2.5 Pro not on free tier, 2.0 Flash deprecated (shutdown June 1 2026). HIGH confidence, official docs, verified 2026-08-20.
- PyPI JSON API (`fastapi`, `uvicorn`, `langgraph`, `langchain-core`, `langchain-google-genai`, `langchain-openai`, `supabase`, `pydantic`, `python-dotenv`, `gunicorn`, `langchain-openrouter`) — current version numbers, verified directly 2026-08-20. HIGH confidence.
- WebSearch (multiple queries, cross-referenced) — OpenRouter free-model roster rotation (DeepSeek/Mistral free variants gone mid-2026, replaced by Nemotron/GLM/Gemma), Cloud Run Gunicorn-vs-single-worker community debate. MEDIUM confidence — roster specifics change often, verify at `openrouter.ai/models?max_price=0` at implementation time rather than trusting any hardcoded list, including this one.
- Existing BONA ecosystem pattern (user's own prior projects: lista-super, bonapp-gastos) — magic-link-breaks-iOS-PWA lesson, no-build-step CDN pattern. HIGH confidence (already validated in production by the user).
<!-- GSD:stack-end -->

<!-- GSD:conventions-start source:CONVENTIONS.md -->
## Conventions

Conventions not yet established. Will populate as patterns emerge during development.
<!-- GSD:conventions-end -->

<!-- GSD:architecture-start source:ARCHITECTURE.md -->
## Architecture

Architecture not yet mapped. Follow existing patterns found in the codebase.
<!-- GSD:architecture-end -->

<!-- GSD:skills-start source:skills/ -->
## Project Skills

No project skills found. Add skills to any of: `.claude/skills/`, `.agents/skills/`, `.cursor/skills/`, `.github/skills/`, or `.codex/skills/` with a `SKILL.md` index file.
<!-- GSD:skills-end -->

<!-- GSD:workflow-start source:GSD defaults -->
## GSD Workflow Enforcement

Before using Edit, Write, or other file-changing tools, start work through a GSD command so planning artifacts and execution context stay in sync.

Use these entry points:
- `/gsd-quick` for small fixes, doc updates, and ad-hoc tasks
- `/gsd-debug` for investigation and bug fixing
- `/gsd-execute-phase` for planned phase work

Do not make direct repo edits outside a GSD workflow unless the user explicitly asks to bypass it.
<!-- GSD:workflow-end -->



<!-- GSD:profile-start -->
## Developer Profile

> Profile not yet configured. Run `/gsd-profile-user` to generate your developer profile.
> This section is managed by `generate-claude-profile` -- do not edit manually.
<!-- GSD:profile-end -->
