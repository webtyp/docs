# Generador de los `main` de la app — master plan
> **Status:** IN PROGRESS · 2026-11-17 · D3 corregida y republicada: `server` v0.2.130 · `appkitc` v0.0.3 · `app` v0.8.26 · `mjosefa-cms` v0.0.128; faltan `devskills` y `docs` (cambios ajenos sin commit).

Gatillo (2026-11-17): `webtyp dev` en `mjosefa-cms` falló con `undefined: layoutapp.Browser` y `config.App.Server undefined`. El binario `webtyp` llevaba `server` v0.2.127 y reescribió `web/browser/main.go` y `migrate/main.go` con la plantilla vieja, aunque la app ya usaba `appkit` v0.2.38 y `layout` v0.3.130. Se reinstaló el binario a mano; esto es la causa de raíz.
Reglas: **api-design** (puerta 4), **api-harness** (regla 9 y «Publication»), **isomorphic-libraries**, **zero-debt**. Dominio vecino: [BUILD_DESTINATIONS](BUILD_DESTINATIONS_MASTER_PLAN.md) decide *qué* archivos genera (D21–D24); aquí se decide *quién* los genera y *con qué versión*.

## Preguntas abiertas

Ninguna.

## Decisiones por implementar
| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (P1) | `appkitc` es un **repo propio** (`gonew`): la plantilla y la generación salen de `server`; ni `appkit` ni `server` lo importan (la herramienta conoce el runtime, nunca al revés). Lo ejecuta `webtyp dev`; producción no lo consume, lo prueba `go list -deps` de los tres `main` del fixture (ninguno depende de `appkitc`; `text/template` y `command` pueden aparecer por otras librerías y no se comprueban). Razón: `mjosefa-cms/go.mod:41` exige `server` en producción y `server/go.mod:9` trae `command` (`server` sigue usándolo para el generador y la compilación) | `appkitc` (fase 2) |
| D2 (P2) | Se llama `appkitc`: el sufijo `c` de `definitionc`, `langc`, `sitec`, `weightsc`, `avatarc` | `gonew appkitc … --owner webtyp` (fase 2) |
| D3 (P3, corregida 2026-11-17) | `appkitc` es una herramienta de desarrollo **dentro del programa `webtyp`**: `app` lo importa y lo conecta al manejador con `SetMainGenerator` (`server` no puede importarlo: `appkitc` importa `server`); **ninguna app consumidora lo lista en su `go.mod`** (la versión con `tool` por app se descartó: ponía una herramienta de desarrollo donde no se usa). `appkitc` conserva `UpgradeGeneratedLibraries` (`go get <librería>@latest`). Sin generador conectado, `server.ErrNoMainGenerator`. Diagramas: [appkitc/docs/diagrams/GENERATED_MAINS.md](../appkitc/docs/diagrams/GENERATED_MAINS.md) | `appkitc`, `server`, `app` |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| 1 | — | Plan de `appkitc` | ✅ no se escribió: por la regla de costo y orden del dueño (2026-11-17) se implementó directo; `Ecosystem search` y `Design gate` quedan en D1–D3 y en [appkitc/docs/ARCHITECTURE.md](../appkitc/docs/ARCHITECTURE.md) |
| 2 | `appkitc` (v0.0.1 creado) | Plantillas, `GenerateMain`, CLI `cmd/appkitc`, fixture compilado (nativo y wasm), prueba de grafo de producción, docs | ✅ local · `gotest` verde (vet, style, race, tests, índice) |
| 3 | `server`, `app`, `mjosefa-cms` | `server`: `project.go`, `generator.go` (`MainGenerator`, `SetMainGenerator`, `ErrNoMainGenerator`), sin `maingen.go`; `app`: `main_generator.go` y wiring en `section-build.go`; `mjosefa-cms` sin `tool` ni `appkitc` en el `go.mod` | ✅ local · `gotest` verde en `server`, `appkitc`, `app`; `mjosefa-cms` compila nativo y wasm y `webtyp dev` arranca (`/health` 200) |
| 4 | `devskills` | **app-server-main**, **documentation** | ✅ local · `devskills -f` ok |
| 5 | `server` v0.2.130 · `appkitc` v0.0.3 · `app` v0.8.26 · `mjosefa-cms` v0.0.128 | 🔵 Republicados de abajo hacia arriba con `gopush --no-cascade`, sin `replace` (2026-11-17); v0.2.129, v0.0.2 y v0.0.127 quedan como versiones con el diseño anterior | ✅ |
| 6 | `devskills`, `docs` | Publicar mis cambios (**app-server-main**, **documentation**, este master): el árbol de `devskills` trae cambios ajenos en **api-design** y **master-plan**, y `docs` el archivo sin seguimiento `DATABASE_DX_MASTER_PLAN.md` | ⬜ espera al dueño |

## Bloqueos
- 2026-11-17 · Versiones con el diseño anterior, sin borrar (regla 10): `server` v0.2.129, `appkitc` v0.0.2, `mjosefa-cms` v0.0.127; ninguna es la última de su repo. Un `webtyp dev` abierto antes de v0.8.26 vuelve a añadir `tool` y `appkitc` al `go.mod` de la app (pasó una vez): reiniciarlo.
- 2026-11-17 · `devskills` y `docs` siguen sin commit (fase 7) y traen cambios ajenos (**api-design**, **master-plan**, `DATABASE_DX_MASTER_PLAN.md`).
- 2026-11-17 · **Para revisar (dueño), nombres elegidos sin `D<n>`**: en `server`, `MainGenerator`, `SetMainGenerator`, `ErrNoMainGenerator`; exportados para `appkitc` y `app`: `HasRunServer`, `ReadModulePath`, `ConfigDir`, `AppVarName`, `GeneratedMigrateDir`; en `appkitc`, `APPKITC_UPDATE_FIXTURE`. Sin protección contra un `webtyp` instalado más viejo que las librerías de la app (se descartó el sello de revisión): solo la prueba del fixture al publicar y que `gopush` reinstala `webtyp`.
- 2026-11-17 · Aviso aparte: `webtyp dev` en `mjosefa-cms` loguea `queued_call` inexistente y 12 columnas `text_search` sobrantes con datos (RECORD_SEARCH).

## Relación con otras olas
- [BUILD_DESTINATIONS](BUILD_DESTINATIONS_MASTER_PLAN.md): D21–D24 fijan qué archivos genera; este master no los cambia.
- [ARCHITECTURE](ARCHITECTURE_MASTER_PLAN.md): D62–D74 (módulos sin `appkit`) dejaron las plantillas que hoy se mueven.
- [EXECUTION_ORDER](EXECUTION_ORDER_MASTER_PLAN.md) (CLOSED): su paso 17c generó los `main` con la plantilla nueva que el binario viejo pisó.
