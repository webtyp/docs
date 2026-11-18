# Observabilidad de seguridad — master plan
> **Status:** PENDING · 2026-11-18 · espera P1–P3 del dueño · el plan de inicio de sesión (ARCHITECTURE D75–D82, `auth/docs/PLAN.md`) espera P1 para que `auth` publique sus eventos una sola vez, con el contrato de aquí

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

**P1 — Un solo contrato de eventos de seguridad para todas las capas: ¿dónde vive?** Hoy el tipo está en `auth`, así que `server/httpd` o `crud` no pueden publicar sin importar `auth` (una pieza de abajo dependería de una de arriba).
```go
// Hoy, solo auth:   m.events.Publish(events.Event{Topic: auth.TopicSecurity, Payload: &auth.SecurityEvent{Type: auth.EventUnknownRUT, IP: ip}})
// A: repo nuevo y chico, solo el vocabulario (nombres de OWASP) y el tema; cada capa publica por el events.Publisher que ya recibe
securityevent.Publish(pub, securityevent.Event{Kind: securityevent.LoginFailed, IP: ip, Mode: "trusted_ip"})   // auth
securityevent.Publish(pub, securityevent.Event{Kind: securityevent.AccessDenied, UserID: uid, Resource: res})   // server/httpd
// B: el vocabulario dentro de `events` (junto al mecanismo de publicar y suscribir)
// C: queda en `auth` y las demás capas importan `auth`
```
¿Cuál evita el acoplamiento y es más mantenible? A: una sola lista de nombres para todo el framework, cada capa depende solo de algo más bajo que ella, y `auth` borra su `SecurityEventType`. B mezcla dos responsabilidades en `events` (cómo se publica y qué significa un ataque). C hace que `server` y `crud` dependan de `auth`. **Recomiendo A** (nombre por decidir en la puerta de **api-design**; dos palabras). Respuesta:

**P2 — Qué se hace con los eventos: ver, detectar y avisar.**
A) `appkit` conecta un solo suscriptor que (1) escribe cada evento como línea con campos (OWASP: hora, app, evento, nivel, IP, usuario, sin secretos), (2) cuenta por IP y por usuario en ventanas de tiempo para detectar fuerza bruta o rastreo (por ejemplo 10 ingresos fallidos en 15 minutos → `authn_login_fail_max` y bloqueo), (3) envía lo grave por los canales de `notify` que la app configuró, y (4) el entorno de desarrollo (`webtyp dev`, MCP) los muestra. Exportar a OpenTelemetry queda como una salida más, después.
B) Solo exportar a OpenTelemetry y dejar la detección a un sistema externo.
C) Solo escribir líneas en el registro.
¿Cuál evita el acoplamiento y es más mantenible? A: funciona sin infraestructura extra (una clínica no tiene un colector), reutiliza `notify` y el contrato de P1 deja abierta la exportación. B exige montar otro sistema. C registra pero nadie se entera a tiempo. **Recomiendo A.** Respuesta:

**P3 — El navegador: qué se puede detectar de verdad.** Ningún navegador puede saber si un programa malicioso en el equipo captura lo que se escribe. Lo que sí se puede: que las llaves no se puedan robar (passkey: la llave privada nunca sale del celular y solo funciona en su dominio), bloquear scripts inyectados con una política de contenido (CSP) estricta, y recibir el reporte cuando el navegador bloquea uno.
A) `server/httpd` pone por defecto las cabeceras de `router/security` con una CSP estricta (con `'wasm-unsafe-eval'`, que necesita el WASM) y `report-to` hacia una ruta que monta el framework; cada reporte se publica como evento de seguridad (P1). B) Dejar las cabeceras opcionales, como hoy (solo las monta `iam`).
¿Cuál evita el acoplamiento y es más mantenible? A: cerrado por defecto (**api-harness** regla 7) y un solo lugar que lo hace para todas las apps. B deja cada app sin protección hasta que alguien se acuerde. **Recomiendo A.** Respuesta:

## Decisiones por implementar

Ninguna todavía.

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| O1 | repo nuevo (P1) | El contrato: vocabulario OWASP, campos, `Publish`; lo que no se registra (contraseñas, tokens, ids de sesión) | 🔴 espera P1 |
| O2 | `auth` | Publicar con el contrato y borrar `auth.SecurityEventType`; va dentro del plan de inicio de sesión (`auth/docs/PLAN.md`), no aparte | 🔴 espera O1 |
| O3 | `server`, `crud`, `router` | Cada capa informa lo suyo: permisos negados (401/403), validación rechazada, cuerpo demasiado grande, ruta de reportes CSP (P3) | 🔴 espera P1, P3 |
| O4 | `appkit`, `notify` | El suscriptor de P2: líneas con campos, conteo por ventana, bloqueo, alertas | 🔴 espera P2 |
| O5 | `app` (`webtyp dev`, MCP) | Ver los eventos de seguridad mientras se desarrolla | 🔴 espera O4 |
| O6 | `devskills` | Skill del dominio y su fila en `framework-map` | 🔴 |

## Bloqueos

- 2026-11-18 · Todo espera P1–P3. El plan de inicio de sesión espera P1 (O2 va dentro de él).

## Relación con otras olas

- ARCHITECTURE: el plan de D75–D82 (`auth/docs/PLAN.md`) incluye O2; no se ejecuta antes de que P1 esté decidida, para no tocar `auth` dos veces.
- El registro sin campos (`func(...any)` en todo el framework) es un problema más amplio que la seguridad: se pregunta aparte cuando se cierre este master.
