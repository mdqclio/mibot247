# Estado actual

_Última actualización: 2026-06-04_

## Bot web Puerto Delfín (n8n `puerto-delfin-bot-web`, id `iUhkwyAsDGCpTFQf`)

Operativo:
- Q&A con la KB (`sistema-de-reservas-d9e54` `/cabanas/knowledge_base`), LLM Claude Haiku 4.5.
- **Disponibilidad + precio multi-turno**: extrae intent/fechas/personas, calcula contra `precios`/`reservas`/`beds` reales, responde con template y números exactos.
- **Captura de nombre** del lead.
- **Memoria** vía historial de Firebase (`mensajes/{convId}`), no Simple Memory.
- **Device** persistido en la conversación.

Flujo (rama normal):
`Webhook → Prep → Write Conversacion (user) → Write Mensaje (user) → Read estado → IF Pausa → Read KB → Build System → Read History → Build Messages → LLM Haiku → Parse LLM → IF Calc → [calc: precios/reservas/beds → Calc Disponibilidad → Build Reply] / [normal] → Reply Final → Write Mensaje bot → Update Conversacion → IF Nombre → Respond`.

## Panel BotControl (`index.html`, repo mibot247)

Vivo: Conversaciones en tiempo real (`onValue`), hilo de mensajes, handoff (toggle de pausa + respuesta de operador + auto-pausa), device icon, sidebar responsive.

- **Pasada de diseño (restyle de Conversaciones + tokens compartidos): in-progress.** Aplicada y pusheada (commit `71c414f`), pendiente review visual y posible 2ª iteración de color/tipografía.

## Datos / infra

- `botcontrol-base` (RTDB): inbox (`clientes/puerto-delfin/conversaciones` + `/mensajes`). Conversaciones demo/test limpiadas (2026-06-04).
- `sistema-de-reservas-d9e54` (RTDB): fuente de verdad de KB, precios, reservas, beds.
- Credenciales n8n (httpQueryAuth): `DdPDjrIIDL6S9yjt` (botcontrol-base), `DuY7hTWmbx52S6ds` (sistema-de-reservas). Anthropic: `PbZibsBHvfFy0G3c`.

## Decisiones de diseño

- **Bot = espejo fiel del sistema de reservas.** Lee precio/capacidad/disponibilidad con la lógica real del sistema; data mal cargada (temporadas sin rangos → siempre precio base; capacidad UI vs dato) es problema del dueño y se autocorrige al arreglar la data.
- **Respuesta de disponibilidad por template**, no 2ª llamada al LLM: evita que el modelo manipule/invente los números.
- **Memoria = historial de Firebase** (últimos N mensajes), no Simple Memory ni slot-filling: fuente de verdad única, sobrevive reinicios.

## Pendiente antes de exponer a un huésped real

- Cierre de reserva (handoff ante "sí, avancemos").
- Cuentas de operador (Franco/Mari).
- Cerrar CORS de `*` al dominio real.
- Rotar secrets de Firebase que aparecieron en chat.
- Confirmar lado-widget de device detection.

_(Detalle en BACKLOG.md.)_
