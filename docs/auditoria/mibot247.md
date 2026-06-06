# Auditoría de producción — mibot247 / BotControl

- **Repo:** `mibot247` (`/home/clio/dev/mibot247`)
- **Fecha:** 2026-06-06
- **Stack:** Panel admin single-file (`index.html`, vanilla JS + Firebase Auth + Firebase RTDB `botcontrol-base`). Widget de chat vanilla (`puerto-delfin-widget-test.html`) que postea a un webhook n8n. Backend del chatbot en n8n (Claude Haiku 4.5, KB/precios/reservas en RTDB `sistema-de-reservas-d9e54`). Multi-tenant: todos los clientes bajo `/clientes/{clienteId}/*`.

**Leyenda:** 🟢 ok · 🟡 mejorable · 🔴 bloqueante

---

## 1) Front comprimido / sin secretos de cliente — 🟡

El panel es un único `index.html` sin minificar ni bundlear (~1800 líneas, CSS+JS inline; Firebase y fuentes vía CDN). No hay secretos de terceros embebidos: la `apiKey` de Firebase (línea 929) es pública por diseño y **no cuenta**. No aparecen tokens de Twilio, SendGrid ni claves de LLM en el cliente: las credenciales de Twilio (SID + Auth Token, líneas 800-801, 1324-1325) son inputs `type=password` que el admin carga y se guardan en RTDB — no hay secreto hardcodeado.

**Riesgo:** bajo en confidencialidad. Mejorable en performance/cache-busting (sin build, sin hashing de assets, sin compresión propia → depende 100% del hosting). El Twilio Auth Token guardándose en RTDB en texto plano se trata en el punto 2.

## 2) RLS / reglas de seguridad de la base — 🔴

**No hay reglas de RTDB en el repo** (`firebase.json`/`database.rules.json` ausentes). Una sola base `botcontrol-base` aloja a todos los clientes bajo `/clientes/*`, y el panel lee con `get(ref(db,'clientes'))` (líneas 996, 1673) y `onValue` sobre `/clientes/{id}/...`. Si las reglas live son el default `auth != null`, **cualquier usuario autenticado lee y escribe TODA la base de todos los tenants**: conversaciones, mensajes de huéspedes (PII), KB, config y el Twilio Auth Token de cada cliente.

**Riesgo:** crítico. Fuga cross-tenant de datos personales y de credenciales de Twilio (quema de saldo / suplantación de WhatsApp del cliente). Sin las reglas en el repo no hay forma de versionarlas ni auditarlas; el aislamiento por tenant hoy depende solo de que el front pida el path correcto, lo cual no es una frontera de seguridad.

## 3) Git sin secretos — 🟢

Escaneo de `git log -p --all` (13 commits): no aparecen claves de terceros, tokens ni passwords. Lo único que matchea son placeholders (`••••••••`), la `apiKey` pública de Firebase y notas en docs que dicen "rotar secrets que aparecieron en chat" (fuera del repo). El historial está limpio.

**Riesgo:** bajo. Pendiente operativo (fuera de git): rotar los secrets de Firebase que, según STATE/BACKLOG, se expusieron en un chat.

## 4) APIs: auth / permisos / validación — 🔴

- **Autorización = decorado de cliente.** Existe una página de "Roles" con toggles de permisos (admin/operador, líneas 1574-1603) que se guardan en `/clientes/{id}/roles`, pero **nada los aplica**: cualquier usuario logueado ejecuta `seedDemoData`, edita KB, cambia config, togglea pausa del bot y escribe mensajes. El login dice "Acceso restringido a administradores" pero no hay distinción de rol real; "Super Admin" está hardcodeado en el sidebar (línea 558).
- **Validación de entrada ausente** en escrituras a RTDB (KB, config, canales, mensajes de operador) — sin validación de tipo/longitud/formato.
- **XSS por innerHTML inconsistente.** Hay una función `esc()` (línea 1354) y se usa bien en `convHTML` y en el hilo de mensajes (`renderThread`). Pero el escape **no es consistente**: la lista de KB inyecta `item.pregunta`, `item.respuesta` y `item.tags` crudos en `innerHTML` **y** dentro de atributos `onclick` (líneas 1205-1209) — vector de inyección de JS; la tabla de clientes inyecta `c.nombre` y `c.color` sin escapar (líneas 1681-1682); `renderRoles` interpola `role` crudo en `onclick` (línea 1592). Dado que las conversaciones provienen de visitantes anónimos vía webhook, un nombre/mensaje malicioso podría llegar a un campo renderizado sin escapar.

**Riesgo:** alto. Los permisos no protegen nada (cualquier operador puede hacer todo en cualquier cliente, combinado con el punto 2). XSS almacenado factible en el panel del operador.

## 5) Hosting / entornos / variables de entorno — 🔴

No hay configuración de hosting ni de entornos en el repo (`firebase.json`, `.firebaserc`, `vercel.json`, `netlify.toml`, `.github/` ausentes). No hay separación dev/staging/prod: una sola base `botcontrol-base` con datos demo sembrados por el propio front (`seedDemoData`). La config de Firebase está hardcodeada (sin `.env`), sin pipeline de deploy ni CI. El widget apunta a un endpoint de producción fijo. STATE/BACKLOG mencionan CORS `*` pendiente de cerrar al dominio real.

**Riesgo:** alto. Sin entornos, los tests/seed contaminan la base de producción; sin CI/deploy reproducible y con CORS abierto, cualquier origen puede pegarle al webhook (ver punto 7).

## 6) Login / sesiones / vulnerabilidades — 🟡

Login vía Firebase `signInWithEmailAndPassword` (sesión y tokens gestionados por Firebase Auth, razonable). `onAuthStateChanged` togglea login/app. Puntos débiles: el manejo de error es genérico (`err.style.display='block'`, OK), pero **no hay verificación de email, ni MFA, ni gating por rol/allowlist** — cualquier cuenta creada en el proyecto Firebase entra al panel con plenos poderes. No hay timeout de sesión ni reautenticación para acciones sensibles. Sin las cuentas reales de operador todavía creadas (BACKLOG).

**Riesgo:** medio. La autenticación en sí es sólida; la **autorización** post-login es el agujero (se cruza con puntos 2 y 4).

## 7) Rate limiting — 🔴

El widget (`puerto-delfin-widget-test.html`) postea `{idExterno, texto}` al webhook n8n **sin autenticación, sin token, sin captcha y sin rate limit**. El `idExterno` es un `crypto.randomUUID()` guardado en localStorage — **totalmente forjable** por el cliente. No hay throttle/debounce en el front ni evidencia de límite en el backend. Cada POST dispara una llamada al LLM (Claude Haiku).

**Riesgo:** crítico. Un atacante puede automatizar miles de POSTs con `idExterno` nuevos → **quema de saldo del LLM**, inflado de RTDB y posible DoS económico. Con CORS `*` (punto 5), explotable desde cualquier origen.

## 8) Caché — 🔴 / 🟡

No hay estrategia de caché propia: sin headers de cache-control configurables (no hay hosting config), sin Service Worker, sin versionado de assets. En datos, cada navegación re-ejecuta `get()` completos (`renderClientsPage`, `loadKB`, `loadConfig`, etc.) sin memoización; los listeners `onValue` traen el nodo entero en cada cambio. Varias métricas del dashboard están hardcodeadas (`'89%'`, `'47'`, línea 1138-1140) — placeholders, no caché real.

**Riesgo:** medio. No bloquea seguridad pero impacta costo de lectura de RTDB y latencia a escala (se solapa con el punto 9). Marco 🟡 por impacto; 🔴 sólo si se cuenta la ausencia total de capa de caché.

## 9) Escalabilidad — 🔴

`onValue` se suscribe a **nodos completos sin filtrar ni paginar**: `/clientes/{id}/conversaciones` (línea 1403) y `/clientes/{id}/mensajes/{convId}` (línea 1450) descargan y re-renderizan todo el conjunto en cada cambio (`container.innerHTML = ...map(...)`). No hay `orderByChild`/`limitToLast`/paginación. `loadClients` baja `/clientes` entero. Con cientos de conversaciones o miles de mensajes por hilo esto crece linealmente en ancho de banda, memoria y costo por cada update, y re-renderiza el DOM completo.

**Riesgo:** alto a escala. Funciona en demo (un cliente, pocas convs) pero no soporta crecimiento; combinado con el punto 2, un `onValue` mal apuntado podría además traer datos de otros tenants.

## 10) Monitoreo / alertas — 🔴

No hay instrumentación: sin Sentry/Datadog/LogRocket, sin analytics, sin logging estructurado. Los errores se tragan en `console.error` (errores de `onValue`, fallos de `fetch` del widget) — invisibles en producción. No hay alertas de saldo del LLM, de tasa de error del webhook, ni de picos de tráfico. El "log" de RTDB (`/clientes/{id}/log`) es un audit-trail de acciones del admin, no monitoreo operativo.

**Riesgo:** alto. Un abuso del webhook (punto 7) o una caída del bot pasarían desapercibidos hasta ver la factura del LLM o un reclamo del cliente.

---

## Tabla resumen

| # | Punto | Estado |
|---|-------|--------|
| 1 | Front comprimido / sin secretos cliente | 🟡 |
| 2 | RLS / reglas de la base | 🔴 |
| 3 | Git sin secretos | 🟢 |
| 4 | APIs: auth / permisos / validación | 🔴 |
| 5 | Hosting / entornos / env vars | 🔴 |
| 6 | Login / sesiones / vulns | 🟡 |
| 7 | Rate limiting | 🔴 |
| 8 | Caché | 🟡 |
| 9 | Escalabilidad | 🔴 |
| 10 | Monitoreo / alertas | 🔴 |

**Conteo:** 1 🟢 · 2 🟡 · 6 🔴 (contando caché como 🟡 por impacto).

---

## Los 3 arreglos más urgentes

1. **Escribir y versionar reglas de RTDB con aislamiento por tenant (punto 2).** Hoy, con el default `auth != null`, cualquier usuario logueado lee/escribe toda la base: PII de huéspedes y el Twilio Auth Token de cada cliente. Definir `database.rules.json` que restrinja cada path `/clientes/{id}/*` al/los usuario(s) de ese tenant (claims o nodo de membresía) y validar tipos/longitudes en las escrituras. Sin esto, los "permisos" del punto 4 son cosméticos.

2. **Cerrar el webhook del widget: auth + rate limit + CORS al dominio real (punto 7).** El `idExterno` es forjable y no hay límite → quema directa del saldo del LLM y DoS económico desde cualquier origen. Añadir un token de sitio/firma, rate limiting por IP/sesión y captcha en abuso, cerrar CORS `*` al dominio de Puerto Delfín, y alertar sobre picos.

3. **Hacer cumplir autorización por rol y escapar todo innerHTML de forma consistente (punto 4).** Aplicar realmente los permisos admin/operador (gating por rol antes de cada acción, idealmente respaldado por las reglas del punto 1) y enrutar todo dato dinámico por `esc()` — hoy la lista de KB, la tabla de clientes y los `onclick` de roles inyectan datos crudos en `innerHTML`/atributos, habilitando XSS almacenado que llega desde visitantes anónimos del webhook.
