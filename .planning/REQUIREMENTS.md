# Requirements: bonasterio

**Defined:** 2026-08-20
**Core Value:** Que el rango de presupuesto final que da el Árbitro sea confiable y quede registrado, para poder medir con el tiempo qué tan preciso es el consejo comparado contra lo realmente facturado.

## v1 Requirements

### Debate (motor del consejo)

- [ ] **DEBATE-01**: Bona puede iniciar un debate ingresando la descripción de un trabajo (tipo de tarea, materiales, zona, complejidad)
- [ ] **DEBATE-02**: El sistema le pide presupuesto en paralelo a los roles activos (2-4 modelos heterogéneos: Gemini + varios vía OpenRouter)
- [ ] **DEBATE-03**: Cada rol ve las respuestas de los demás en una ronda de ajuste y puede mantener o modificar su número con justificación
- [ ] **DEBATE-04**: El rol Árbitro da la conclusión final — rango de precio, justificación, y notas explícitas de dónde hubo desacuerdo (no un promedio numérico simple)
- [ ] **DEBATE-05**: Cada ronda se guarda en la base de datos a medida que se genera, no todo al final

### Progreso en vivo

- [ ] **LIVE-01**: Bona ve las respuestas de cada ronda aparecer en vivo mientras el debate corre, sin esperar a que termine todo (vía Supabase Realtime)

### Historial

- [ ] **HIST-01**: Bona puede ver un timeline expandible de todas las rondas de un debate (quién dijo qué, en qué ronda)
- [ ] **HIST-02**: Bona puede ver una lista de debates pasados
- [ ] **HIST-03**: Bona puede anotar retroactivamente el presupuesto real facturado en un debate pasado, para comparar precisión

### Roles

- [ ] **ROLES-01**: Bona puede crear/editar/desactivar roles (nombre, modelo, system_prompt) desde la PWA sin tocar código
- [ ] **ROLES-02**: La app viene con los 4 roles sugeridos del spec precargados como default (Perito Conservador, Perito de Mercado, Perito AAIERIC, Árbitro)

### Autenticación

- [ ] **AUTH-01**: Bona puede loguearse con un código OTP enviado por email
- [ ] **AUTH-02**: La sesión persiste entre refrescos del navegador
- [ ] **AUTH-03**: Todas las cuentas logueadas comparten el mismo historial de debates y roles configurados (sin aislamiento por cuenta)

## v2 Requirements

Reconocidos pero fuera del roadmap actual — necesitan datos o validación que todavía no existen.

### Analítica

- **ANALYTICS-01**: Estadísticas de precisión por rol/modelo (quién tiende a sobre/subestimar) — necesita ~10-20 debates anotados con factura real para ser significativo
- **ANALYTICS-02**: Mapeo automático a categoría AAIERIC — sin necesidad concreta todavía

### Conveniencia

- **CONV-01**: Export simple de texto/copiar del resultado de un debate — solo si Bona termina retipeando el resultado en otro lado

## Out of Scope

Excluido explícitamente. Documentado para prevenir scope creep.

| Feature | Reason |
|---------|--------|
| Cotizaciones al cliente (PDF, firma electrónica, cobro) | Herramienta interna de decisión, no un producto de cotización — Bona ya tiene su propio proceso para comunicarle el precio al cliente |
| CRM / gestión de clientes | bonasterio no tiene concepto de "cliente" en su modelo de datos; pertenecería a otra app del ecosistema si hiciera falta |
| Toma de medidas automática desde planos (NLP/CV sobre PDFs) | Complejidad altísima para un caso de uso que no existe hoy — Bona describe los trabajos en texto |
| Cantidad de rondas configurable por debate individual | Conflictúa con el límite de costo cero — llamadas ilimitadas contra free tiers rate-limited |
| "Debatir hasta converger" (rondas adaptativas) | Mismo motivo — volumen de llamados impredecible |
| WebSocket para actualización en vivo | Supabase Realtime alcanza para este volumen de uso |
| Reusar el motor de debate para otro tipo de decisión (specs técnicas, arquitectura) | La arquitectura en grafo lo permite después sin rework, pero no se construye ningún otro caso de uso en este milestone |
| Fine-tuning de un modelo Árbitro propio | Sobre-ingeniería para el volumen de debates que va a tener esta herramienta |
| Permisos multi-usuario / roles de equipo | No existe un segundo usuario real todavía — YAGNI; el login OTP ya deja la puerta abierta si hace falta después |

## Traceability

Completado durante la creación del roadmap (5 fases, ver ROADMAP.md).

| Requirement | Phase | Status |
|-------------|-------|--------|
| DEBATE-01 | Phase 5 | Pending |
| DEBATE-02 | Phase 3 | Pending |
| DEBATE-03 | Phase 3 | Pending |
| DEBATE-04 | Phase 3 | Pending |
| DEBATE-05 | Phase 3 | Pending |
| LIVE-01 | Phase 5 | Pending |
| HIST-01 | Phase 2 | Pending |
| HIST-02 | Phase 2 | Pending |
| HIST-03 | Phase 2 | Pending |
| ROLES-01 | Phase 2 | Pending |
| ROLES-02 | Phase 2 | Pending |
| AUTH-01 | Phase 2 | Pending |
| AUTH-02 | Phase 2 | Pending |
| AUTH-03 | Phase 1 | Pending |

**Coverage:**
- v1 requirements: 14 total
- Mapped a fases: 14 (100%)
- Sin mapear: 0 ✓

---
*Requirements defined: 2026-08-20*
*Last updated: 2026-08-20 after roadmap creation*
