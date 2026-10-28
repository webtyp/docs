# Diagrama de la base de datos — master plan
> **Status:** IN PROGRESS · 2026-10-28 · `erd` v0.0.2 e `icons` v0.0.38 publicados; `ormc` y `ddl` cerrados; `erd` (CLI) y `appkit` despachados (revisa Antigravity); `layout` v0.3.79 cerrada; E5a despachada (Antigravity)

**Meta:** toda app webtyp con base de datos trae, sin escribirlo, un grupo «Developer tools» con el
explorador de API y un enlace «Base de datos» que dibuja tablas y relaciones y marca en rojo los errores.

## Preguntas abiertas

Ninguna.

## Decisiones por implementar
| # | Decisión | Dónde se implementa |
|---|---|---|
| D7 (P3) | `ddl.Step{Name, Run}` (el contrato que hoy cada app declara suelto) y `ddl.Capture(steps)`: corre las migraciones contra un registrador que no toca base y devuelve cada tabla con su paso | `webtyp/ddl` |
| D8 (P4) | `appkit.ServerConfig.Migrations []ddl.Step` no vacío → `GET /_schema` con `Requires(erd.ResourceSchema, model.Read)`, una banda por paso; nunca `Public()` | `webtyp/appkit` |
| D9 (P5, P6) | Al guardar `model.go`, `app` reescribe `docs/diagrams/database.svg` con `erd/repo.Write` tras `ormc`; sin modelos no escribe nada. La app no escribe archivo: su servidor dibuja en memoria | `webtyp/app` |
| D10 (P7) | Permisos `api_explorer` y `db_schema` por enlace; el grupo muestra solo los permitidos y se oculta sin ninguno (test nuevo en E5a). Una app sin base nunca otorga `db_schema` → no ve el enlace | `webtyp/layout` |
| D11 (P8) | Grupo «Developer tools» (id `devtools`, `terminal`): «API Explorer» (`code`) y «Base de datos» (`database`) | `webtyp/layout` |
| D12 (P9, P11) | El botón DDL pasa a `app/ddl_button.go`; sin modelos dice «… open Developer tools → Database»; `ddlc` sale del `go.mod` y el repo se archiva | `webtyp/app`, GitHub |
| D14 | `ormc.Models(root)` (orden de archivo) es la única forma de leer los modelos de un repo; `erd/repo.Write` y el comando `erd` la usan; la raíz de `erd` nunca importa `ormc` (la importa la pantalla del navegador) | `webtyp/ormc`, `webtyp/erd` |

## Fases
| Fase | Repo | Qué | Estado |
|---|---|---|---|
| E2 | `webtyp/erd` | grafo, hallazgos, SVG determinista, `Mount` (diseño en `erd/docs/ARCHITECTURE.md`) | 🟢 cerrada v0.0.2 (local) |
| E3 | `webtyp/icons` | glifos `database` y `code` | 🟢 cerrada v0.0.38 (local) |
| E7a | `webtyp/orm` | README sin `ddlc` | 🟢 cerrada v0.12.19 |
| E1 | `webtyp/ddl` | D7 — `Step`, `Table`, `Capture` | 🟢 cerrada v0.0.32 |
| E1b | `webtyp/ormc` | D14 — `Models(root)` | 🟢 cerrada v0.1.27 (el agente también corrigió que el stub guardaba `UserModel` como tabla destino; ahora una sola `refTableName`) |
| E2b | `webtyp/erd` | D14 — `repo.Write` + `cmd/erd`, `docs/PLAN.md` (v0.0.3) | [Claude · 2026-10-28 14:45] despachado (jules, ses. 17258430540539880632) |
| E4 | `webtyp/appkit` | D8 — `docs/PLAN.md` (v0.0.7) | [Claude · 2026-10-28 14:47] despachado (jules, ses. 13470001113254648269) |
| E5a | `webtyp/layout` | D10, D11 — `docs/PLAN.md` (v0.3.80) | [Antigravity · 2026-10-28 15:15] despachado (jules, ses. 9772909075004117390) |
| E5b | `webtyp/layout` | pantalla `schemaview` — `docs/PLAN_SCHEMAVIEW.md` (v0.3.81) | ⚪ aparcada tras E5a (Antigravity) |
| E6 | `webtyp/app` | D9, D12 — `docs/PLAN.md` | ⚪ despachar tras `erd` v0.0.3 |
| E7b | GitHub `webtyp/ddlc` | archivar el repo (el usuario, a mano) | ⚪ tras E6 publicado |
| E8 | 13 módulos `veltylabs` | por módulo, cuando cierre su plan de REACTIVE_RECORDS: `go install webtyp.com/erd/cmd/erd@latest && erd` en la raíz; borrar `docs/diagrams/database.md`; en `docs/ARCHITECTURE.md` cambiar su enlace por `![Base de datos](diagrams/database.svg)`; `gopush`. Al terminar: quitar las líneas `STATUS` de **module-layout** y **documentation** | ⚪ tras E2b; `payment_manager`, `room_layout` con plan en vuelo |
| Z | `mjosefa-cms` | `config.MigrationStep` → `ddl.Step` (se borra el tipo propio); `Migrations: config.MigrationSteps()` en el servidor; rol admin con `db_schema:read`; captura revisada del grupo y del diagrama; test que recorre el menú como admin y sin permisos (todo enlace visible abre algo no vacío) | ⚪ tras E4, E5b y B2 de APP_COMPOSITION |

## Bloqueos
- 2026-10-28 · La cascada de `ddl` v0.0.32 dejó sin actualizar 7 dependientes por pruebas rojas que ya fallaban con v0.0.31 (no las causa `Capture`): `rbac` (`TestMigrateDetectsPreexistingDuplicates`), `sqlt`, `sqlt/tests`, `goflare` (falta `web/edge/main.go`), `agentmemory`, y `patient_directory`/`staff_manager` (usan `model.RowOf`, que ya no existe: espera REACTIVE_RECORDS G3/G4). Ninguno bloquea esta ola, pero deben ponerse al día con `ddl` por su cuenta.
- 2026-10-28 · `webtyp/layout` v0.3.79 cerrada y publicada: E5a en curso; E5b espera a E5a.

## Relación con otras olas
- REACTIVE_RECORDS: G2 (`layout` v0.3.79) cambia el `BuildMenu` que tocan E5a/E5b; F5 se diseña sobre `appkit` v0.0.7 (ya con `Migrations`); G3/G4 tienen planes en módulos de E8.
- APP_COMPOSITION: agrega «Developer tools» a la lista D10; cambia el lugar del explorador que dejó R0b (de suelto a grupo); Z depende de B2.
