# Changelog

Formato: por fecha (más reciente arriba). Solo lo completado y verificado.

## 2026-06-04

### Bot web Puerto Delfín — workflow `puerto-delfin-bot-web` (id `iUhkwyAsDGCpTFQf`, n8n)

- **Disponibilidad + precio end-to-end.** Extracción del LLM extendida a `{reply, nombre, intent, fechaEntrada, fechaSalida, personas}`. Gate (`IF Calc`) que calcula solo con `intent=disponibilidad` + fechas + personas presentes. Lee `precios`/`reservas`/`beds` de `sistema-de-reservas-d9e54` (cred `DuY7hTWmbx52S6ds`); un Code node replica la lógica fiel del sistema de reservas (capacidad ≥ personas, cabaña libre en todo el rango, precio por noche × noches). La respuesta sale por **template** con los números del cálculo — el LLM no toca los números.
- **Resolución de fechas relativas.** `Prep` calcula anclajes de calendario (`diaSemanaHoy`, `proximoViernes/Sabado/Domingo`, `findeQueViene`) con luxon `setZone('America/Argentina/Buenos_Aires')`; el LLM usa los anclajes en vez de calcular el día de la semana. Regla de negocio: **"el finde" = check-in viernes, check-out domingo (2 noches)**.
- **Build System.** El system prompt se arma en un Code node (JS, template literal) en vez de dentro del expression del nodo LLM (arregla un `invalid syntax` por la concatenación pesada).
- **Memoria conversacional.** `Read History` (últimos 10 msgs de `botcontrol-base` `mensajes/{convId}`) + `Build Messages` → el LLM recibe el historial, no solo el mensaje actual. Habilita disponibilidad multi-turno.
- **Fix de prompt.** El bot ya no deflecta disponibilidad a un humano: pide los datos que faltan (fechas/personas) y nunca inventa precios. La derivación a humano queda solo para consultas fuera de la KB.
- **Device detection.** n8n persiste `device` en la conversación; el panel muestra ícono 📱 (mobile) / 💻 (desktop). _(Falta confirmar el lado del widget — ver BACKLOG.)_

### Panel BotControl (`index.html`)

- **Limpieza de conversaciones demo/test** en `botcontrol-base` (read-before-delete, con backup): se eliminaron las 5 demo viejas y las 36 de test; ninguna real (pre-launch). Inbox limpio.

> La pasada de diseño del panel (restyle de Conversaciones + tokens) se aplicó pero queda **in-progress** (pendiente review visual y posible 2ª iteración) — ver STATE/BACKLOG.

## 2026-06-03 (previo, base)

- Bot web: webhook → escritura en `botcontrol-base` (modelo inbox), Q&A con KB (Haiku 4.5), captura de nombre del lead.
- Panel: Conversaciones en tiempo real (`onValue`) + hilo de mensajes; handoff del operador (toggle de pausa `botPausado` + respuesta `rol=operador` + auto-pausa); sidebar responsive (drawer mobile); regla de Firebase para lectura pública por `convId`.
- Widget de chat drop-in (`puerto-delfin-widget-test.html`) + CORS en el webhook.
