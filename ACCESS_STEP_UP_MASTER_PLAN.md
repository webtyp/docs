# «Usuarios y roles» exige segundo factor — master plan
> **Status:** IN PROGRESS · 2026-11-19 · F1 (desarrollo) publicada en `auth` v0.0.125+; siguen F2 (passkey en mjosefa, falta diseño) y F3 (aviso en pantalla)

**Para retomar:** skills `sign-in-and-permissions`, `api-design`, `ui-consistency`, `zero-debt`. Prueba en `veltylabs/mjosefa-cms`.

**El problema:** `webtyp/access/definition.go:22,42,58,76` exige segundo factor en sus 4 entidades (persona, rol, asignación, auditoría), lectura incluida; `server/httpd/adapter.go:487-491` responde 401 con `WWW-Authenticate: … insufficient_user_authentication` a una sesión de un factor. `mjosefa-cms/config/app.go:36` solo habilita `trustedip.Mode` (un factor), así que **en producción nadie puede abrir «Usuarios y roles»**, y la pantalla muestra «0 / 0» sin avisar por qué.

**Publicado (`auth` v0.0.125+):** con `-tags dev`, DEV_AUTOLOGIN abre la sesión normal **y** la de segundo factor, y se la agrega a una sesión de un factor en un host con credencial viva (`auth/authority/devlogin.go`, `middleware.go`; tests `TestDevAutologin_OpensBothFactors`, `…_UpgradesAOneFactorSession`, `…_HandLoginOnASignedOutHostKeepsOneFactor` en `auth/tests/dev_autologin_tabs_test.go`). Verificado en vivo: `GET /api/access/access_person` → 200 en `user3.localhost`.

## Preguntas abiertas
Ninguna por ahora. Al empezar F2 hay que leer `webtyp/auth/passkey` (`mode.go`, `setup.go:178`) y escribir aquí las preguntas de diseño: cómo se registra la passkey de cada persona y en qué momento se pide (al entrar o al abrir una sección que lo exige).

## Decisiones por implementar
| # | Decisión | Dónde se implementa |
|---|---|---|
| D2 (P1) | El segundo factor en `access` es **obligatorio y sin opción para apagarlo** (dueño: mjosefa-cms está en internet; la seguridad no puede depender de una configuración que se olvide). `mjosefa-cms` agrega la passkey como segundo factor (`webtyp/auth/passkey` ya emite la sesión de dos factores: `passkey/setup.go:178`) | `mjosefa-cms/config/app.go` + `webtyp/auth/passkey` |
| D3 (P2) | Desarrollo: **A** publicada (ver arriba); falta **B**: autenticador virtual de Chrome (CDP `WebAuthn`) en `devbrowser`, para probar la pantalla de passkey de verdad cuando exista (D2) | A: `auth/authority/devlogin.go` · B: `devbrowser` |
| D1 | Una pantalla que recibe 401 con `router.ChallengeMFARequired` lo dice («Esta sección pide un segundo factor») en vez de mostrar una lista vacía. Silencio = defecto (`zero-debt`) | cliente de listas (ubicar en F3: `crudview`/`layout`) |

## Fases
| Fase | Repo | Qué | Estado |
|---|---|---|---|
| F2 | veltylabs/mjosefa-cms + webtyp/auth/passkey | D2: preguntas de diseño (ver arriba), luego habilitar la passkey y su pantalla | 🔴 |
| F3 | cliente de listas | D1: ubicar el repo que hace los `GET` de las listas y mostrar el aviso | 🔴 |
| F4 | webtyp/devbrowser | D3-B, tras F2 | 🔴 |

Cerrado: F1 auth (D3-A) · auth v0.0.125.

## Bloqueos
- Ninguno.

## Relación con otras olas
- DEV_AUTOLOGIN_MULTIUSER (cerrada): allí se descubrió; el comportamiento final está en la skill `dev-autologin`.
- ACCESS_REUSE: creó `webtyp/access`; su promesa «MFA configurable» (línea 41) queda descartada por D2.
