# Experiencia de desarrollo en DATABASE y MIGRATE — master plan
> **Status:** IN PROGRESS · 2026-11-17 · sin preguntas abiertas (D1–D6 cerradas); fases 0, 1 y 2 listas para ejecutar

## Preguntas abiertas
Ninguna.

## Decisiones por implementar
| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (P1) | Todos los textos de la pestaña `DATABASE` y logs/botones de `MIGRATE` (`app_migrate.go:17-50,243-249`, `section-database.go:19`) pasan por `lang.Translate` + `app/lang.json` respetando el idioma activo de la TUI | `app` |
| D2 (P2) | Cuando hay columnas sobrantes, `MIGRATE` imprime primero una cabecera explicativa clara (el modelo ya no usa estas columnas pero tienen datos en la BD local y el navegador espera tu decisión) y el pie muestra `i/N · tabla.columna` con botones explícitos (`Sí, borrar columna y datos` / `No, conservar columna`) | `app/app_migrate.go` |
| D3 (P3) | Elegir «No, conservar columna» persiste la decisión en el `removed.go` de la app (`config/removed.go` o `config/services/removed.go`) para que no vuelva a preguntarse en cada arranque de `webtyp dev` (se elimina `m.kept` solo en memoria de `app_migrate.go:83,285`) | `ddl`, `migrate`, `app` |
| D4 (P4) | Interacción paso a paso (`WaitingForUser() bool` / `HandlerInteractive` + `HandlerSelection`): `MIGRATE` muestra en pantalla **una sola columna sobrante a la vez** (`1 de N`, con su tabla, filas y ejemplos), el pie entra automáticamente en modo interactivo cuando `WaitingForUser()` es `true` (`◄► opción • ↵ confirmar`), y al confirmar avanza a la siguiente (`2 de N`) hasta terminar | `tui`, `devtui`, `app/app_migrate.go` |
| D5 (P5) | `appMigrator.Pending()` (`app_migrate.go:90`) mantiene su comportamiento actual: retiene la apertura del navegador mientras haya bloqueo de datos pendientes de decidir en `DATABASE` | `app` (sin cambios) |
| D6 (P6) | `ddl.Leftovers` (`ddl/leftovers.go:77`) compara contra `withTextSearchColumn(t.Model.Schema())` igual que `ddl.Sync` (`ddl/sync.go:114`), para no reportar nunca la columna interna `text_search` como columna sobrante | `ddl` |

## Fases
| Fase | Repo | Qué | Estado |
|---|---|---|---|
| 0 | `ddl`, `migrate` | `Leftovers` usa `withTextSearchColumn(t.Model.Schema())` (D6) y respeta columnas ya decididas/conservadas en `Removed` (D3); conformance en `ddl/conformance` y test en `ddl/tests/leftovers_test.go` | 🔴 pendiente |
| 1 | `tui`, `devtui` | Soportar `WaitingForUser() bool` en `HandlerSelection` (`Classify` prioriza `HandlerSelection` cuando tiene `Options()`, `NewSelectionHandler` cablea `WaitingForUser`, auto-activación al refrescar y avance paso a paso sin cerrar el modo interactivo mientras `WaitingForUser()` siga `true`) (D4) | 🔴 pendiente |
| 2 | `app` | Flujo interactivo paso a paso en `app_migrate.go` (1 columna a la vez con `WaitingForUser()`, cabecera explicativa, pie `i/N · tabla.columna`), persistencia de «Conservar» en `removed.go` (D3), y traducción completa en `app/lang.json` (D1, D2, D4) | 🔴 pendiente |

## Bloqueos
Ninguno.

## Relación con otras olas
- Extiende [TUI_STAGES_MASTER_PLAN.md](TUI_STAGES_MASTER_PLAN.md) (une la escritura de `removed.go` de su fase 3b con D3).
- Relacionado con [RECORD_SEARCH_MASTER_PLAN.md](RECORD_SEARCH_MASTER_PLAN.md) (introdujo `text_search` en `ddl`).
