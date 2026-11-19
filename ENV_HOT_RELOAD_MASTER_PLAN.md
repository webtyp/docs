# Recarga al cambiar `.env` — master plan
> **Status:** PENDING · 2026-11-19 · sin investigar a fondo; esperando que el dueño lo priorice (nació de DEV_AUTOLOGIN_MULTIUSER, cerrada)

**Qué se ve hoy:** en `mjosefa-cms`, guardar `.env` (p. ej. cambiar `DEV_AUTOLOGIN`) no provoca nada. El valor nuevo solo entra cuando el servidor se reinicia por otro motivo (un cambio de código, o de un módulo con `replace`), y las pestañas del navegador de DEV_AUTOLOGIN no se vuelven a abrir.

**Regla del dueño (2026-11-19):** el manejador interesado escucha el evento de cambio del `.env` y provoca el cambio que le corresponde; no un reinicio general a ciegas.

**Lo que se sabe (código):**
- El servidor lee `.env` en cada arranque: `gorun.EnvFile` (`server/strategies.go:376`, `gorun/envfile.go:16`). Nadie vigila el archivo.
- La base de datos lo lee aparte: `app/db_sync.go:28`; los almacenes del daemon: `app/daemon_stores.go:55`.
- Las pestañas de DEV_AUTOLOGIN se abren solo al abrir el navegador (`app/autologin_tabs.go`, `OpenOrNavigate`).

## Preguntas abiertas
Ninguna todavía: falta inventariar qué manejadores leen `.env` y qué debe hacer cada uno (reiniciar el servidor, reconectar la base, reabrir pestañas…). Ese inventario genera las preguntas.

## Decisiones por implementar
| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 | Un evento «cambió `.env`» (con las claves cambiadas); cada manejador que depende de una clave lo escucha y reacciona él mismo (regla del dueño) | por definir: `devwatch` / `app` / `server` |

## Fases
| Fase | Repo | Qué | Estado |
|---|---|---|---|
| F1 | — | Inventario de lectores de `.env` y su reacción esperada → preguntas | 🔴 |

## Bloqueos
- Ninguno.

## Relación con otras olas
- Skill `dev-autologin`: al cambiar `DEV_AUTOLOGIN` el servidor debe reiniciarse y `app` reabrir las pestañas (`app.OpenOrNavigate`).
- DOTENV_INTEGRITY: el mismo archivo; esa ola cuidó que no se corrompa.
