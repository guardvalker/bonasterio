# Pitfalls Research

**Domain:** Multi-LLM debate/orchestration system (LangGraph + FastAPI on Cloud Run + Supabase), built entirely on free-tier AI APIs
**Researched:** 2026-08-20
**Confidence:** MEDIUM-HIGH (Cloud Run/Supabase claims verified against official docs/pricing; OpenRouter/Gemini free-tier numbers are volatile and explicitly called out as snapshots, not guarantees)

## Critical Pitfalls

### Pitfall 1: Background execution model collides with Cloud Run's CPU throttling

**What goes wrong:**
The spec's own suggested design — `POST /debates` returns a `debate_id` immediately with status `en_curso`, then the LangGraph run continues "in the background" while the PWA watches via Supabase Realtime/polling — is the single most dangerous architectural assumption in this project. Cloud Run, by default, only allocates CPU to a container **while it is actively handling a request**. The instant the HTTP response is sent, CPU is throttled to near-zero. Any work kicked off via `BackgroundTasks`, `asyncio.create_task`, or a detached thread after the response is returned can stall indefinitely or simply never finish — the debate gets stuck in `en_curso` forever with no error logged.

**Why it happens:**
This is the single most common Cloud Run gotcha, precisely because it works perfectly in local dev (where the process keeps running) and even sometimes in production for a while (the instance may stay warm and finish the work before being recycled), so it looks done until an instance is reused for another request or scaled to zero mid-debate.

**How to avoid:**
- Prefer the simple, correct option for this project's scale: **keep the HTTP request open for the full debate duration** (15-30s is well within Cloud Run's 300s default timeout and its 3600s max) and return the complete result, or return partial state as the debate writes each round to Supabase incrementally and let the PWA read progress via Realtime/polling on `debate_rounds`/`debates` while the *original* request is still in flight server-side.
- If background-after-response is truly wanted later, that requires either (a) Cloud Run "CPU always allocated" — which is a billing setting, not a free perk, and directly conflicts with the "cero costo permanente" constraint — or (b) offloading to Cloud Tasks / Pub/Sub with a second endpoint, which adds real infra complexity for a solo-user internal tool.
- Simplest correct v1: synchronous request that runs the whole graph server-side, with `nodo_persistencia` writing each round to Supabase *as it happens* (already planned) so Realtime gives the "watch it live" UX without needing true background execution.

**Warning signs:**
- Debates that get stuck in `en_curso` status with no `error` and no further rounds appearing.
- Works fine when testing continuously (warm instance) but breaks after a period of no traffic (cold instance recycled).

**Phase to address:** Backend/graph execution phase (before deployment phase) — this is a foundational request-lifecycle decision, not something to patch later.

---

### Pitfall 2: Free-tier LLM pool exhaustion during parallel fan-out

**What goes wrong:**
A single debate round fans out to 3+ models in parallel, then repeats for N rounds, then hits the arbiter — easily 8-12+ LLM calls per debate. OpenRouter's free pool caps unfunded accounts at roughly 20 requests/minute and 50 requests/day (verified via OpenRouter's own docs/FAQ and third-party trackers as of 2026); Gemini's free tier is similarly capped per-model (low single/double-digit RPM, low hundreds RPD depending on model). During active development — where you might run 10-20 test debates in an afternoon — it is very easy to blow through the **daily** cap, not just the per-minute one, and get silently throttled or hard-errored mid-debate.

**Why it happens:**
Rate limits on free tiers are designed for occasional/hobbyist use, not iterative development. Free-tier requests also sit behind paid traffic in the provider's queue, so under load you get slower and less predictable responses, not just outright rejections.

**How to avoid:**
- Wrap every external model call in retry-with-backoff and treat 429s as expected, not exceptional.
- Track daily call counts per provider (even a simple counter row in Supabase) so you can see quota burn during dev without guessing.
- Consider the one-time $10 OpenRouter credit purchase (raises daily cap from 50 to 1,000) as a pragmatic dev-time mitigation — flag this explicitly as a decision point since it's a one-time cost that touches the "cero costo permanente" constraint (it's a balance top-up, not a per-call cost, but worth calling out rather than silently doing it).
- Build a "dry run" / mock-model mode for local development so you're not burning real quota testing the graph shape, persistence, and UI.

**Warning signs:**
- Debates that fail partway through only during periods of heavy manual testing.
- Errors that correlate with time-of-day (quota resets) rather than input size/complexity.

**Phase to address:** Backend integration phase — build the retry/backoff and quota-awareness in from the first real API call, not as a later hardening pass.

---

### Pitfall 3: Free-model lineup rotates without notice, silently breaking hardcoded role→model mappings

**What goes wrong:**
OpenRouter's free model catalog is not stable. Models with a `:free` suffix are added and withdrawn based on provider spare capacity — models that were popular free options previously have since become paid-only or been removed entirely. Since `debate_roles.modelo` stores a specific model string and roles are editable from the PWA without code, a role can silently start failing (404/model-not-found, or worse, silently start routing to a *paid* model if the `:free` suffix is dropped/mistyped) with zero visibility into why a debate suddenly errors or costs money.

**Why it happens:**
People treat "I picked a free model once" as a permanent fact rather than a snapshot. There's no built-in validation step when a role is created/edited in the PWA that checks the model ID is (a) real and (b) still on the free tier.

**How to avoid:**
- Always use the exact `:free`-suffixed model ID (never assume that omitting it defaults to free — OpenRouter will route to the paid version if the account has any credit balance).
- Validate model IDs against OpenRouter's live `/models` endpoint (or a periodically-refreshed cached list) when a role is saved in the PWA, rejecting unknown/non-free IDs.
- Add a lightweight periodic health check (even a manual "test this role" button in the Config screen) that makes one cheap call per active role and flags roles whose model has disappeared.
- Log the exact model ID used on every `debate_rounds` row (not just a generic label) so historical debates remain interpretable even after a model is deprecated.

**Warning signs:**
- A role that worked last week suddenly errors on every debate.
- Unexpected charges on an account that's supposed to be zero-cost (a dropped `:free` suffix silently routing to a paid endpoint is the single scariest version of this).

**Phase to address:** Role management (CRUD) phase — validation belongs with role creation/editing, not bolted on later.

---

### Pitfall 4: Inconsistent structured-output/JSON support across free models breaks price extraction

**What goes wrong:**
Structured output / JSON mode support on OpenRouter is determined **per provider endpoint, not per model** — the same nominal model may be served by multiple backing providers, only some of which support `response_format`/JSON schema enforcement, and that support can change over time. Free-tier variants are especially likely to be served by whichever provider has spare capacity, which may not be the one with the best structured-output support. Relying on `response_format: json_schema` working uniformly across Gemini + several different OpenRouter free models is not a safe assumption — some will comply, some will ignore it and return prose, some will wrap valid JSON in markdown fences or add conversational preamble ("Sure! Here's the estimate:").

**Why it happens:**
Structured output is a relatively recent, unevenly-adopted feature; free/open-weight models used via OpenRouter often have weaker instruction-following for strict format compliance than flagship paid models, and the routing layer adds another point of inconsistency.

**How to avoid:**
- Do not depend on JSON mode alone. Design the price extraction as a **layered fallback**: (1) try structured output where the endpoint reports support for it, (2) prompt for a clearly delimited plain-text pattern (e.g., `RANGO: $X - $Y`) that's easy to regex regardless of JSON support, (3) regex/parse the free-text response as the universal fallback that always runs regardless of what mode was requested.
- Always store the full raw `contenido` text (already in the planned schema) so extraction can be re-run/improved later without re-querying the model.
- See Pitfall 8 for the extraction logic itself.

**Warning signs:**
- `presupuesto_propuesto_min/max` is null for specific roles/models more often than others — a strong signal that model's structured-output support is unreliable and the fallback path needs strengthening for it.

**Phase to address:** Price extraction / structured-output phase — design the fallback chain up front, don't assume JSON mode "just works" across the whole role roster.

---

### Pitfall 5: LangGraph parallel fan-out with unmanaged state reducers loses data

**What goes wrong:**
When `nodo_presupuesto_inicial` fans out to multiple role nodes in parallel and each writes its result back into shared `DebateState`, the default LangGraph state-merge behavior is **last-write-wins** for any field without an explicit reducer. If `rondas: list[RondaResult]` isn't annotated with a merge reducer (e.g., `Annotated[list, operator.add]`), parallel writes from different roles in the same superstep can silently overwrite each other instead of accumulating — you'd see a debate finish with only one role's response saved even though all three "ran."

**Why it happens:**
This is the most-cited LangGraph gotcha in community reports: it's invisible in sequential/single-branch testing and only manifests once real parallelism kicks in, which is exactly this project's core pattern (parallel fan-out is a named requirement).

**How to avoid:**
- Annotate every state field that parallel nodes write to with an explicit reducer (list concatenation for `rondas`, or a dict-merge keyed by role for per-role results) from the very first version of the graph, not after noticing missing data.
- Write a test that runs the fan-out node with 3+ mocked roles and asserts all 3 results are present in the resulting state — this is cheap to write and catches the bug immediately, before it ever reaches a real API call.

**Warning signs:**
- Debate history shows fewer `debate_rounds` rows than `roles × rounds` should produce, with no corresponding error logged.

**Phase to address:** Graph/state design phase — this must be correct before the first parallel round is ever run against a real model.

---

### Pitfall 6: Unbounded round-loop with no hard exit condition

**What goes wrong:**
A conditional loop (round 1 → round 2 → ... → arbiter) that decides whether to continue based on model output (e.g., "did they converge?") or a config value read at runtime can loop far longer than intended if the condition logic has an edge case — documented real-world cases include validators looping 80+ iterations on malformed input before anyone noticed. Even with a fixed `cantidad_rondas` from config (as currently planned, which is the safer design), a bug in the conditional edge routing could still create a cycle that never reaches `END`.

**Why it happens:**
Conditional edges are easy to get subtly wrong (off-by-one on round counting, a condition that's never true/always true), and LangGraph won't stop you — it will just keep executing, burning API quota (see Pitfall 2) on every extra iteration.

**How to avoid:**
- Always include an explicit iteration counter in `DebateState` (`ronda_actual`) with a hard ceiling check in the conditional edge — regardless of config value — that forces a transition to the arbiter node no matter what.
- Never let the loop-continuation decision depend solely on LLM-generated content (e.g., "the model said it agrees") without a hard numeric backstop.

**Warning signs:**
- A debate that takes dramatically longer than expected, or one that burns far more API calls than `roles × (rounds + 1)` would predict.

**Phase to address:** Graph/state design phase, alongside Pitfall 5.

---

### Pitfall 7: RLS skipped or under-scoped because "it's just for one user"

**What goes wrong:**
Because this is a solo-user internal tool sharing the existing `bonapps` Supabase project, there's a strong temptation to either skip RLS on the new tables (`debates`, `debate_rounds`, `debate_roles`, `debate_config`) entirely, or write an overly-broad policy like `USING (true)`. But the PWA is a static site on GitHub Pages shipping the Supabase anon key in its JS bundle — that key is public to anyone who opens dev tools. If RLS is off or too broad on these tables, **anyone on the internet with that anon key can read/write the entire debate history**, including any future debates from other BONA ecosystem users sharing the same Supabase project.

**Why it happens:**
"Single user" is treated as a security boundary, but the actual security boundary is "anyone with the public anon key," which is a much larger set of people than intended, especially once the key is committed to a public GitHub Pages repo.

**How to avoid:**
- Enable RLS on all four new tables from the first migration, with default-deny.
- Even for a nominally single-user app, add an `owner_id`/`user_id` column tied to the OTP-authenticated Supabase user and scope every policy to `auth.uid() = owner_id` — this is already consistent with the project's decision to use account-based OTP login (partly "in case another team member uses it later"), so the schema should assume multi-user from day one even if only one person uses it initially.
- Test RLS by attempting reads/writes as a different (or anonymous, unauthenticated) user and confirming they're rejected — don't just test the happy path as the logged-in owner.

**Warning signs:**
- Any table in the Supabase dashboard showing "RLS disabled."
- A policy using `USING (true)` or no `WITH CHECK` clause on insert/update policies.

**Phase to address:** Database schema phase — RLS policies should ship in the same migration as the tables, not as a follow-up.

---

### Pitfall 8: Supabase Realtime silently returns nothing because the table isn't in the publication

**What goes wrong:**
Enabling RLS correctly is necessary but not sufficient for Realtime to work — tables must also be explicitly added to the `supabase_realtime` publication (`ALTER PUBLICATION supabase_realtime ADD TABLE debate_rounds;`) for `postgres_changes` events to fire at all. This is a separate, easy-to-forget step from RLS, and when it's missed the failure mode is silent: no errors, the subscription just never receives events, and the "live" debate view looks broken with no obvious cause.

**Why it happens:**
RLS and Realtime publication are two different Postgres/Supabase concepts that get conflated because they're both "permission-adjacent," so people configure one and assume the other follows automatically.

**How to avoid:**
- Explicitly add every table the PWA subscribes to (`debate_rounds`, `debates`) to the realtime publication as part of the schema migration, and verify it in the Supabase dashboard's Replication settings.
- Confirm the RLS SELECT policy that gates realtime access actually matches the columns/filters the client subscribes with (Realtime enforces the same RLS SELECT policy per-row as regular reads).

**Warning signs:**
- The "live debate" view never updates even though rows are confirmed to be inserting via direct Supabase dashboard inspection.

**Phase to address:** Same database schema phase as Pitfall 7 — ship publication config alongside RLS policies.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|-----------------|------------------|
| Hardcode 3 roles in Python instead of reading from `debate_roles` | Faster to validate the graph flow works at all | Have to rewrite the model-dispatch layer once dynamic roles land; risk of divergence between hardcoded prompts and DB-configured ones | Only for the very first "does the graph run end-to-end" spike — spec itself suggests this, keep it short-lived |
| Regex-only price extraction, no structured output attempt | Simple, works across every model regardless of JSON support | Misses well-formed numbers wrapped in odd phrasing; more manual-review debates | Acceptable for MVP if paired with a "needs review" flag and raw text always stored — never acceptable as the *only* long-term strategy |
| `USING (true)` RLS policy "temporarily" while building | Unblocks frontend development fast | Real security hole if shipped, and easy to forget to tighten later | Never acceptable even temporarily on a repo connected to a real Supabase project — use a real dev/staging project instead if you need to move fast |
| Fixed `cantidad_rondas` with no convergence detection (as scoped for v1) | Simple, predictable cost/time per debate | Can't cut a debate short when models agree quickly, or extend when they're still diverging | Acceptable long-term per project's own Out of Scope decision — just ensure the hard round-count ceiling (Pitfall 6) exists regardless |

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|-----------------|-------------------|
| OpenRouter | Omitting `:free` suffix, silently billing a paid model once account has any credit | Always pin the exact `:free`-suffixed model ID; validate against live model list at role save time |
| Gemini API | Assuming one universal free-tier limit across all Gemini model variants | Rate limits differ per model (Flash vs Flash-Lite vs Pro); Pro was removed from free tier in 2026 — pin to a model confirmed still on the free tier and re-check periodically |
| Cloud Run + Supabase service key | Baking `SUPABASE_SERVICE_KEY` into the Docker image or a plaintext env var in a public repo/CI config | Store via Secret Manager / Cloud Run's native secret mounting, never in the image or committed `.env` |
| Cloud Run background execution | Returning HTTP response then continuing LangGraph execution in a background task/thread | Keep the debate-driving request open until the graph completes (or explicitly design for Cloud Tasks if async execution is truly needed later) |
| Supabase Realtime | Enabling RLS but forgetting to add the table to the `supabase_realtime` publication | Add tables to the publication explicitly in the same migration as RLS policies |

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|-----------------|
| Sync HTTP clients (e.g., `requests`, sync `google-generativeai` calls) used inside async FastAPI/LangGraph nodes | "Parallel" fan-out nodes actually execute sequentially, round time balloons | Use async HTTP clients or wrap sync SDK calls in `asyncio.to_thread`/a thread pool | Immediately noticeable once 3 real models are called — round time becomes sum instead of max of individual latencies |
| No per-model timeout on individual LLM calls | One slow/hung free-tier model call stalls the entire round and risks hitting Cloud Run's request timeout | Set an explicit per-call timeout (e.g., 15-20s) shorter than the overall request budget, with a documented "role skipped this round" fallback rather than blocking everyone | As soon as any one free model has a slow day — free-tier traffic queues behind paid traffic, so this is common, not rare |
| Free-tier daily quota shared across dev + prod usage | Dev testing exhausts the same daily cap production debates need | Separate API keys for dev/test vs. the "real" account used for actual work, or a mock-model dev mode | Any day with more than a handful of manual test debates |

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Treating "solo-user app" as a reason to skip RLS or `owner_id` scoping | Full read/write DB exposure to anyone with the public anon key (shipped in the PWA's client-side bundle) | Enable RLS + `owner_id`-scoped policies on every new table regardless of expected user count |
| Using `SUPABASE_SERVICE_KEY` anywhere reachable from the PWA/frontend | Total bypass of RLS, full admin DB access if leaked | Service key lives only in Cloud Run's server-side environment, mounted via Secret Manager, never in frontend code or client-callable endpoints |
| Storing LLM API keys as plain env vars baked into the Docker image | Keys leak if the image is ever pushed to a registry or repo, or via container inspection | Use Cloud Run's Secret Manager integration for `GEMINI_API_KEY`/`OPENROUTER_API_KEY`, not `ENV` lines in the Dockerfile |
| No validation on role `system_prompt`/`modelo` fields editable from the PWA | A malformed or malicious model ID string could cause unexpected API routing (see Pitfall 3) | Validate model ID against a known-good/live list on save; treat role config as data that reaches an external API, not inert text |

## UX Pitfalls

| Pitfall | User Impact | Better Approach |
|---------|-------------|-------------------|
| Debate appears "stuck" in `en_curso` with no feedback when a model call fails or times out | Bona can't tell if it's still working or dead, has to guess whether to wait or retry | Surface partial failures explicitly per role/round (e.g., "Perito de Mercado didn't respond this round") instead of only showing success or total silence |
| Price range extraction silently fails (nulls) with no visible flag | Historical debates end up with unusable data for the accuracy-tracking goal that is this project's stated Core Value | Flag rows where extraction failed/low-confidence as "needs manual review" in the UI, don't let them blend invisibly into the dataset |
| No indication of which specific model/version produced a given round's opinion | Bona can't tell, months later, whether a "Perito AAIERIC" opinion came from a model that has since been deprecated/changed | Store the exact model ID (not just role name) per `debate_rounds` row, and surface it in the expandable timeline |

## "Looks Done But Isn't" Checklist

- [ ] **Background debate execution**: Often "works" during dev with a warm Cloud Run instance — verify it survives a cold start / period of inactivity before trusting it in production.
- [ ] **Price range extraction**: Often tested only against clean, well-formatted model outputs — verify against a model that ignores JSON mode and returns prose ("aproximadamente $150.000 a $200.000 dependiendo de materiales").
- [ ] **RLS policies**: Often verified only as the logged-in owner — verify explicitly with an unauthenticated/anon request that it's rejected.
- [ ] **Realtime "live" updates**: Often looks done because rows insert correctly — verify the publication config, not just the insert, and test from a second browser/device.
- [ ] **Role CRUD from the PWA**: Often only tested with valid, currently-active model IDs — verify what happens when a saved role's model has since rotated out of the free tier.
- [ ] **Round loop termination**: Often only tested with the default config's round count — verify the hard ceiling actually fires if the conditional logic is bypassed/misconfigured.

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|----------------|------------------|
| Background-execution debates stuck in `en_curso` | LOW | Add a "stale debate" sweep (cron or manual endpoint) that marks debates past a reasonable age as `error`, then move to the synchronous-request design |
| A role's model silently rotated out / started billing | MEDIUM | Add model-ID validation against live catalog; retroactively audit `debate_rounds.modelo` history to spot when a role started failing |
| Missing price extraction on historical debates | LOW | Since raw `contenido` is always stored, re-run extraction logic against historical rows once the extraction pipeline improves |
| RLS gap discovered after some data already exposed | MEDIUM-HIGH | Lock down policies immediately, rotate any keys if service key was involved, audit Supabase logs for anomalous anon-key access patterns |

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|-------------------|----------------|
| Background execution vs. Cloud Run CPU throttling | Backend/graph execution design (before deployment) | Deploy early, let the instance go cold, confirm a debate started after a cold start still completes |
| Free-tier quota exhaustion during parallel fan-out | Backend integration (first real API wiring) | Load a debate history table with call counts; confirm retry/backoff triggers on induced 429s |
| Free-model rotation breaking role→model mapping | Role management CRUD phase | Attempt to save a role with a known-deprecated/invalid model ID, confirm it's rejected |
| Inconsistent JSON support across free models | Price extraction phase | Run extraction against a deliberately non-JSON, prose-only mock response and confirm graceful fallback |
| LangGraph state reducer conflicts on fan-out | Graph/state design phase | Unit test fan-out node with 3+ mocked roles, assert all results present in merged state |
| Unbounded round loop | Graph/state design phase | Force the continuation condition to always return "continue" in a test and confirm the hard ceiling still exits |
| RLS skipped/under-scoped for "single user" | Database schema phase (same migration as tables) | Attempt read/write as unauthenticated and as a different user; both must fail |
| Realtime publication not configured | Database schema phase (same migration as RLS) | Insert a row directly in SQL, confirm the PWA's live subscription receives the event |

## Sources

- [OpenRouter Free Tier 2026: Rate Limits, Models, BYOK](https://klymentiev.com/blog/openrouter-free-tier) — MEDIUM confidence, third-party but consistent with OpenRouter's own FAQ
- [OpenRouter FAQ](https://openrouter.ai/docs/faq) — HIGH confidence, official
- [OpenRouter Free Models in 2026: Limits and Catches](https://ask-coreai.com/blog/openrouter-free-models-2026-limits-catches) — MEDIUM confidence
- [OpenRouter Structured Outputs docs](https://openrouter.ai/docs/guides/features/structured-outputs) — HIGH confidence, official
- [OpenRouter Response Healing announcement](https://openrouter.ai/announcements/response-healing-reduce-json-defects-by-80percent) — HIGH confidence, official, confirms JSON defects are common enough to warrant a platform-level fix
- [Gemini API Free Tier Complete Guide 2026](https://www.aifreeapi.com/en/posts/gemini-api-free-tier-complete-guide) — MEDIUM confidence, third-party, numbers should be re-verified against Google's official quota docs at implementation time
- [Gemini API Rate Limits: Free Tier Quotas 2026](https://tinkerllm.com/blog/gemini-api-free-tier-limits-rate-quotas/) — MEDIUM confidence
- [LangGraph State Reducer Conflicts + Concurrent Updates (2026)](https://itsourcecode.com/runtimeerror/langgraph-state-reducer-conflicts-fix/) — MEDIUM confidence, corroborated by official LangGraph fan-out/fan-in docs pattern
- [LangGraph Parallel Execution: Fan-Out and Fan-In Patterns](https://markaicode.com/langgraph-parallel-fan-out-fan-in/) — MEDIUM confidence
- [Use the graph API — LangChain/LangGraph official docs](https://docs.langchain.com/oss/python/langgraph/use-graph-api) — HIGH confidence, official
- [Cloud Run pricing — official](https://cloud.google.com/run/pricing) — HIGH confidence, official
- [Configure request timeout for services — Cloud Run official docs](https://docs.cloud.google.com/run/docs/configuring/request-timeout) — HIGH confidence, official
- [Cloud Run container stops processing background FastAPI tasks — Google Developer forums](https://discuss.google.dev/t/cloud-run-container-stops-processing-when-running-parallel-fastapi-background-tasks/179730/3) — MEDIUM confidence, community-reported but consistent with Cloud Run's documented CPU-allocation model
- [Cloud Run "always-on CPU allocation" — official Google Cloud blog](https://cloud.google.com/blog/products/serverless/cloud-run-gets-always-on-cpu-allocation) — HIGH confidence, official
- [Supabase troubleshooting: service role key / RLS](https://supabase.com/docs/guides/troubleshooting/why-is-my-service-role-key-client-getting-rls-errors-or-not-returning-data-7_1K9z) — HIGH confidence, official
- [Supabase Row Level Security Guide 2026](https://www.agilesoftlabs.com/blog/2026/06/supabase-row-level-security-guide-2026) — MEDIUM confidence
- [Securing your data — Supabase official docs](https://supabase.com/docs/guides/database/secure-data) — HIGH confidence, official
- [10 Common Supabase Security Misconfigurations](https://modernpentest.com/blog/supabase-security-misconfigurations) — MEDIUM confidence

---
*Pitfalls research for: bonasterio — multi-LLM debate council for electrical work budget estimates*
*Researched: 2026-08-20*
