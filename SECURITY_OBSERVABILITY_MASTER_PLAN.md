# Observabilidad de seguridad — master plan
> **Status:** PENDING · 2026-11-18 · D1–D4 decididas; O1 publicado (`securityevent` v0.0.2); sigue O2 dentro del plan de inicio de sesión · el plan de inicio de sesión (ARCHITECTURE D75–D82, `auth/docs/PLAN.md`) espera O1 para que `auth` publique sus eventos una sola vez

Problema que se ve: si alguien intenta entrar a la fuerza, prueba RUT al azar o inyecta un script en el navegador, nadie se entera. Hoy:
- `auth` define 22 tipos de evento de seguridad (`auth/user.go:56-80`) y solo `auth/authority` los publica (`module.go:137`, tema `auth.security`); el único que escucha es `appkit` (`backend/signin.go:111`), que los escribe como una línea de texto. No se guardan, no se cuentan, no avisan a nadie.
- Las demás capas no informan: el control de permisos de `server/httpd` (rechazos 401/403), los rechazos de validación de `crud`, el tope de cuerpo de `router/security` (413).
- El registro es `func(...any)` en todas partes (`server/httpd/httpd.go:25`): texto sin campos.
- `notify` ya tiene alertas y canales (`notify/README.md`), pero nada de seguridad llega a él.
- En el navegador, las cabeceras de seguridad (`router/security`) son opcionales y no hay reporte de scripts bloqueados.

Cómo lo hacen otros (documentación oficial leída el 2026-11-18):
- [OWASP Logging Vocabulary](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Vocabulary_Cheat_Sheet.html): nombres estándar (`authn_login_fail`, `authn_login_fail_max`, `authn_login_lock`, `authz_fail`, `excess_rate_limit_exceeded`, `input_validation_fail`, `session_use_after_expire`…) y campos (`datetime`, `appid`, `event`, `level`, `description`, `source_ip`…). [OWASP Logging](https://cheatsheetseries.owasp.org/cheatsheets/Logging_Cheat_Sheet.html): los fallos repetidos de ingreso son el primer indicio de fuerza bruta; alertar de inmediato lo grave; nunca registrar contraseñas, tokens ni ids de sesión.
- [Spring Security](https://docs.spring.io/spring-security/reference/servlet/authentication/events.html): el framework publica un evento por cada ingreso exitoso o fallido (`AuthenticationFailureBadCredentialsEvent`…) y la app escucha con `@EventListener`.
- [OpenTelemetry](https://opentelemetry.io/docs/concepts/signals/logs/): las librerías emiten por una API común y la app elige a dónde se exporta (salida estándar, un colector o cualquier servicio).
- [MDN, CSP `report-to`](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/report-to): el navegador bloquea un script inyectado y envía el reporte (`csp-violation`) por POST a la ruta que el servidor declara en `Reporting-Endpoints`.

Reglas: **api-design** (puerta 4), **api-harness**, **zero-debt**, **console-logs**, **sign-in-and-permissions**.

## Preguntas abiertas

**P5 — ¿Quién bloquea la fuerza bruta: cada modo o el suscriptor común (D2)?** Hoy `trusted_ip` crea su propio limitador (`auth/trusted_ip/mode.go:61`, 3 intentos por IP cada 15 min) y `email_password` acepta otro; cada modo cuenta por su lado, así que alguien puede probar 3 veces en un modo y 3 en otro. El bloqueo tiene que ocurrir **antes** de atender el intento (en la misma petición); el suscriptor de D2 recibe el evento **después**.
```go
// A: un solo limitador para toda la app, en Ports; cada modo lo consulta antes de trabajar; el suscriptor solo cuenta para avisar
ports.Limiter = auth.NewIPLimiter(3, 15*60, 15*60)            // appkit, una vez
if err := p.Limiter.Check(ip); err != nil { /* RateLimited */ } // trusted_ip, email_password, passkey
securityevent.Listen(broker, alertOnRepeatedFailures)          // appkit: authn_login_fail_max → notify
// B: el suscriptor cuenta los LoginFailed y los modos le preguntan si la IP está bloqueada (el suscriptor pasa a ser un puerto)
// C: como hoy, cada modo con su limitador; el suscriptor solo avisa
```
¿Cuál evita el acoplamiento y es más mantenible? A: un solo contador por IP para todos los modos, el bloqueo sigue en la petición (síncrono) y el suscriptor solo observa y avisa; ningún modo depende del suscriptor. B hace que los modos dependan de quien escucha eventos (dos responsabilidades en el suscriptor). C deja que un atacante sume intentos cambiando de modo. **Recomiendo A.** Respuesta:

## Decisiones por implementar

Ninguna todavía.

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| O1 | `securityevent` (D1, D4) | El contrato: `Kind` (16 tipos con nombre OWASP y nivel), `Event`, `Reporter` (`New`, `Report`), `Listen`; nunca secretos | ✅ publicado `securityevent` v0.0.2 (errores traducidos, `gotest` 100%) |
| O2 | `auth` | Publicar con el contrato y borrar `auth.SecurityEventType`; va dentro del plan de inicio de sesión | 🟡 hecho en local (etapa 4b del plan), sin publicar; `appkit` escribe cada evento en el registro |
| O3 | `server`, `crud`, `router` | Cada capa informa lo suyo: permisos negados (401/403), validación rechazada, cuerpo demasiado grande, ruta de reportes CSP (D3) | 🔴 espera O1 |
| O3b | `sitec`, `dom`, `components`, `layout`, `server` | D3: nada en línea, firmas (hash) del script y la hoja de estilos en `integrity` y en la CSP, sprite y diccionario a archivos, `style=""` e `innerHTML` corregidos, CSP estricta siempre activa | 🔴 |
| O4 | `appkit`, `notify` | El suscriptor de D2: líneas con campos, conteo por ventana, bloqueo, alertas | 🔴 espera O1 |
| O5 | `app` (`webtyp dev`, MCP) | Ver los eventos de seguridad mientras se desarrolla | 🔴 espera O4 |
| O6 | `devskills` | Skill del dominio y su fila en `framework-map` | 🔴 |

## Bloqueos

- 2026-11-18 · O1 (el contrato) va primero; el plan de inicio de sesión lo espera, porque O2 va dentro de él.

## Relación con otras olas

- ARCHITECTURE: el plan de D75–D82 (`auth/docs/PLAN.md`) incluye O2; no se ejecuta antes de que P1 esté decidida, para no tocar `auth` dos veces.
- El registro sin campos (`func(...any)` en todo el framework) es un problema más amplio que la seguridad: se pregunta aparte cuando se cierre este master.
## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (P1-A) | **Un solo contrato de eventos de seguridad, en un repo nuevo y chico:** el vocabulario de OWASP (`authn_login_fail`, `authn_login_fail_max`, `authn_login_lock`, `authz_fail`, `excess_rate_limit_exceeded`, `input_validation_fail`, `malicious_csp_violation`…), los campos y `Publish` sobre `events.Publisher`; nunca lleva contraseñas, tokens ni ids de sesión. `auth` borra `SecurityEventType`; nombre del repo en la puerta de **api-design** (dos palabras) | repo nuevo, `auth`, `server`, `crud`, `router` |
| D2 (P2-A) | **Un suscriptor en `appkit`:** línea con campos por evento, conteo por IP y por usuario en ventanas (fuerza bruta, rastreo) con bloqueo, alertas por los canales de `notify`, y vista en `webtyp dev`/MCP; exportar a OpenTelemetry queda como una salida más, después | `appkit`, `notify`, `app` |
| D3 (P3-A, nivel bancario) | **Política de contenido estricta por defecto, sin nada en línea:** el HTML no lleva CSS, JS ni SVG dentro. `server/httpd` envía siempre una CSP estricta basada en hashes (como recomienda [web.dev](https://web.dev/articles/strict-csp) para páginas estáticas): `script-src 'sha384-…' 'strict-dynamic' 'wasm-unsafe-eval'`, `style-src 'self'`, `object-src 'none'`, `base-uri 'none'`, `frame-ancestors 'none'`, `require-trusted-types-for 'script'` ([Trusted Types](https://developer.mozilla.org/en-US/docs/Web/API/Trusted_Types_API)) y `report-to` hacia la ruta de reportes del framework. `sitec` calcula al compilar la firma (hash) del script y de la hoja de estilos y la escribe como `integrity` ([SRI](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity)) y en la CSP: si el archivo cambia un byte, el navegador no lo ejecuta. **El SVG no se puede firmar con SRI** (solo cubre `<script>` y `<link rel=stylesheet>`): sale del HTML a un archivo propio del mismo origen, con su hash en el nombre (`sprite.<hash>.svg`), usado con `<use href>`, sin estilos en línea; la CSP impide que corra código dentro de él. El diccionario de traducciones sale también a un archivo. Lo que hoy rompería esta política se corrige en el mismo cambio: `style=""` en `components/calendarslider/calendarslider.go:610`, `components/cellgrid/cellgrid.go:57,83`, `layout/login/login.go:117` (pasan a clases o a `element.style.prop`, que la CSP permite según [MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Security-Policy/style-src)); `innerHTML`/`insertAdjacentHTML` en `dom/dom_frontend.go:202,279,1028` (pasan a construir nodos, o a una sola política de Trusted Types en `dom`) | `server`, `router/security`, `sitec`, `dom`, `components`, `layout` |
| D4 (P4-A) | **El repo del contrato se llama `webtyp.com/securityevent`** (creado con `gonew` el 2026-11-18, v0.0.1, público, vacío): `securityevent.Event`, `securityevent.Publish`, `securityevent.LoginFailed`… | `securityevent` |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| O1 | `securityevent` (D1, D4) | El contrato: `Kind` (16 tipos con nombre OWASP y nivel), `Event`, `Reporter` (`New`, `Report`), `Listen`; nunca secretos | 🟡 hecho en local, `gotest` ✅ (98%); espera revisión del dueño para publicar |
| O2 | `auth` | Publicar con el contrato y borrar `auth.SecurityEventType`; va dentro del plan de inicio de sesión (`auth/docs/PLAN.md`), no aparte | 🔴 espera O1 |
| O3 | `server`, `crud`, `router` | Cada capa informa lo suyo: permisos negados (401/403), validación rechazada, cuerpo demasiado grande, ruta de reportes CSP (D3) | 🔴 espera O1 |
| O3b | `sitec`, `dom`, `components`, `layout`, `server` | D3: nada en línea, firmas (hash) del script y la hoja de estilos en `integrity` y en la CSP, sprite y diccionario a archivos, `style=""` e `innerHTML` corregidos, CSP estricta siempre activa | 🔴 |
| O4 | `appkit`, `notify` | El suscriptor de D2: líneas con campos, conteo por ventana, bloqueo, alertas | 🔴 espera O1 |
| O5 | `app` (`webtyp dev`, MCP) | Ver los eventos de seguridad mientras se desarrolla | 🔴 espera O4 |
| O6 | `devskills` | Skill del dominio y su fila en `framework-map` | 🔴 |

## Bloqueos

- 2026-11-18 · O1 (el contrato) va primero; el plan de inicio de sesión lo espera, porque O2 va dentro de él.

## Relación con otras olas

- ARCHITECTURE: el plan de D75–D82 (`auth/docs/PLAN.md`) incluye O2; no se ejecuta antes de que P1 esté decidida, para no tocar `auth` dos veces.
- El registro sin campos (`func(...any)` en todo el framework) es un problema más amplio que la seguridad: se pregunta aparte cuando se cierre este master.
