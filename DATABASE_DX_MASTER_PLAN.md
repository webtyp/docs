# Experiencia de desarrollo en DATABASE y MIGRATE — master plan
> **Status:** IN PROGRESS · 2026-11-17 · fases 0, 1 y 2 ejecutadas en local (`gotest` verde); esperando revisión del dueño para publicar en cascada (`ddl`, `tui` → `devtui` → `app`)

## Preguntas abiertas
Ninguna.

## Decisiones por implementar
| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (P1) | Todos los textos de `DATABASE`/`MIGRATE` pasan por `lang.Translate` + `app/lang.json` (español completo en local) | `app` |
| D2 (P2) | Cabecera explicativa en `MIGRATE` + pie `i/N · tabla.columna` con botones explícitos | `app` |
| D3 (P3) | «No, conservar columna» persiste `Keep: true` en `removed.go` (`ddl.Removed.Keep`) | `ddl`, `app` |
| D4 (P4) | Interacción paso a paso (`WaitingForUser()` en `HandlerSelection`): 1 columna a la vez (`1/N`), activación automática del pie y avance automático | `tui`, `devtui`, `app` |
| D5 (P5) | `appMigrator.Pending()` sigue bloqueando el navegador mientras haya decisiones pendientes | `app` (sin cambios) |
| D6 (P6) | `ddl.Leftovers` compara contra `withTextSearchColumn(t.Model.Schema())` para ignorar `text_search` | `ddl` |

## Fases
| Fase | Repo | Qué | Estado |
|---|---|---|---|
| 0 | `ddl` | `Leftovers` usa `withTextSearchColumn` (D6) y omite `Removed.Keep` (D3); tests en `ddl/tests/leftovers_test.go` | 🟡 en local · `gotest ./...` ✅ |
| 1 | `tui`, `devtui` | `Classify` prioriza `HandlerSelection` sobre `HandlerInteractive`; `NewSelectionHandler` cablea `WaitingForUser()` y auto-activa/avanza paso a paso (D4) | 🟡 en local (`devtui` con `replace ../tui`) · `gotest ./...` ✅ |
| 2 | `app` | `app_migrate.go` interactivo paso a paso (1/N, cabecera, muestras truncadas), `app_migrate_removed.go` persiste `Keep: true`, `app/lang.json` en `es`, `webtyp` instalado en local | 🟡 en local (`app` con `replace ../ddl, ../tui, ../devtui`) · `gotest ./...` ✅ |

## Bloqueos
Ninguno.

## Relación con otras olas
- Extiende [TUI_STAGES_MASTER_PLAN.md](TUI_STAGES_MASTER_PLAN.md) (une la escritura de `removed.go` de su fase 3b con D3).
- Relacionado con [RECORD_SEARCH_MASTER_PLAN.md](RECORD_SEARCH_MASTER_PLAN.md) (introdujo `text_search` en `ddl`).
