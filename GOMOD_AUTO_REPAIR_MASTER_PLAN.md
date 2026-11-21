# Reparación automática de go.mod / go.sum — master plan
> **Status:** PENDING · 2026-11-20 · Esperando respuestas a P1–P2 (dueño)
> **Para retomar:** leer skills `master-plan`, `api-design`, `console-logs`; reproducir en `veltylabs/mjosefa-cms` con `replace` hacia un módulo local con `go.sum` incompleto.

## Preguntas abiertas

**P1 — Dónde se repara un módulo `replace` con `go.sum` incompleto (firma del contrato).**
Al recargar un módulo local reemplazado (`~/Dev/Project/veltylabs/modules/client_directory/ui`), `sitec` ejecuta `go` dentro de ese módulo y falla con `missing go.sum entry`. Hoy `SSRFileWatcher` llama `GoModHandler.TidyIfNeeded(err.Error())` (`webtyp/devflow/go_tidy.go:20`), que siempre corre `go mod tidy` en `mjosefa-cms` (`g.rootDir`), dejando roto `client_directory` y fallando el reintento.

```go
// Hoy (webtyp/devflow/go_tidy.go:20 — siempre usa g.rootDir):
ran, err := tidier.TidyIfNeeded(err.Error())

// Opción A — GoModHandler.TidyModuleIfNeeded(dir, buildOutput string) (bool, error):
// Sube desde dir hasta encontrar el go.mod dueño (client_directory/ui → client_directory)
// y corre `go mod tidy` ahí (con mutex por módulo). Si dir == "", usa g.rootDir.
ran, err := tidier.TidyModuleIfNeeded(moduleDir, err.Error())

// Opción B — Mantener TidyIfNeeded(buildOutput) y que GoModHandler parsee la ruta
// desde el texto del error de Go para adivinar qué módulo falló.
ran, err := tidier.TidyIfNeeded(err.Error())
```

- **A)** `TidyModuleIfNeeded(dir, buildOutput string) (bool, error)` (o evolucionar `TidyIfNeeded` recibiendo `dir` explícito): el llamador (`SSRFileWatcher`, `AssetsHandler`, `ServerHandler`, `WasmClient`) ya sabe en qué carpeta corrió `go`, y `GOMOD` resuelve el `go.mod` dueño de esa carpeta sin parsear texto frágil.
- **B)** Parsear rutas desde el stdout/stderr del compilador dentro de `GoModHandler` (frágil: el mensaje de `missing go.sum entry` cita archivos de `~/go/pkg/mod/...`, no el módulo que lo importó).
- **Recomiendo A** (`webtyp/devflow/go_tidy.go:20`, `webtyp/app/ssr_watcher.go:58-63`): es explícito, determinista y funciona igual para la app raíz y cualquier `replace` local. ¿Cuál evita el acoplamiento y es más mantenible?
- Respuesta:

**P2 — Cuándo reparar: solo reactivo (al fallar un build/SSR) o también preventivo al arrancar.**
En el arranque de `webtyp dev`, `EnsureGoModSynced` (`webtyp/app/section-build.go:58`) revisa solo `mjosefa-cms`, y `ScheduleSSRLoad` (`webtyp/app/ssr_loader.go:263`) clasifica `missing go.sum entry` como error determinista y aborta sin llamar a `GOMOD`.

```go
// Opción A — Solo reactivo (en los 4 consumidores: SSR arranque, SSR hot-reload, SERVER, CLIENT):
// Si ExtractAll / ReloadSSRModule / CompileProgram falla con NeedsGoModTidy(err),
// llama tidier(moduleDir, err) una vez y reintenta. Costo 0 ms cuando todo está sano.

// Opción B — Preventivo al arrancar + reactivo:
// EnsureGoModSynced recorre también todos los replace locales con `go list -m -json all`
// al arrancar, además del reintento reactivo.
```

- **A)** Solo reactivo en los 4 puntos de compilación (incluido `ScheduleSSRLoad` en arranque): cero latencia extra cuando los `go.mod`/`go.sum` están sanos; si falta una entrada en `go.sum` (app o `replace`), `GOMOD` la repara en el acto y reintenta 1 vez de forma transparente.
- **B)** Preventivo sobre todos los `replace` al arrancar + reactivo: detecta desync antes de que `sitec` corra, pero suma N invocaciones de `go list` en cada arranque cuando hay muchos `replace` locales.
- **Recomiendo A** (`webtyp/app/ssr_loader.go:263`, `webtyp/app/section-build.go:136`): mantiene el arranque rápido (mismo principio de `ModuleFingerprint`) y resuelve de forma transparente tanto el arranque (`ExtractAll`) como el hot-reload (`ReloadSSRModule`, `SERVER`, `CLIENT`).
- Respuesta:

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| — | (Pendiente cerrar P1–P2) | — |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| G1 | `webtyp/devflow` | Reparación de `go.mod`/`go.sum` por directorio de módulo (buscando el `go.mod` dueño + serialización por módulo) y tests unitarios | 🔴 espera P1–P2 |
| G2 | `webtyp/sitec` | Si `ExtractAll` falla en un módulo `replace`, incluir el `moduleDir` en el error tipado (o en el callback) para que `app` sepa qué módulo reparar | 🔴 espera P1–P2 |
| G3 | `webtyp/server`, `webtyp/app` | Conectar el tidier por módulo en `ScheduleSSRLoad` (arranque), `SSRFileWatcher` (hot-reload), `ServerHandler` y `WasmClient` + test de integración | 🔴 espera G1–G2 |

## Bloqueos

Ninguno.

## Relación con otras olas

- [DEV_LOGS_DX](MASTER_PLANS.md) (cerrado): introdujo `NeedsGoModTidy` y `TidyIfNeeded` solo para `g.rootDir`; este master extiende esa capacidad a módulos `replace` locales y al escaneo SSR de arranque.
- [LOG_NOISE_MASTER_PLAN.md](LOG_NOISE_MASTER_PLAN.md): los avisos de `GOMOD` siguen la regla de una línea por operación (`console-logs`).
