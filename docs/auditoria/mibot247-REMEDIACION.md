# Remediación — mibot247 / BotControl

- **Repo:** `mibot247` (`/home/clio/dev/mibot247`)
- **Fecha:** 2026-06-06
- **Base:** auditoría en `docs/auditoria/mibot247.md`
- **Alcance de esta tanda:** solo arreglos seguros y reversibles **en código** del repo. **No** hubo despliegues, **no** se reescribió historial, **no** se hizo force-push.

**Leyenda:** ✅ hecho en código · 📄 documentado / plantilla lista (requiere acción de deploy o decisión) · ⏳ pendiente, dueño del producto

---

## ✅ Hecho en esta tanda (código)

### 1. Reglas de RTDB versionadas — `database.rules.json`
- Se creó `database.rules.json` con **deny-by-default** en la raíz (`".read": false`, `".write": false`).
- **Tier 1 (aplicado):** `/clientes` es legible y `/clientes/{clienteId}` escribible **solo** con `auth != null`. Esto cierra el agujero de "default `auth != null` global / lectura abierta" y deja una frontera explícita y auditable en el repo. **Sigue siendo cross-tenant entre usuarios autenticados** (ver Tier 2).
- **Tier 2 (documentado, NO aplicado):** aislamiento por tenant. Hoy **no existe** en el código un mapeo confiable `uid → cliente`: el panel no lee ni escribe ningún nodo `/usuarios/*`, "Super Admin" está hardcodeado en el sidebar y los roles (`/clientes/{id}/roles`) son decorativos. Por eso no se puede expresar el aislamiento por tenant con confianza todavía. Se dejó un placeholder `/usuarios/{uid}` (read solo del propio uid) como gancho para el Tier 2.

  **Para activar Tier 2 (dueño), elegí una vía:**
  - **a) Custom claims** (recomendado): al crear cada operador, setear `clienteId` (y opcional `rol`) como custom claim vía Admin SDK. Luego en las reglas:
    ```json
    "clientes": {
      "$clienteId": {
        ".read": "auth != null && auth.token.clienteId === $clienteId",
        ".write": "auth != null && auth.token.clienteId === $clienteId"
      }
    }
    ```
    El panel deja de hacer `get(ref(db,'clientes'))` global y pasa a leer solo su `clienteId` (del claim). Super Admin = claim aparte (`auth.token.superAdmin === true`) con regla `|| auth.token.superAdmin === true`.
  - **b) Nodo de membresía** `/usuarios/{uid}/cliente = "<clienteId>"` (escrito solo por Admin SDK/backend) y reglas con `root.child('usuarios').child(auth.uid).child('cliente').val() === $clienteId`. Más simple de poblar sin Admin SDK, pero requiere proteger `/usuarios` de escritura del cliente (ya está: `".write": false`).
- También conviene añadir **validación de tipo/longitud** (`.validate`) en KB, config, canales y mensajes de operador — pendiente, ver ⏳.

### 2. Wiring de Firebase CLI
- `firebase.json`: apunta `database.rules` → `database.rules.json`, `storage.rules` → `storage.rules`, y deja `hosting.public = "."` con `ignore` de archivos de config/docs (para que un futuro `firebase deploy --only hosting` no suba reglas ni `.md`).
- `.firebaserc`: `default` → `botcontrol-base` (projectId del `firebaseConfig` embebido en `index.html`).

### 3. Reglas de Storage — `storage.rules`
- Creado auth-only (`request.auth != null`) deny-by-default. Mismo Tier 2 pendiente que RTDB (comentado en el archivo). Aplica solo si se usa el bucket `botcontrol-base.firebasestorage.app`; si no se usa, no molesta.

### 4. XSS — `esc()` consistente en todos los sinks de `innerHTML` con datos dinámicos
Se aplicó `esc()` y se **eliminaron los `onclick` con strings interpolados** (vector de inyección de JS en el atributo), reemplazándolos por patrón `data-*` + `addEventListener`:
- **`renderClientSidebar`**: `c.color`, `c.nombre` → `esc()`.
- **`loadKB`** (vector principal, dato de RTDB editable): `item.pregunta`, `item.tags`, `item.tipo` → `esc()`; clase `kb-ic` sanitizada a un set fijo (`faq|doc|url`); botones editar/eliminar pasan de `onclick="editKB('...id...','...pregunta cruda...')"` a `data-id` + listener (mira el item por id, sin meter el texto crudo en el atributo).
- **`renderClientsPage`**: `c.nombre`, `c.color`, `c.config.bot_nombre`, `c.plan` → `esc()`; botones Editar/Desactivar pasan a `data-id` + listener.
- **`loadRoles`**: `roleNames[role]||role` y `p.label` → `esc()`; toggle de permisos pasa de `onclick="toggleRole('${role}','${p.key}',this)"` a `data-role`/`data-key` + listener.
- **`loadLog`**: `l.accion`, `l.quien` → `esc()` (el log lo escribe el panel, pero `accion` incorpora texto de KB que viene de cualquier origen).
- **`loadDashUnans`**: `u.text` → `esc()` y botón a `data-text` + listener (datos hoy hardcodeados; queda blindado para cuando se conecte a datos reales).
- **`sendSandbox`**: input del admin → `esc()` antes de `innerHTML +=`.

Quedan sin escapar solo interpolaciones de **arrays hardcodeados** en el propio código (`perms`, `topics`, barras de actividad) — no son datos externos.

**Verificación de sintaxis:** se extrajo el `<script type="module">`, se quitaron los `import` de URL (Node no resuelve specifiers `https://` en `--check`) y se corrió `node --check` → **SYNTAX OK**. JSON de reglas/config validado con `JSON.parse`.

### 5. `.gitignore`
- Creado (no existía): `node_modules/`, artefactos de Firebase (`.firebase/`, `firebase-debug*.log`, `.runtimeconfig.json`), `.env*` (con `!.env.example`), basura de editor/SO y logs.

---

## ⏳ Pendiente — dueño del producto (fuera del alcance "código seguro")

### A. Desplegar las reglas
Las reglas están en el repo pero **no surten efecto hasta desplegarlas**. La base sigue con las reglas live actuales hasta entonces.
```bash
firebase login
firebase deploy --only database          # publica database.rules.json
firebase deploy --only storage            # si se usa Storage
```
Antes de desplegar Tier 2, **poblar el mapeo uid→cliente** (claims o `/usuarios`) o todos los operadores pierden acceso.

### B. Cerrar el webhook del widget en n8n (auth + rate limit + CORS) ⏳ **externo (n8n)**
El widget (`puerto-delfin-widget-test.html`) postea `{idExterno, texto}` a un webhook n8n **sin auth, sin token, sin rate limit, con CORS `*`**, y cada POST dispara una llamada al LLM. **Vive en n8n, fuera de este repo.** Acciones (en el flow de n8n / proxy delante):
1. **Auth:** token de sitio firmado (HMAC) o header secreto que el widget incluya; rechazar lo que no lo traiga. El `idExterno` (`crypto.randomUUID()` en localStorage) es **forjable** y no sirve como identidad.
2. **Rate limiting:** por IP y por `idExterno` (p. ej. N msg/min, burst corto), con captcha/challenge ante abuso.
3. **CORS:** cerrar de `*` al dominio real de Puerto Delfín (`Access-Control-Allow-Origin: https://puertodelfin.com`).
4. **Alertas:** pico de POSTs / gasto del LLM (ver D).

### C. Autorización por rol de verdad (panel)
Los toggles de Roles no aplican nada: cualquier usuario logueado hace todo en cualquier cliente. Hace falta **gating real por rol** en el panel **respaldado por las reglas del Tier 2** (claims). Hasta entonces, los permisos son cosméticos. (Cambio de comportamiento → no se tocó en esta tanda de "arreglos seguros").

### D. Monitoreo / alertas
Sin Sentry/Datadog/logging estructurado; errores se tragan en `console.error`. Sumar: captura de errores del panel, tasa de error del webhook, **alerta de saldo/uso del LLM**, picos de tráfico.

### E. Rate limiting general
Además del webhook (B): throttle/debounce en el front del widget y límites en escrituras del panel.

### F. Separar entornos (dev/staging/prod)
Una sola base `botcontrol-base` con datos demo sembrados por el propio front (`seedDemoData`). Crear proyectos/instancias separadas; que los tests/seed no toquen prod. Wirear `.firebaserc` con targets `staging`/`prod`.

### G. Operativo (de la auditoría, fuera de git)
Rotar los secrets de Firebase que, según STATE/BACKLOG, se expusieron en un chat. (La `apiKey` de Firebase es pública por diseño; esto refiere a otros secretos del proyecto.)

---

## Resumen

| Ítem | Estado |
|---|---|
| `database.rules.json` deny-by-default + auth (Tier 1) | ✅ |
| Aislamiento por tenant (Tier 2) | 📄 documentado, requiere claims/membresía + deploy |
| `firebase.json` + `.firebaserc` | ✅ |
| `storage.rules` auth-only | ✅ |
| `esc()` consistente + onclick → dataset (XSS) | ✅ |
| `.gitignore` | ✅ |
| Desplegar reglas | ⏳ dueño |
| Webhook n8n (auth + rate limit + CORS) | ⏳ dueño (externo) |
| Autorización por rol real | ⏳ dueño |
| Monitoreo / alertas | ⏳ dueño |
| Rate limiting | ⏳ dueño |
| Separar entornos | ⏳ dueño |
