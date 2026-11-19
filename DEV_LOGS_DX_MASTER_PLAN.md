# Señal de errores, TUI compacta y `logs.log` crítico — master plan
> **Status:** IN PROGRESS · 2026-11-19 · L1–L5 implementadas y verificadas en local (`gotest` 100% verde en los 8 repos; binario local `webtyp` instalado); esperando luz verde para publicar en cascada

Problema que se ve:
- En `webtyp dev`, los errores se duplican (`ASSETS`/`SERVER` + `WATCH`), se repiten por línea (`undefined: X` × 8) y `go.mod`/`go.sum` desfasado detiene el hot-reload (D1–D3).
- Los manejadores de `BUILD` acumulan líneas por cada guardado en vez de mantener 1 línea viva con `OperationID` + `tui.LogOpen`/`tui.LogClose` (D4–D5).
- Al cerrar la app, las alertas críticas de seguridad (`appkit/backend/security_watch.go:105`) se pierden si no hay canal configurado; falta persistir solo lo crítico en `logs.log` con poda de 7 días / 1 MB (D6–D7).

Reglas: **api-design**, **api-harness**, **console-logs**, **bug-fix-flow**, **zero-debt**, **isomorphic-libraries**.

## Preguntas abiertas

Ninguna.

## Puerta `api-design` / `api-harness` (Step 0 + 5 respuestas)

- **Step 0 (búsqueda en ecosistema):**
  - `devwatch/devwatch.go:24-39` ya usa interfaz estructural (`UnsupportedEventError` + `IsUnsupportedEvent`) para no acoplar handlers a `devwatch` → **extender** con `AlreadyLoggedError`.
  - `webtyp/filepath` (`Tilde`, `CompactList`) es formato de rutas puro; `webtyp/gobuild` es el único dueño de compilación Go → **extender** `gobuild` con `CompactErrors` (reusado por `app` para `SSR`, sin duplicar ni agregar dep a `sitec`).
  - `devtui` y `HeadlessTUI` ya implementan `GetLastOperationID`/`SetLastOperationID` + `tui.LogOpen`/`tui.LogClose` → **reusar** el contrato único existente (**0 formas nuevas**).
  - `webtyp/notify` posee `Alert`/`TopicAlert` (sin deps de I/O); `appkit` solo cablea (`framework-map`) → **extender** `notify` (`!wasm`, solo stdlib `os`) con `CriticalFileLog` y cablearlo en `appkit/backend` en 2 líneas.
- **1. Prior art:** Go `errors.Is`/interfaces estructurales ([Go errors](https://pkg.go.dev/errors)), Vite/esbuild compact single-line terminal status ([Vite CLI](https://vite.dev/guide/cli.html)), Go `slog` / RFC 5424 structured key=value log lines ([Go log/slog](https://pkg.go.dev/log/slog)), `lumberjack` bounded size/age rotation ([lumberjack](https://pkg.go.dev/gopkg.in/natefinch/lumberjack.v2)).
- **2. Novice-name test:** `devwatch.AlreadyLoggedError` (`AlreadyLogged() bool`), `gobuild.CompactErrors(output string) string`, `notify.NewCriticalFileLog(path string)` + `WriteAlert(notify.Alert) error`. Todos verb+noun / 2–3 palabras literales, sin booleanos ni abreviaturas.
- **3. Complexity ledger:** conceptos nuevos `+0` (reusa `OperationID`, `LogOpen/LogClose`, `notify.Listen`), líneas en consumidores `-N` (menos ruido en TUI/MCP), **formas de hacer lo mismo `+0 / -0`** (no se agrega mecanismo paralelo en `HeadlessTUI`).
- **4. Dónde pertenece (SRP):** supresión de eco → `devwatch`; compactación de errores Go → `gobuild`; `go mod tidy` → `devflow`; estado de 1 línea → cada handler existente (`server`, `gobuild`, `devbrowser`, `app`); persistencia crítica `logs.log` → `notify` (`!wasm`), cableado en `appkit/backend` (`!wasm`).
- **5. Qué borra:** borra el eco `"Watch updating file error:"`/`"InitialRegistration file error:"` cuando el handler ya logueó, borra líneas intermedias sueltas en `server` (`Go file modified`, `Restarting`, `Stopping`), y elimina la dependencia de `-debug` para no perder alertas críticas.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (P1-A) | **Sin eco duplicado en `WATCH`:** interfaz estructural `AlreadyLoggedError { AlreadyLogged() bool }` + `IsAlreadyLogged(err)` en `devwatch`; handlers que ya loguean devuelven error marcado y `devwatch` omite reload sin repetir texto | `devwatch`, `server`, `app` |
| D2 (P2-A) | **Compactar errores repetidos de compilación:** `gobuild.CompactErrors(output string) string` agrupa por archivo + mensaje con `wpath.Tilde` (`~/Dev/…/file.go:181 (+7 líneas: …): msg`), usado en `gobuild` y `app/ssr_watcher.go` | `gobuild`, `app` |
| D3 (P3-A) | **Auto `go mod tidy` en `webtyp dev`:** `devflow` detecta `updates to go.mod needed` / `missing go.sum entry`, corre `go mod tidy` en el módulo afectado (máx. 1 reintento por cambio) y loguea `GOMOD: ran go mod tidy in <dir>` | `devflow`, `app` |
| D4 (P4-A) | **1 línea viva para todos los manejadores de `BUILD`** (salvo `SECURITY`): usan el contrato existente `GetLastOperationID`/`SetLastOperationID` + `tui.LogOpen`/`tui.LogClose` (sin mecanismo paralelo) | `server`, `gobuild`, `devbrowser`, `app` |
| D5 (P5-A) | **Errores resueltos desaparecen de la TUI:** al cerrar en verde (`tui.LogClose`), la línea única del manejador reemplaza el error; lo crítico persiste en `logs.log` y el historial sigue en `app_get_logs` | `server`, `gobuild`, `app` |
| D6 (P6-A) | **Persistencia crítica en `notify` (`!wasm`) cableada en `appkit/backend`:** `notify.NewCriticalFileLog("logs.log")` suscrito a `notify.TopicAlert` siempre (dev y prod self-host), haya o no email/WhatsApp | `notify`, `appkit` |
| D7 (P7-A) | **Retención y formato de `logs.log`:** solo `Warning`/`Error` críticos, poda automática a últimos **7 días** y tope duro de **1 MB**, 1 línea plana ISO-8601 con campos `clave=valor` | `notify` |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| L1 | `devwatch` | D1: `AlreadyLoggedError` + `IsAlreadyLogged` + tests | 🔵 ✅ local (`gotest` ✅ 86.1%, sin publicar) |
| L2 | `gobuild` | D2: `CompactErrors` (agrupado por archivo+mensaje con `~`) + tests | 🔵 ✅ local (`gotest` ✅ 82.8%, sin publicar) |
| L3 | `devflow` | D3: `NeedsGoModTidy` + `TidyIfNeeded` (1 reintento) + tests | 🔵 ✅ local (`gotest` ✅ 66.1%, sin publicar) |
| L4 | `server`, `devbrowser`, `app` | D1–D5: `AlreadyLogged`, `CompactErrors`, auto-tidy y 1 línea viva (`OperationID`/`LogOpen`/`LogClose`) en todos los manejadores de `BUILD` | 🔵 ✅ local (`gotest` ✅ + `go install`, sin publicar) |
| L5 | `notify`, `appkit` | D6+D7: `notify.NewCriticalFileLog` (`!wasm`, 7 días / 1 MB) + cableado en `appkit/backend` + tests | 🔵 ✅ local (`gotest` ✅ `notify` 97.6% / `appkit` 85.2%, sin publicar) |

## Bloqueos

Ninguno.

## Relación con otras olas

- Complementa `TUI_SIGNAL` (cerrado), `SECURITY_OBSERVABILITY` (cerrado 2026-11-19) y la skill **console-logs**.
