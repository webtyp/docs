# Logs de `webtyp dev` sin ruido repetido — master plan
> **Status:** IN PROGRESS · 2026-11-20 · F1a (TUI) en local con `replace`, esperando que el dueño lo vea en vivo; F1b (anillo MCP) y D1 sin empezar

**Para retomar:** skills `console-logs`, `webtyp-app`, `testing`, `codejob` («gopush and local `replace`»). Probar en `~/Dev/Project/veltylabs/mjosefa-cms`: guardar varias veces un `.go` del servidor y mirar `BUILD/SERVER` en la TUI y en `app_get_logs`.

**Causa del ruido que ya se arregló en local (devtui, modo cliente):** el frame de la animación `[...` reescribía «la última línea del manejador», no la suya. La salida de la app que llegaba durante un reinicio («Starting server on port 8080») era pisada por el siguiente frame, y cada reinicio dejaba una copia congelada de «Restarting server... .». Además una línea idéntica solo se juntaba si era la última del manejador.

**Causas que siguen abiertas:**
1. Doble línea del build: `app/section-build.go:458` y `app/client_file_event.go:36` (`w.LogSuccessState()` sin texto, `client_change.go:203-206`).
2. El anillo del daemon (`app/sse_publisher.go` `AddToRing`) agrega cada entrada sin juntar nada: el MCP sigue viendo cada «Starting server…» y cada «Restarting…».

## Preguntas abiertas
Ninguna.

## Decisiones por implementar
| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 | Un build imprime **una** línea: se borra la segunda (`client_file_event.go:36`; la de `section-build.go:458` ya lleva el tamaño) | `webtyp/app` |
| D2 (P1) | Una línea idéntica de un mismo manejador reutiliza su línea donde esté: pasa al final con la hora nueva, sin contador (dueño 2026-11-20: «si el mensaje final es el mismo debería seguir usando la misma línea»). Igual en TUI y MCP | `devtui` (hecho en local), `app/sse_publisher.go` (F1b) |
| D3 (P2) | El estado de SERVER es una sola línea viva (`OperationID` `server-status`): «Restarting server…» se anima y termina en «Server running on port N», que se reutiliza en cada reinicio; la animación solo reescribe la línea que abrió `[...`, nunca otra | `devtui` + `server` (hecho en local) |

## Fases
| Fase | Repo | Qué | Estado |
|---|---|---|---|
| F1a | devtui, server | D2 y D3 en la TUI. **devtui** (sin commit): `tabSection.go` (animación por `Id` de línea: `lineAnimation`, `closeAnimation`, `setLineContent`), `operation_accumulate.go` (`accumulateIndex`), `print.go` (`sendMessageWithHandler` devuelve la línea, `notifyContent`), `handlerRegistration.go`, `sse_client.go` (pasa `OperationID`), `client_mode_test.go` (test `TestSSEClient_AnimationKeepsLinesEmittedMeanwhile`). **server** (sin commit, lo dejó Antigravity): `strategies.go` vuelve a poner `OperationID` antes de cada `...]`. **app/go.mod**: `replace webtyp.com/devtui => ../devtui` y `replace webtyp.com/server => ../server`; `~/go/bin/webtyp` instalado desde ese código. Verificado: `go test -race` de devtui ✅; `gotest` style ❌ es previo (funciones sin usar en `field.go` y tests) | 🔴 espera que el dueño lo vea en vivo; luego `gopush` en devtui y server (la cascada quita los `replace`) |
| F1b | app | D2 en el anillo del MCP (`sse_publisher.go` `AddToRing`: misma regla que `accumulateIndex`) y D1, con tests que fallan primero | 🔴 app tiene trabajo sin commit de otro agente: `app_migrate.go`, `client_file_event.go`, `section-build.go` |
| F2 | devskills | `console-logs`: D2 y D3 (la animación reescribe solo su línea; una línea idéntica se reutiliza donde esté) | 🔴 tras F1a |

## Bloqueos
- Ninguno.

## Relación con otras olas
- ENV_HOT_RELOAD: cada reinicio es también cuando el servidor relee `.env`.
