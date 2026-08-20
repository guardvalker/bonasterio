# bonasterio

## What This Is

Herramienta interna de BONA que arma un "consejo de IAs" para generar presupuestos de trabajos eléctricos. En vez de consultar varios modelos por separado y comparar a mano, arma un debate estructurado entre varios roles/personalidades de IA: cada uno propone un presupuesto, ve las respuestas de los demás, tiene una ronda para ajustar o defender su número, y un rol Árbitro da la conclusión final (rango de precio, justificación, y dónde hubo desacuerdo). Todo el debate queda guardado para comparar después contra lo que Bona terminó facturando realmente.

## Core Value

Que el rango de presupuesto final que da el Árbitro sea confiable y quede registrado, para poder medir con el tiempo qué tan preciso es el consejo comparado contra lo realmente facturado.

## Requirements

### Validated

(None yet — ship to validate)

### Active

- [ ] Bona puede iniciar un debate describiendo un trabajo (tipo de tarea, materiales, zona, complejidad)
- [ ] El sistema le pide presupuesto a varios modelos distintos en paralelo, cada uno con un rol/personalidad propia
- [ ] Cada modelo ve las respuestas de los demás y tiene una ronda para ajustar o defender su número
- [ ] Un rol Árbitro da la conclusión final: rango de precio, justificación, y dónde hubo desacuerdo
- [ ] Bona puede ver el debate en vivo mientras corre (respuestas apareciendo ronda a ronda, sin esperar a que termine todo)
- [ ] Bona puede leer un timeline expandible de todas las rondas después (quién dijo qué, en qué ronda)
- [ ] Bona puede ver un historial de debates pasados
- [ ] Bona puede anotar retroactivamente el presupuesto real facturado en un debate pasado, para comparar precisión
- [ ] Bona puede crear/editar/desactivar roles (nombre, modelo, system_prompt) desde la PWA, sin tocar código
- [ ] Login por cuenta (OTP por email), mismo patrón que las otras apps del ecosistema BONA
- [ ] Cada debate corre sobre modelos gratuitos de labs/arquitecturas distintas (Gemini + varios vía OpenRouter), sin costo de API nunca
- [ ] El backend corre en Google Cloud Run (Docker), sin depender de que una máquina propia esté siempre prendida

### Out of Scope

- WebSocket para actualización en vivo — Supabase Realtime alcanza para el volumen de uso; se puede migrar después sin romper nada
- Mapeo automático a categorías AAIERIC — el campo existe en el schema, pero completarlo automáticamente (en vez de a mano) queda para más adelante
- Modelos de pago (Claude/GPT vía API real) — descartado por el requisito de costo cero permanente; se puede reconsiderar si el free tier alguna vez resulta insuficiente
- Cantidad de rondas configurable por corrida individual — arranca con una config fija, ajustarla por debate puntual es v2
- Reusar el motor de debate para otro tipo de decisión (specs técnicas, arquitectura de otra PWA) — la arquitectura en grafo lo permite a futuro, pero no se construye ningún otro caso de uso en este milestone

## Context

- Parte del ecosistema BONA — mismo Supabase compartido (proyecto `bonapps`) y mismo patrón visual (monospaced/copper/blueprint) que Toolbox y bonapp-gastos.
- El spec original (`bona-debate-ai-spec.md`) proponía Anthropic + OpenAI + Google de pago. Se reemplazó por Gemini (free tier oficial de Google) + OpenRouter (free tier de modelos open-source de labs distintos: Llama, DeepSeek, etc.) tras decidir en la fase de questioning que el costo debía ser cero permanente, no solo bajo.
- Se descartó explícitamente automatizar las cuentas free de claude.ai/chatgpt.com (scraping de la interfaz web) por violar los términos de servicio de ambos y ser una integración frágil — se usan únicamente APIs oficiales con free tier documentado.
- Se descartó correr el backend en una laptop dedicada propia (disponibilidad atada a la electricidad/internet de casa) a favor de Google Cloud Run.
- Repo ya creada y clonada localmente: `guardvalker/bonasterio` (vacía al momento de iniciar el proyecto).

## Constraints

- **Costo**: Cero costo de API permanente — todo modelo de IA usado debe tener free tier oficial documentado; nada de scraping de cuentas consumer.
- **Infraestructura backend**: Google Cloud Run (Docker) — no depende de hardware propio siempre encendido.
- **Base de datos**: Supabase compartido del proyecto `bonapps` (mismo que las demás apps BONA), no un proyecto separado.
- **Frontend**: PWA estática en GitHub Pages, mismo patrón que el resto del ecosistema.
- **Auth**: OTP por email vía Supabase, no magic link — lección ya aprendida en lista-super: magic link no funciona en PWAs instaladas en iOS (storage aislado del navegador que abre el mail).

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Gemini + varios modelos vía OpenRouter en vez de Claude/GPT/Gemini de pago | Requisito explícito de costo cero permanente; Anthropic/OpenAI no tienen free tier de API duradero (solo crédito de prueba único) | — Pending |
| Backend en Google Cloud Run en vez de laptop propia o VM | No depender de electricidad/internet de casa; cero mantenimiento de hardware; cuota gratuita mensual cubre de sobra este volumen de uso | — Pending |
| Roles editables (CRUD completo) desde v1, no hardcodeados primero | Ya se decidió login con cuentas — tiene sentido que el dueño gestione sus propios roles desde el día 1 en vez de depender de un redeploy de código | — Pending |
| Auth con cuentas (OTP por email) en vez de app sin login | Mismo patrón que el resto del ecosistema BONA, por si en el futuro otra persona del equipo la usa | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd:complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-08-20 after initialization*
