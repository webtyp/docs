# Señal de errores, TUI compacta y `logs.log` crítico — master plan
> **Status:** IN PROGRESS · 2026-11-19 · D1–D3 cerradas (P1–P3); esperando respuesta del dueño a P4–P7

Problema que se ve:
- En `webtyp dev`, los errores se duplican (`ASSETS`/`SERVER` + `WATCH`), se repiten por línea (`undefined: X` × 8) y `go.mod`/`go.sum` desfasado detiene el hot-reload (D1–D3).
- Manejadores de hot-reload (`SERVER`, `CLIENT`, `BROWSER`, `ASSETS`) acumulan líneas por cada guardado en vez de actualizar una sola línea con `OperationID` + `tui.LogOpen`/`tui.LogClose` (`devtui/example/HandlerWriterTracker.go:15-16`, `tui/interfaces.go:170-173`).
- Al cerrar la app, los eventos críticos de seguridad/alertas (`appkit/backend/security_watch.go:105`) se pierden si no hay canal configurado; falta persistir solo lo crítico en `logs.log` con ventana corta de fechas.

Reglas: **console-logs**, **api-design**, **bug-fix-flow**, **zero-debt**, **isomorphic-libraries**.

## Preguntas abiertas

**P4 — Una línea fija por manejador vs una línea por ciclo de hot-reload (`SERVER`, `CLIENT`, `BROWSER`, `ASSETS`).** Hoy `SERVER` escribe 4+ líneas por cada guardado (`Go file modified`, `Restarting`, `Stopping`, `Starting`). Con `GetLastOperationID`/`SetLastOperationID` + `tui.LogOpen`/`tui.LogClose` se animan los pasos en la misma línea:
```go
// A) Un OperationID fijo por manejador (ej. "server-status"): siempre 1 sola línea viva en la TUI
h.SetLastOperationID("server-status")
h.Logger(tui.LogOpen, "Restarting server...")     // anima spinner en la misma línea
h.Logger(tui.LogClose, "Running on :8080")        // reemplaza la línea y detiene animación
// B) Un OperationID nuevo por cada guardado ("server-reload-1", "server-reload-2"...): 1 línea por ciclo (crece con cada guardado)
```
**Recomiendo A**: la terminal nunca crece de forma desproporcionada; el historial completo sigue en el ring de MCP (`app_get_logs`). Respuesta:

**P5 — ¿Los errores una vez solucionados deben desaparecer de la TUI?**
- **A)** **Sí:** al compilar/arrancar bien el siguiente ciclo (`tui.LogClose` con `✅`), la línea del manejador vuelve a estado OK y el error resuelto sale de la TUI (lo crítico queda en `logs.log` y el historial reciente en `app_get_logs`).
- **B)** **No:** conservar el error resuelto en la TUI colapsado a 1 línea (`✓ resuelto: <resumen>`) hasta reiniciar la sesión.
**Recomiendo A**: la TUI muestra el estado vivo del proyecto (como un tablero de control); un error ya arreglado en pantalla solo agrega ruido. Respuesta:

**P6 — Dónde y cuándo escribir alertas críticas en `logs.log` (raíz del proyecto).** Hoy `appkit/backend/security_watch.go:105` publica `notify.Alert` en `notify.TopicAlert`, y `app/logs.go:63` solo escribe `logs.log` con `-debug`. ¿Cuál evita el acoplamiento y es más mantenible?
```go
// A) Suscriptor en appkit/backend (build !wasm) sobre notify.TopicAlert: escribe SIEMPRE lo crítico en <root>/logs.log (dev y prod self-host), haya o no email/WhatsApp configurado:
// 2026-11-19T11:15:00Z ERROR security.malicious_content_blocked resource=https://evil/x.js IP=::1
// B) Canal fallback en webtyp/notify (ChannelFile): solo escribe en logs.log si la app NO tiene otro canal configurado.
```
**Recomiendo A**: incluso con email/WhatsApp configurado, `logs.log` en disco sirve de respaldo si el canal externo falla o en desarrollo sin credenciales; con build tag `!wasm` no entra al binario WASM/Worker. Respuesta:

**P7 — Retención y filtro de `logs.log` (rango corto de fechas, solo lo crítico).**
- **A)** **Solo `Warning`/`Error` críticos** (alertas `notify.Alert` y fallos fatales de arranque/seguridad, nunca `Info`/debug), **retención de 7 días** con poda automática al abrir/rotar y tope duro de 1 MB, en **1 línea plana con fecha ISO-8601 y campos `clave=valor`** (legible directo en VS Code y fácil de parsear).
- **B)** Igual filtro pero **3 días** de retención en **JSONL** (1 objeto JSON por línea).
**Recomiendo A** (7 días, tope 1 MB, línea plana `clave=valor`): suficiente para revisar qué pasó la semana en curso sin que el archivo crezca ni pierda legibilidad humana en VS Code. Respuesta:

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (P1-A) | **Sin eco duplicado en `WATCH`:** cuando un handler (`ASSETS`, `SERVER`, `CLIENT`) ya logueó su error, devuelve `devwatch.ErrReported` y `WATCH` no repite el texto | `devwatch`, `app` |
| D2 (P2-A) | **Compactar errores repetidos de compilación:** agrupar por archivo + mensaje con ruta `~` (`~/Dev/…/file.go:181 (+7 líneas: …): msg`) | `gobuild`, `sitec`, `app` |
| D3 (P3-A) | **Auto `go mod tidy` en `webtyp dev`:** al detectar `updates to go.mod needed` / `missing go.sum entry`, ejecutar `go mod tidy` en el módulo afectado (máx. 1 reintento por cambio) y loguear `GOMOD: ran go mod tidy in <dir>` | `devflow`, `app` |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| L1 | `devwatch`, `app` | D1: eliminar eco duplicado entre `ASSETS`/`SERVER` y `WATCH` | 🔴 listo para planificar (espera P4–P5 para tocar `app` en una sola pasada) |
| L2 | `gobuild`, `sitec`, `app` | D2: compactar errores repetidos de compilación con `~` y líneas agrupadas | 🔴 listo para planificar |
| L3 | `devflow`, `app` | D3: detección y `go mod tidy` automático (1 reintento) en `webtyp dev` | 🔴 listo para planificar |
| L4 | `server`, `gobuild`, `devbrowser`, `app` | P4–P5: `OperationID` + `tui.LogOpen`/`tui.LogClose` en manejadores de hot-reload | 🔴 espera P4–P5 |
| L5 | `appkit` (o `notify`) | P6–P7: persistencia crítica en `logs.log` con poda por rango corto de fechas | 🔴 espera P6–P7 |

## Bloqueos

- 2026-11-19 · Esperando respuesta del dueño a P4–P7.

## Relación con otras olas

- Complementa `TUI_SIGNAL` (cerrado), `SECURITY_OBSERVABILITY` (cerrado 2026-11-19) y la skill **console-logs**.
