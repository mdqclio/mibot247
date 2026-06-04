# Backlog

Pendientes y próximos pasos. `[next]` = lo siguiente a tomar.

## Bot / reservas

- **[next] Cierre de reserva (handoff).** El bot invita "¿avanzamos con la reserva?" pero no hay handler para el "sí". Ante intención de reservar → pausar el bot + avisar al operador (o pausar para que lo tome del inbox).
- **UX respuesta de disponibilidad.** Definir "desde $X" vs listar opciones por capacidad.
- **Device detection — lado widget.** Confirmar que `prompt-device-widget.md` se corrió en el CC de puertodelfin (que el widget mande `device` en el POST). La parte n8n + panel ya está; sin el widget, no llega `device`.

## Panel

- **Pasada de diseño (in-progress).** Restyle de Conversaciones + tokens aplicado (commit `71c414f`); pendiente review visual y posible 2ª iteración de color/tipografía.

## Operación / seguridad (antes de exponer a huésped real)

- **Cuentas de operador (Franco/Mari)** en Firebase.
- **Cerrar CORS** de `*` al dominio real de Puerto Delfín (webhook + header del Respond).
- **Rotar los secrets de Firebase** que aparecieron en chat (botcontrol-base, sistema-de-reservas).

## Hecho recientemente (no re-tomar)

- Disponibilidad + precio end-to-end (slices A/B/C), fechas relativas, memoria, fix de deflección, device (n8n+panel) — 2026-06-04.
- Limpieza de conversaciones demo/test de `botcontrol-base` — 2026-06-04.
