# Señal de errores y `go.mod` en `webtyp dev` — master plan
> **Status:** PENDING · 2026-11-19 · esperando respuesta del dueño a P1–P3

Problema que se ve en `webtyp dev` / MCP cuando hay un fallo de compilación o `go.mod` desfasado:
- El mismo error se imprime **dos veces**: primero en el handler que falló (`ASSETS` o `SERVER`) y de inmediato otra vez en `WATCH` (`InitialRegistration file error:` / `Watch updating file error:`).
- Un error repetido en el mismo archivo/símbolo imprime 8+ líneas idénticas (`undefined: injectComponentID` en 8 líneas de `dom_frontend.go`), contaminando el contexto de la TUI y del MCP en vez de agruparse como hace `wpath.CompactList` en `WATCH`.
- Cuando aparece `updates to go.mod needed; to update it: go mod tidy` o `missing go.sum entry`, el desarrollador tiene que descubrir en qué módulo falta y correr `go mod tidy` a mano.

Reglas: **console-logs**, **api-design**, **bug-fix-flow**, **zero-debt**.

## Preguntas abiertas

**P1 — Deduplicar el error entre el handler (`ASSETS`/`SERVER`) y `WATCH`.** Hoy `SSRFileWatcher.NewFileEvent` (`app/ssr_watcher.go:64`) y `Server` loguean el error en su propia sección **y además** devuelven el mismo `error` a `devwatch`, que vuelve a imprimirlo bajo `WATCH`. ¿Cuál evita el acoplamiento y es más mantenible?
```go
// Hoy: dos líneas idénticas en TUI/MCP (ASSETS + WATCH, o SERVER + WATCH)
// A) El handler que ya logueó el error no se re-imprime en WATCH (devwatch recibe un sentinel/tipo ErrAlreadyLogged o solo loguea si el handler no emitió log):
return devwatch.ErrReported // WATCH detiene el flujo sin repetir el texto
// B) Ningún handler loguea errores de NewFileEvent; solo devwatch imprime el error con el nombre del handler origen.
```
**Recomiendo A**: preserva la sección dueña (`ASSETS`, `SERVER`, `CLIENT`) donde el desarrollador busca el fallo y elimina el eco en `WATCH`. Respuesta:

**P2 — Compactar errores repetidos de compilación (como `wpath.CompactList`).** Hoy `gobuild` / `sitec` imprimen cada línea cruda de `go build` (`../../webtyp/dom/dom_frontend.go:181,268,485,491,496,530,536,541: undefined: injectComponentID` en 8 líneas).
```go
// Hoy (8 líneas):
// ../../webtyp/dom/dom_frontend.go:181:3: undefined: injectComponentID
// ../../webtyp/dom/dom_frontend.go:268:3: undefined: injectComponentID
// ...
// A) Agrupar por archivo + mensaje y acortar con ~:
// ~/Dev/Project/webtyp/dom/dom_frontend.go:181 (+7 líneas: 268,485,491,496,530,536,541): undefined: injectComponentID
// B) Mostrar solo el primer error por paquete + "(+N errores más; ver -debug)".
```
**Recomiendo A**: conserva todas las líneas exactas para el agente/dev en una sola línea legible y usa `~` (**console-logs**). Respuesta:

**P3 — ¿Corrección automática de `go mod tidy` cuando el compilador reporta `updates to go.mod needed` / `missing go.sum entry`?** Durante el desarrollo con `replace`, `go build` / `sitec` fallan con `go: updates to go.mod needed` o `missing go.sum entry` sin aclarar siempre en qué módulo ejecutar `go mod tidy`.
- **A)** Auto-reparar en `webtyp dev`: detectar el mensaje de `go.mod`/`go.sum`, ejecutar `go mod tidy` en el módulo afectado (con 1 reintento, nunca en bucle) y loguear una línea `GOMOD: ran go mod tidy in <dir>`.
- **B)** No modificar `go.mod`/`go.sum` automáticamente: solo resumir en 1 línea el comando exacto (`run: cd <dir> && go mod tidy`) y dejar que el desarrollador/agente lo ejecute.
**Recomiendo A** (con tope de 1 reintento por cambio de archivo): en desarrollo el objetivo es no interrumpir el flujo por un `go.sum` desfasado tras un `replace`. Respuesta:

## Decisiones por implementar

Ninguna aún (esperan P1–P3).

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| L1 | `devwatch`, `app` | P1: eliminar eco duplicado entre `ASSETS`/`SERVER` y `WATCH` | 🔴 espera P1 |
| L2 | `gobuild`, `sitec`, `app` | P2: compactar errores repetidos de compilación con `~` y líneas agrupadas | 🔴 espera P2 |
| L3 | `devflow`, `app` | P3: detección/reparación automática de `go.mod`/`go.sum` en `webtyp dev` | 🔴 espera P3 |

## Bloqueos

- 2026-11-19 · Esperando respuesta del dueño a P1–P3.

## Relación con otras olas

- Complementa `TUI_SIGNAL` (cerrado 2026-11-02) y la skill **console-logs**.
