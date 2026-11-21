# Reparación automática de go.mod / go.sum — master plan
> **Status:** IN PROGRESS · 2026-11-21 · `sitec` v0.2.102 y `server` v0.2.143 publicados; `/home/cesar/go/bin/webtyp` recompilado en local; `devflow` y `app` en espera de que el otro agente termine en `webtyp/devflow`
> **Para retomar:** leer skills `master-plan`, `codejob`; cuando el otro agente termine en `webtyp/devflow`, publicar `devflow` (`gopush`) y luego `app` (`gopush`, que reconcilia los `replace` de `devflow`/`sitec`/`server`).

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (P1=A) | Reemplazar `GoModHandler.TidyIfNeeded(buildOutput)` por `TidyModuleIfNeeded(dir, buildOutput string) (bool, error)`: sube desde `dir` hasta el `go.mod` dueño (si `dir == ""` usa `g.rootDir`), serializa `go mod tidy` con mutex por módulo y registra una línea con `wpath.Tilde(modDir)` | `webtyp/devflow` (local, sin publicar) |
| D2 (P2=A) | Reparación solo reactiva en los 4 puntos de compilación (`ScheduleSSRLoad` arranque, `SSRFileWatcher` hot-reload, `ServerHandler` arranque/restart, `WasmClient` `RecompileMainWasm`): reintenta 1 vez si `TidyModuleIfNeeded` corrió; `sitec` envuelve errores con `*sitec.ModuleBuildError{ModuleDir, Err}` y ejecuta `ExtractModule` en el módulo raíz de la app para `replace` locales | `webtyp/sitec` v0.2.102 ✅, `webtyp/server` v0.2.143 ✅, `webtyp/app` (local, sin publicar) |

## Fases

Cerrado (publicado, sin pendientes): `sitec` v0.2.102 · `server` v0.2.143.

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| G1 | `webtyp/devflow` | `TidyModuleIfNeeded(dir, buildOutput)` con búsqueda ascendente de `go.mod` + mutex por módulo (`go_mod.go`, `go_tidy.go`, `tests/go_tidy_test.go`) | 🟡 local verificado (`gotest` ✅ 66.5%) — pausado hasta que otro agente termine en `devflow` |
| G3 | `webtyp/app` | `SetGoModTidier` conectado en `AssetsHandler.ScheduleSSRLoad`, `SSRFileWatcher`, `ServerHandler` y `WasmClient.RecompileMainWasm` (`client*.go`, `ssr_loader.go`, `ssr_watcher.go`, `section-build.go`, `tests/gomod_autorepair_test.go`) | 🟡 local verificado (`gotest` ✅ 67.1% con `replace`) — espera publicación de `devflow` |

### Trabajo local sin commitear
- `webtyp/devflow`: `go_mod.go`, `go_tidy.go`, `tests/go_tidy_test.go` *(cambios en curso de otro agente: `docs/GONEW.md`, `gotest.go`, `gotest_destinations.go`, `tests/gonew_test.go` — no tocar ni publicar aún)*.
- `webtyp/app`: `client.go`, `client_change.go`, `client_extensions.go`, `section-build.go`, `ssr_loader.go`, `ssr_watcher.go`, `tests/gomod_autorepair_test.go`, y `replace` temporales en `go.mod` (`webtyp.com/devflow => ../devflow`, `webtyp.com/server => ../server`, `webtyp.com/sitec => ../sitec`).
- Binario instalado desde local: `~/go/bin/webtyp` (`go install ./cmd/webtyp` desde `webtyp/app`).
- Al desbloquear `devflow`: 1) `gopush` en `webtyp/devflow`, 2) `gopush` en `webtyp/app` (reconcilia automáticamente los 3 `replace`), 3) borrar este master y su fila en `MASTER_PLANS.md`.

## Bloqueos

- 2026-11-20 · `webtyp/devflow` tiene trabajo en curso de otro agente (`gotest.go`, `gotest_destinations.go`, `docs/GONEW.md`, `tests/gonew_test.go`); se pospone `gopush` de `devflow` y `app` por instrucción del dueño.

## Relación con otras olas

- [DEV_LOGS_DX](MASTER_PLANS.md) (cerrado): introdujo `NeedsGoModTidy` y `TidyIfNeeded` solo para `g.rootDir`; este master extiende esa capacidad a módulos `replace` locales y a los 4 puntos de compilación.
- [LOG_NOISE_MASTER_PLAN.md](LOG_NOISE_MASTER_PLAN.md): los avisos de `GOMOD` siguen la regla de una línea por operación (`console-logs`).
