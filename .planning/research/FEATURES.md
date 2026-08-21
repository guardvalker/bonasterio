# Feature Research

**Domain:** Multi-agent AI debate/consensus tool for internal budget estimation (electrical contracting)
**Researched:** 2026-08-20
**Confidence:** MEDIUM-HIGH

## Feature Landscape

Two adjacent domains inform this tool, since nothing exactly like it exists as an off-the-shelf product:

1. **Multi-agent LLM debate / LLM-as-judge ensemble systems** — academic + applied research on how debate/consensus architectures should behave (rounds, roles, disagreement handling, bias). HIGH confidence, multiple 2025-2026 arXiv sources and industry eval-tooling blogs (orq.ai, DeepEval, FutureAGI) agree on the same core patterns.
2. **AI-assisted contractor estimating/quoting software** (BuildOps, PataBid, QuoteIQ, STACK) — commercial products aimed at *customer-facing* quoting. MEDIUM confidence (vendor marketing content), and importantly **most of their feature set is explicitly out of scope here** because bonasterio is an internal single-user decision-support tool, not a customer quoting/invoicing product.

The gap between these two domains is exactly where the anti-features list below comes from: bonasterio should borrow the debate/consensus *mechanics* from domain 1, and borrow almost nothing customer-facing from domain 2.

### Table Stakes (Users Expect These)

Features a debate/consensus tool + an estimation tool are assumed to have. Missing these makes the tool feel broken or untrustworthy, not just "basic."

| Feature | Why Expected | Complexity | Notes |
|---------|--------------|------------|-------|
| Independent first-round proposals (no cross-contamination) | Core to debate research: agents must commit to an initial position before seeing others, or the "debate" is just one model's opinion echoed 3 times. Already in spec (`nodo_presupuesto_inicial` runs in parallel). | LOW | Already designed correctly — parallel calls, not sequential. |
| Visible disagreement, not just a final number | LLM-as-judge research: hiding where judges disagree throws away the most useful signal — disagreement is what tells you to trust or distrust a verdict. Users of any multi-model system expect to see *why* the number moved, not just the number. | LOW-MED | Spec already requires arbiter to note "dónde hubo desacuerdo" — keep this as a first-class field, not buried prose. |
| Structured job input (task type, materials, zone, complexity) | Freeform text alone produces inconsistent estimates across roles; every AI-estimating tool in the commercial space anchors on structured inputs (scope, materials, location) before generating a number. | LOW-MED | Already in requirements. Structure doesn't need to be rigid dropdowns — labeled free-text fields are enough for a solo user. |
| Live/streaming progress while debate runs | 15-30s waits with a blank screen read as "broken" to any user of a multi-step AI tool in 2026 — this is now a baseline UX expectation for anything that takes >5s, debate tools included. | LOW-MED | Spec already covers this via polling + Supabase Realtime; no need for WebSocket. |
| Full expandable history of every round ("who said what, when") | This is the entire value proposition of a debate tool vs. asking one model — if you can't audit the reasoning trail, you might as well have just asked one model for a number. | LOW | Already in requirements (`debate_rounds` table + expandable timeline UI). |
| Persisted history of past runs | Any tool used repeatedly over time needs a list view of past sessions — table stakes for literally every SaaS tool, doubly so for one whose core value is tracking accuracy *over time*. | LOW | Already in requirements. |
| Role/persona configurability without redeploy | Given Bona already committed to CRUD-editable roles from v1, this is now a stated requirement, not optional — and it's genuinely table stakes for a debate tool, since debate quality lives entirely in the system prompts. | MED | Already in requirements — CRUD screen for `debate_roles`. |
| Login/auth on private business data | Estimating data (real invoiced amounts, pricing strategy per role) is commercially sensitive even for a one-person shop. Any tool storing this needs to gate access. | LOW | Already decided (OTP email, Supabase, same pattern as rest of BONA ecosystem). |
| Fixed, small round count (2-4 agents, 1-2 adjustment rounds) | Multi-agent debate research consistently shows accuracy **plateaus after 2-3 rounds and 2-4 agents** — more rounds add cost and latency without improving the outcome, and can even hurt it by amplifying groupthink. This validates the spec's decision to *not* make round count configurable per-run. | LOW | Confirms the "Out of Scope: cantidad de rondas configurable" decision was correct, not just cost-driven — it's evidence-based. |

### Differentiators (Competitive Advantage)

Nothing in the commercial estimating-software space combines multi-model debate with a real-invoice accuracy feedback loop. These are where bonasterio actually stands out — for Bona personally, not a market, but "differentiator" here means "materially better than the obvious simpler alternative (asking ChatGPT once)."

| Feature | Value Proposition | Complexity | Notes |
|---------|-------------------|------------|-------|
| Retroactive real-invoice annotation + accuracy tracking | This is the single feature that turns bonasterio from "AI chat with extra steps" into a self-improving system. No commercial quoting tool tracks *per-debate, per-role* prediction vs. actual outcome at this granularity — most forecast-accuracy tooling exists in sales/BI contexts (MAPE dashboards), not AI-debate contexts. Already required in PROJECT.md; the research confirms this pattern (predicted-vs-actual + trendline) is the right shape to build toward, just point it at debates instead of sales deals. | LOW (data entry) → MED (analytics later) | v1 just needs the annotation field (already spec'd). Trendline/bias-per-role analytics is a natural v1.x add once ~10-20 real debates have real invoices attached — not enough data to be meaningful at launch. |
| Heterogeneous model ensemble (Gemini + multiple OpenRouter labs) | Research is explicit that **heterogeneous** multi-agent systems (different model families/training data) hold up better than **homogeneous** ones (same model resampled) — mixing genuinely different labs' blind spots is what makes disagreement meaningful instead of noise. Already the plan (Gemini + Llama/DeepSeek/etc via OpenRouter), and this decision is validated by the research, not just cost-driven. | LOW (already architected) | Worth stating explicitly as a design principle so future role additions preserve diversity rather than adding a 4th same-family model. |
| Per-role calibration bias over time ("this role tends to overestimate by 15%") | Once enough real-invoice data accumulates, this is the payoff of the accuracy-tracking table stakes feature above — it lets Bona learn which persona/model to trust more, or recalibrate a role's system prompt. This is a distinctly AI-debate-native feature; no commercial estimating tool does per-judge bias tracking because none of them run a debate. | MED | Depends on accuracy-tracking data existing first (see Feature Dependencies below). Not v1 — needs a data backlog. |
| Fully editable roles/prompts as a live pricing-strategy lever | Because AAIERIC's official labor-cost table is periodically revised (it's explicitly indexed for Argentine inflation, updated multiple times a year), a debate role's prompt is a place to inject "current AAIERIC baseline" context without redeploying code. This turns role editing from a nice-to-have into a recurring operational need given the local pricing environment. | LOW (mechanism already required) | Confirms the CRUD-roles requirement is right-sized — the win isn't "custom personas" in the abstract, it's "keep pricing context current without a deploy." |
| Zero permanent API cost via free-tier-only models | Not a user-facing feature, but a sustaining differentiator vs. any competing approach (subscribing to a paid estimating SaaS, or manually running paid Claude/GPT queries) — this is what makes running the tool indefinitely viable for a one-person business. | N/A (architecture) | Already decided; worth naming explicitly as "why this survives long-term" for the roadmap's phase-ordering rationale. |

### Anti-Features (Commonly Requested, Often Problematic)

Things that look like natural additions (because commercial estimating/quoting tools have them, or because "more AI capability" always sounds good) but are wrong for an internal, single-user, non-customer-facing debate tool.

| Feature | Why Requested | Why Problematic | Alternative |
|---------|---------------|------------------|-------------|
| Customer-facing quote/PDF export, e-signature, payment collection | Every commercial estimating tool (BuildOps, QuoteIQ, PataBid, STACK) bundles this because *their* users send quotes to clients. It feels like the "natural next step" after getting a number. | bonasterio's stated purpose is internal decision support — the price range informs Bona, who then manually communicates/quotes the customer however he already does. Building a quoting/invoicing layer duplicates whatever process he already uses and turns a research tool into a full business-ops product, doubling scope. | Keep output as a screen Bona reads and manually acts on. If a "share this result" need emerges later, a simple text/copy export is enough — not PDF branding, e-signature, or payment. |
| CRM / customer contact management | Commercial tools bundle this because they own the full sales pipeline. | bonasterio has no concept of "customer" in its data model — jobs are described by task/materials/zone/complexity, not by client. Adding a CRM layer is solving a problem bonasterio doesn't have and BONA's ecosystem doesn't currently need in this app. | If customer tracking is ever needed, it belongs in a separate BONA app (or gets bolted onto an existing one), not into the debate engine. |
| Automated blueprint/drawing takeoff (NLP on plans, quantity extraction from PDFs) | Commercial AI estimating tools increasingly market "upload a drawing, get a takeoff" as their headline AI feature. | High complexity (document parsing, computer vision/NLP on technical drawings), and nothing in the current workflow suggests Bona works from formal drawings for these jobs — he describes jobs in his own words. Building this would be the single most expensive feature in the whole app for a use case that doesn't exist yet. | Structured free-text input (already planned) is sufficient. Revisit only if a specific recurring job type actually requires reading a plan. |
| Configurable round count / "debate until convergence" per individual run | Feels like it gives more control, and some debate research explores adaptive stopping (agents debate until they agree instead of a fixed count). | Research shows accuracy plateaus after 2-3 rounds regardless — adaptive convergence loops add unpredictable latency and, more importantly, unpredictable API call volume against free-tier rate limits, which directly threatens the zero-cost constraint. Already correctly flagged Out of Scope in PROJECT.md. | Fixed round count via `debate_config`, adjustable centrally (not per-run) if it ever needs tuning. |
| WebSocket live updates | Feels more "real-time" and technically impressive than polling. | Already correctly identified as unnecessary in PROJECT.md — Supabase Realtime/polling comfortably covers a single user checking one debate at a time. Building WebSocket infra adds server complexity (connection state, reconnect handling) for a UX difference the single user won't perceive at this volume. | Supabase Realtime subscription or short-interval polling, as already decided. |
| Majority-vote / pure numeric averaging as the final verdict | Simple to implement, feels "objective" (just average the three numbers). | LLM-as-judge research is explicit that pure voting throws away the *reasoning* behind disagreement — a 2-vs-1 split where the outlier caught a real complication (e.g., "this zone needs conduit, not surface wiring") is more valuable than the number it produced. Averaging silently discards exactly the information this tool exists to surface. | Keep the arbiter role as a qualitative synthesizer that explains disagreement and produces a justified range — already the spec'd design, just don't regress toward "just average the three numbers" as a shortcut. |
| Multi-user permissions / team roles / granular access control | OTP-by-account login is already planned "in case someone else on the team uses it later," which can tempt building role-based permissions now. | No second user exists today, and permission systems are pure speculative complexity — YAGNI. OTP login is already sufficient scaffolding (per-account rows) if a second user is ever added; building permission tiers now is solving a problem that doesn't exist. | Ship with a single implicit "owner" role. If/when a second real user appears, add permissioning then, informed by actual need instead of guesswork. |
| Automatic AAIERIC category mapping | Sounds like an obvious accuracy improvement — auto-tag every debate with its official AAIERIC labor category. | Already correctly deferred in PROJECT.md: the schema field exists but auto-populating it requires either a classifier or careful prompt engineering against a table that itself gets revised periodically for inflation — real complexity for a "nice to have" analytics dimension, not core to getting a usable price range. | Leave the field manually fillable (or empty) for v1; revisit once there's a concrete need to slice accuracy stats by AAIERIC category. |
| Fine-tuning or training a custom judge/arbiter model | "Distilled small judges" is a real pattern in production LLM-judge systems at scale, and it sounds like the "proper" way to build a judge. | Massive overkill for a solo-user tool with a low debate volume — fine-tuning requires labeled data at a volume this tool won't produce for a long time (if ever), plus training infra that conflicts directly with the zero-cost, free-tier-only constraint. | A well-crafted system prompt for the Árbitro role, refined manually over time as accuracy data comes in, is the right-sized approach here. |

## Feature Dependencies

```
Structured job input (task/materials/zone/complexity)
    └──requires──> nothing (foundational)

Parallel initial round + adjustment round + arbiter
    └──requires──> Role CRUD schema (debate_roles) existing, even if hardcoded first
                       └──requires──> Supabase schema (debates, debate_roles, debate_rounds)

Live progress view
    └──requires──> Incremental per-round persistence (nodo_persistencia writes as it goes,
                    not all-at-once at the end)

Expandable timeline of past debates
    └──requires──> debate_rounds table populated with role_id, numero_ronda, contenido

Retroactive real-invoice annotation
    └──requires──> debates table with presupuesto_real_facturado field
                       └──enables──> Accuracy tracking / calibration analytics

Per-role calibration bias over time (differentiator)
    └──requires──> Retroactive real-invoice annotation
    └──requires──> Numeric extraction of presupuesto_propuesto_min/max from free-text model
                    responses (structured parsing, not just prose)
    └──requires──> Sufficient volume of annotated debates (rough threshold: 10-20+)
                       (NOT viable at launch — this is a v1.x/v2 feature, not v1)

Role/prompt CRUD from PWA
    └──enables──> Keeping AAIERIC pricing context current without redeploy
    └──conflicts with──> Hardcoded roles-in-code (spec explicitly sequences hardcoded-first,
                          then CRUD, as an implementation strategy — not a permanent state)

Configurable round count per debate (deferred)
    └──conflicts with──> Zero-cost/free-tier constraint (unbounded rounds risk rate limits)
```

### Dependency Notes

- **Per-role calibration bias requires retroactive annotation + volume:** this is the clearest reason to sequence it as v1.x rather than v1 — the feature literally cannot produce a meaningful signal until enough real debates have real invoices attached. Building the analytics UI before the data exists is wasted work; the roadmap should treat "annotate the Nth debate" as an implicit prerequisite milestone, not just a feature toggle.
- **Live progress view requires incremental persistence:** if `nodo_persistencia` only writes at the very end, the polling-based live view has nothing to show mid-run. This is already correctly sequenced in the spec (persistence happens "a medida que se generan," not batched) — worth calling out explicitly as a phase-ordering constraint since it's easy to accidentally build persistence as an afterthought.
- **Role CRUD conflicts with hardcoded-roles-first as a *sequencing* choice, not a real conflict:** the spec's own suggested order (hardcode roles to validate the debate flow, then wire up dynamic loading from `debate_roles`) is sound — it lets the harder graph logic get validated before adding the config layer on top. The roadmap should preserve this order rather than trying to build both simultaneously.
- **Configurable round count conflicts with the zero-cost constraint:** any feature that makes the *number* of model calls per debate unbounded or user-controlled at runtime is in tension with staying inside free-tier rate limits reliably — this is why it's correctly parked as Out of Scope rather than a natural v1.1.

## MVP Definition

### Launch With (v1)

Matches PROJECT.md's Active requirements almost exactly — validated against research as the right-sized core loop.

- [ ] Structured job description input (task type, materials, zone, complexity) — foundation for everything downstream
- [ ] Parallel initial-round proposals across 2-4 heterogeneous models/roles — core debate mechanic, validated as the accuracy sweet spot by research
- [ ] One adjustment round where each role sees the others' answers — where the "debate" actually happens
- [ ] Árbitro role producing final range + justification + explicit disagreement notes — the actual deliverable
- [ ] Live/streaming progress view while a debate runs — baseline UX expectation for anything >5s
- [ ] Expandable full timeline of past debates (who said what, which round) — the audit trail that makes this trustworthy
- [ ] History list of past debates — needed the moment there's a second debate
- [ ] Retroactive real-invoice annotation on a past debate — the feature that makes this a system, not a toy
- [ ] Role CRUD (create/edit/deactivate, edit system_prompt) from the PWA — already a committed requirement, and needed to keep pricing context current given AAIERIC's periodic revisions
- [ ] OTP email login — required before storing real business pricing data anywhere

### Add After Validation (v1.x)

- [ ] Per-role/per-model accuracy stats and bias tracking — trigger: once roughly 10-20 debates have real-invoice annotations, this becomes statistically meaningful rather than noise
- [ ] AAIERIC automatic category mapping/suggestion — trigger: once there's an actual need to slice accuracy analytics by AAIERIC category, not before
- [ ] Simple text/copy export of a debate result — trigger: if Bona finds himself manually retyping the arbiter's output somewhere else

### Future Consideration (v2+)

- [ ] Configurable round count per individual debate — defer: research shows fixed 2-3 rounds is already near-optimal; only revisit if a specific job type proves to need deeper deliberation
- [ ] Reuse the debate graph engine for other decision types (spec review, architecture decisions) — defer: explicitly out of scope for this milestone per PROJECT.md; the graph architecture already supports it later without rework
- [ ] Adaptive "debate until convergence" instead of fixed rounds — defer: conflicts with free-tier rate-limit safety; only worth revisiting if the cost/infra constraint changes

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---------|------------|----------------------|----------|
| Structured job input | HIGH | LOW | P1 |
| Parallel initial round | HIGH | MEDIUM | P1 |
| Adjustment round | HIGH | MEDIUM | P1 |
| Árbitro final verdict w/ disagreement notes | HIGH | MEDIUM | P1 |
| Live progress view | MEDIUM | LOW-MEDIUM | P1 |
| Expandable debate timeline | HIGH | LOW | P1 |
| Debate history list | MEDIUM | LOW | P1 |
| Retroactive real-invoice annotation | HIGH | LOW | P1 |
| Role CRUD from PWA | MEDIUM | MEDIUM | P1 |
| OTP login | MEDIUM | LOW | P1 |
| Per-role accuracy/bias tracking | HIGH (long-term) | MEDIUM | P2 |
| AAIERIC auto-category mapping | LOW-MEDIUM | MEDIUM | P3 |
| Configurable round count per debate | LOW | LOW-MEDIUM | P3 |
| Reusable debate engine for other decisions | LOW (now) / HIGH (later) | N/A (architectural, not a feature) | P3 |
| Customer-facing quote export/PDF/e-signature | LOW (out of scope) | HIGH | Anti-feature |
| CRM/customer management | LOW (out of scope) | HIGH | Anti-feature |
| Blueprint/drawing takeoff | LOW (no current need) | VERY HIGH | Anti-feature |

**Priority key:**
- P1: Must have for launch
- P2: Should have, add when possible
- P3: Nice to have, future consideration

## Competitor Feature Analysis

There is no direct competitor (a solo-user AI-debate estimation tool for a specific trade). The closest reference points are shown for contrast — mainly to justify *not* copying their feature sets wholesale.

| Feature | Commercial AI Estimating Tools (BuildOps, PataBid, QuoteIQ, STACK) | Generic Multi-Agent-Debate / LLM-Judge Research | bonasterio's Approach |
|---------|----------------------------------------------------------------------|--------------------------------------------------|------------------------|
| Input method | Blueprint/drawing upload + NLP parsing of specs | N/A (task-specific prompts) | Structured free-text description (task/materials/zone/complexity) — no drawing parsing |
| Number of "opinions" consulted | Single AI engine, pricebook-driven | 2-4 heterogeneous agents (plateau point per research) | 2-4 heterogeneous free-tier models/roles, matches research-validated sweet spot |
| Disagreement handling | None — single estimate output | Explicit: surfaced disagreement, ensemble judging, pairwise comparison | Árbitro explicitly narrates disagreement, not just a final number |
| Accuracy feedback loop | Time-tracking feeds back into future *labor hour* estimates (per-job, not per-model) | Human-judgment agreement rate against a holdout set (eval-time, not production) | Retroactive real-invoice annotation feeding per-role/per-model bias over time — closest existing pattern is sales-forecast MAPE dashboards, applied to AI debate instead |
| Customer-facing output | PDF quotes, e-signature, payment collection | N/A | None — internal-only, Bona reads and acts manually |
| Role/persona customization | None (fixed engine logic) | Role-play patterns exist in research (e.g., "Six Thinking Hats") but rarely user-editable in shipped products | Full CRUD on roles/prompts from the PWA, without redeploy |

## Sources

- [Multi-Agent Debate: Framework & Applications — Emergent Mind](https://www.emergentmind.com/topics/multi-agent-debate-approach) — MEDIUM-HIGH, aggregation of multiple arXiv papers
- [Multi-Agent Debate Strategies: Survey, Taxonomy, and Challenges (arXiv 2607.26212)](https://arxiv.org/html/2607.26212v1) — HIGH, peer-reviewed-track survey
- [ARMOR-MAD: Adaptive Routing for Heterogeneous Multi-Agent Debate (arXiv 2606.13197)](https://arxiv.org/pdf/2606.13197) — HIGH
- [CortexDebate: Debating Sparsely and Equally for Multi-Agent Debate (arXiv 2507.03928)](https://arxiv.org/pdf/2507.03928) — HIGH
- [Weak judges, strong panel: an ensemble approach to LLM eval — orq.ai](https://orq.ai/blog/llm-juries-in-practice) — MEDIUM, industry eval-tooling vendor, consistent with academic sources
- [LLM-as-a-Judge in 2026 — DeepEval](https://deepeval.com/blog/llm-as-a-judge) — MEDIUM, eval framework vendor blog
- [LLM-as-Judge Best Practices in 2026 — FutureAGI](https://futureagi.com/blog/llm-as-judge-best-practices-2026) — MEDIUM
- [Self-Consistency Is Losing Its Edge (arXiv 2511.00751)](https://arxiv.org/html/2511.00751) — HIGH, directly informs "fixed small round count" table-stakes finding
- [Rethinking Mixture-of-Agents: Is Mixing Different LLMs Beneficial? (arXiv 2502.00674)](https://arxiv.org/pdf/2502.00674) — HIGH, informs heterogeneous-vs-homogeneous differentiator
- [Understanding Agent Scaling in LLM-Based Multi-Agent Systems via Diversity (arXiv 2602.03794)](https://arxiv.org/html/2602.03794v1) — HIGH
- [12 Best AI Tools for Electrical Contractors and Techs — BuildOps](https://buildops.com/resources/electrical-ai-tools) — MEDIUM, vendor marketing content, used only for contrast/anti-feature justification
- [Inside the Engine: AI Electrical Estimating Software Features — PataBid](https://www.patabid.com/post/inside-the-engine-ai-electrical-estimating-software-features-that-drive-quantify) — MEDIUM, same caveat
- [Best AI Estimating Software For Electrical Contractors 2026 — MyQuoteIQ](https://myquoteiq.com/best-ai-estimating-software-electrical-contractors-2026/) — MEDIUM, same caveat
- [Sales Forecasting Accuracy Guide — Forecastio](https://forecastio.ai/blog/sales-forecasting-accuracy-and-analysis) — MEDIUM, analogous pattern (predicted vs. actual + trendline) applied to sales, mapped onto the accuracy-tracking differentiator
- [Forecast Accuracy — Count.co](https://count.co/metric/forecast-accuracy) — MEDIUM
- [AAIERIC — Costos Sugeridos de Mano de Obra](https://aaieric.org.ar/costos-mano-de-obra) — HIGH, official association source, confirms periodic inflation-indexed revisions relevant to the role-editing differentiator
- Project context: `/home/bona/bonasterio/.planning/PROJECT.md`, `/home/bona/bonasterio/bona-debate-ai-spec.md` — original spec and validated requirements

---
*Feature research for: Multi-AI debate/consensus budget estimation tool (bonasterio)*
*Researched: 2026-08-20*
