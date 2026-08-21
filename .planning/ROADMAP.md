# Roadmap: bonasterio

## Overview

bonasterio builds outward from the data foundation to the riskiest integration. Phase 1 locks in the shared-team Supabase schema, RLS policy, and Realtime plumbing — the security-critical decision every later phase depends on. Phase 2 builds every PWA screen that talks directly to Supabase with zero backend (auth, role CRUD, debate history/timeline, retroactive invoice annotation), giving Bona a usable tool early and validating the schema against a real logged-in session. Phase 3 isolates the hardest, most novel logic — the LangGraph debate engine itself (parallel proposals, adjustment round, arbiter verdict, incremental persistence) — as a standalone script run against real free-tier models, decoupled from any HTTP/deploy concerns. Phase 4 wraps that engine in FastAPI and deploys it to Cloud Run, proving it survives cold starts and CPU throttling with just one thin endpoint. Phase 5 ties every prior phase together: Bona starts a real debate from the PWA and watches it unfold live via Supabase Realtime, ending in the Árbitro's verdict — the full core value of the project, delivered last because it's the only piece touching everything else.

## Phases

**Phase Numbering:**
- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Schema, RLS & Realtime Foundation** - Shared-team Supabase schema with RLS and Realtime publication for debates/rounds/roles/config.
- [ ] **Phase 2: PWA — Auth, Roles & History** - Bona logs in, manages the AI role roster, and reviews/annotates past debates, entirely Supabase-direct with no backend.
- [ ] **Phase 3: Debate Engine (LangGraph)** - The parallel-proposal → adjustment-round → arbiter-verdict mechanic works end-to-end against real free-tier models, persisting incrementally.
- [ ] **Phase 4: Backend Deploy (FastAPI + Cloud Run)** - The debate engine is reachable as a secured HTTP endpoint that survives Cloud Run cold starts.
- [ ] **Phase 5: Live Debate Flow** - Bona starts a real debate from the PWA and watches every round arrive live, ending in the Árbitro's verdict.

## Phase Details

### Phase 1: Schema, RLS & Realtime Foundation
**Goal**: Supabase (shared `bonapps` project) has a secure, realtime-ready data foundation for debates, rounds, roles, and config, with the shared-team-data RLS model (no `owner_id` isolation) explicitly resolved and implemented.
**Mode:** mvp
**Depends on**: Nothing (first phase)
**Requirements**: AUTH-03
**Success Criteria** (what must be TRUE):
  1. Las 4 tablas (`debate_roles`, `debates`, `debate_rounds`, `debate_config`) existen con su esquema, foreign keys e índices correctos en el proyecto `bonapps` compartido.
  2. Cualquier cuenta autenticada puede leer y escribir todos los datos de debates/roles/config (visibilidad compartida entre cuentas, sin `owner_id`); pedidos anónimos son rechazados por RLS.
  3. Los inserts/updates en `debates` y `debate_rounds` se publican en la publicación `supabase_realtime`, verificable con una suscripción de prueba que recibe el evento.
**Plans**: TBD

### Phase 2: PWA — Auth, Roles & History
**Goal**: Bona can log in, manage the AI role roster, and review/annotate past debates entirely from the PWA, with zero backend dependency — validating Phase 1's schema/RLS against a real OTP-logged-in session.
**Mode:** mvp
**Depends on**: Phase 1
**Requirements**: AUTH-01, AUTH-02, ROLES-01, ROLES-02, HIST-01, HIST-02, HIST-03
**Success Criteria** (what must be TRUE):
  1. Bona puede loguearse con un código OTP enviado por email, y la sesión persiste entre refrescos del navegador.
  2. La app viene precargada con los 4 roles default del spec (Perito Conservador, Perito de Mercado, Perito AAIERIC, Árbitro), y Bona puede crear/editar/desactivar roles (nombre, modelo, system_prompt) desde la PWA sin tocar código.
  3. Bona puede ver una lista de debates pasados y abrir un timeline expandible de todas las rondas de cualquier debate (quién dijo qué, en qué ronda), verificable con datos sembrados manualmente ya que el motor de debate todavía no corre.
  4. Bona puede anotar retroactivamente el presupuesto real facturado en un debate pasado.
**Plans**: TBD
**UI hint**: yes

### Phase 3: Debate Engine (LangGraph)
**Goal**: The multi-role debate mechanic — parallel initial proposals, one adjustment round, arbiter verdict, incremental persistence — works correctly end-to-end when run as a standalone script against real free-tier models and real `debate_roles` data.
**Mode:** mvp
**Depends on**: Phase 1, Phase 2
**Requirements**: DEBATE-02, DEBATE-03, DEBATE-04, DEBATE-05
**Success Criteria** (what must be TRUE):
  1. Corriendo el grafo con un job de prueba, los 2-4 roles activos devuelven presupuesto inicial en paralelo, cada uno usando su propio modelo (Gemini u OpenRouter), sin ver las respuestas de los demás todavía.
  2. En la ronda de ajuste, cada rol ve las respuestas de los demás y devuelve si mantiene o modifica su número, con justificación.
  3. El rol Árbitro devuelve un veredicto final con rango de precio, justificación, y notas explícitas de dónde hubo desacuerdo (no un promedio numérico simple).
  4. Cada ronda se escribe en `debate_rounds` en Supabase a medida que el nodo del grafo termina, no todo al final — verificable viendo aparecer las filas durante la corrida del script.
**Plans**: TBD

### Phase 4: Backend Deploy (FastAPI + Cloud Run)
**Goal**: The debate engine from Phase 3 is reachable as a secured, deployed HTTP endpoint (`POST /debates`) that survives Cloud Run's cold-start and CPU-throttling model.
**Mode:** mvp
**Depends on**: Phase 3
**Requirements**: (none — infrastructure/deploy phase enabling Phase 5)
**Success Criteria** (what must be TRUE):
  1. `POST /debates`, desplegado en Cloud Run, verifica el JWT de Supabase entrante y rechaza requests sin token o con token inválido antes de llamar a ningún modelo.
  2. Una corrida completa de debate invocada por HTTP se completa y persiste todas sus rondas en Supabase incluso después de que la instancia estuvo fría (cold start), dentro del timeout configurado.
  3. Ningún secreto (service-role key, API keys de Gemini/OpenRouter) está horneado en la imagen Docker ni expuesto al cliente; se inyectan vía Secret Manager/variables de entorno en runtime.
**Plans**: TBD

### Phase 5: Live Debate Flow
**Goal**: Bona can kick off a real debate from the PWA describing a job and watch every round arrive live as it's generated, ending in the Árbitro's final verdict — the core value of the project, fully working end-to-end.
**Mode:** mvp
**Depends on**: Phase 2, Phase 4
**Requirements**: DEBATE-01, LIVE-01
**Success Criteria** (what must be TRUE):
  1. Bona puede iniciar un debate ingresando la descripción de un trabajo (tipo de tarea, materiales, zona, complejidad) desde la pantalla "nuevo debate".
  2. Al enviar, la PWA genera el `debate_id` en el cliente, abre su suscripción Realtime antes de disparar el POST, y las respuestas de cada ronda aparecen en pantalla a medida que se generan, sin esperar a que termine todo el debate.
  3. Cuando el backend termina, el veredicto final del Árbitro (rango de precio, justificación, desacuerdos) se muestra en pantalla vía el update de Realtime en `debates`, incluso si la request POST original del cliente todavía no resolvió.
**Plans**: TBD
**UI hint**: yes

## Progress

**Execution Order:**
Phases execute in numeric order: 1 → 2 → 3 → 4 → 5

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Schema, RLS & Realtime Foundation | 0/TBD | Not started | - |
| 2. PWA — Auth, Roles & History | 0/TBD | Not started | - |
| 3. Debate Engine (LangGraph) | 0/TBD | Not started | - |
| 4. Backend Deploy (FastAPI + Cloud Run) | 0/TBD | Not started | - |
| 5. Live Debate Flow | 0/TBD | Not started | - |
