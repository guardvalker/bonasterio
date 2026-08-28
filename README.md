# bonasterio

**App:** todavía sin deploy — proyecto en etapa de planificación, sin código de la app escrito aún.

Herramienta interna de BONA que arma un "consejo de IAs" para presupuestar
trabajos eléctricos: varios roles de IA proponen un presupuesto, ven las
respuestas de los demás, ajustan en una ronda de debate, y un rol Árbitro da
la conclusión final (rango de precio, justificación, puntos de desacuerdo).
Cada debate queda guardado para comparar más adelante contra lo realmente
facturado.

Frontend PWA (mismo patrón sin build que el resto del ecosistema) + backend
en Google Cloud Run (LangGraph) + Supabase compartido (`bonapps`). Ver
`bona-debate-ai-spec.md` para el detalle completo.
