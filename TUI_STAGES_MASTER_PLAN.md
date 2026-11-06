# Pestañas de `webtyp dev` por etapa, con migración propia — master plan
> **Status:** PENDING · 2026-11-05 · fases 1 y 1b cerradas (`ddl` v0.0.45, `postgres` v0.4.36, `sqlt` v0.0.14); fases 2 y 5 listas para plan; DEV_INSTANCES cerrada

**Qué duele hoy.** Al quitar `timezone` del modelo de `business_calendar`, la columna siguió en la base porque tenía un
dato (P47 de DATA_IMPORT). Nadie lo supo: `ddl.Sync` lo anota en un log que no se ve y sigue (`ddl/sync.go:208-211`).

## Preguntas abiertas
Ninguna.

## Decisiones por implementar
| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (P1) | Etapas de izquierda a derecha: `WIZARD` → `SECRETS` → `DATABASE` → `BUILD` → `DEPLOY` → `MCP` | `app/start.go` |
| D2 (P2) | Una lista `h.stages` en ese orden; al arrancar se abre la primera con `Pending()`, si ninguna BUILD; se borran los `SetActiveTab` sueltos (`start.go:219,246`, `section-deploy.go:84,105`). Pendiente nuevo durante el trabajo: no salta, marca la pestaña (`DATABASE •`) y una línea en BUILD | `app` |
| D3 (P3) | DATABASE muestra por columna sobrante: filas, hasta 3 ejemplos, la nota del módulo, y `[Sí] [No, conservarla]`; más `[Migrar ahora]` | `app` (datos de `ddl`, fase 1) |
| D4 (P4) | El módulo declara en su migración a dónde fueron los datos (nota o columna destino con copia antes de borrar); sin declaración, solo datos + pregunta. Forma: D7 | `ddl` + `app` (genera) |
| D5 (P5) | En producción nunca se borra una columna con datos; el servidor avisa al arrancar cuáles sobran (ex D53) | `appkit`/`ddl` |
| D7 (P7) | El «antes» es la base local: columnas de la tabla (`ModelName()`) vs campos del modelo que genera `ormc` al guardar; sin git. Si en un guardado sale un campo y entra otro del mismo tipo, DATABASE propone «¿se fue a X?»; la respuesta queda en una lista `Removed` generada (`MovedTo` copia; `Note` la agrega el agente por MCP) que producción repite sin borrar | `ddl` (`Removed`), `app` (genera) |
| D8 (P8) | La lista `Removed` siempre vive en la app: `config/removed.go` generado (`var Removed []ddl.Removed`), que el `migrate` generado pasa junto a `config.Migrations`; aunque el modelo sea de un módulo (se baja del caché, no se escribe ahí). Otra app del mismo módulo contesta su propia pregunta | `app` (genera), `migrate` (lo recibe) |
| D6 (P6) | El `migrate` generado acepta `-plan` (una línea por columna sobrante) y `-drop tabla.columna`; MCP `db_migrate_plan` y `db_drop_column` llaman lo mismo | `migrate`, `app` |
| D9 (P9) | Los módulos no cambian: `migrate` saca los modelos con `ddl.Capture`, corre los pasos y llama `ddl.DB.Leftovers(tables, removed)` (solo lee; filas hasta 100 + `More`, 3 ejemplos); `-drop` usa `DB.DropColumn` | `ddl`, `migrate` |
| D10 (P10) | `MovedTo` sí: nueva `ddl.OpCopyColumn` (`UPDATE t SET dest = orig WHERE dest IS NULL AND orig IS NOT NULL`) vía `DB.CopyColumn`; `postgres` y `sqlt` la escriben y la prueban en `ddl/conformance` | `ddl`, `postgres`, `sqlt` |

## Fases
| Fase | Repo | Qué | Estado |
|---|---|---|---|
| 1 | `ddl` | `Removed`, `Leftovers`, `DropColumn`, `CopyColumn`, `OpCopyColumn` + conformance (D7, D9, D10) | cerrada v0.0.45 (ejecutada en local) |
| 1b | `postgres`, `sqlt` | escriben `OpCopyColumn`; corren los dos casos nuevos de `ddl/conformance` (D10); `sqlt` además sube `storage` a v0.1.18 (NULL → cero en su conexión de prueba) | cerrada (`postgres` v0.4.36, `sqlt` v0.0.14; conformance real en Postgres local) |
| 2 | `migrate` | `-plan` y `-drop` (D6); `migrate.App` recibe `[]ddl.Removed` (D8); D64: una sola migración al arrancar | lista para plan (usa `ddl` v0.0.45) |
| 3 | `app` | pestaña DATABASE (D3); la migración sale de BUILD (`app_migrate.go`, `section-build.go:338`); genera `config/removed.go` (D7, D8); MCP de D6 | tras fase 2 |
| 4 | `app` | `h.stages` y «primera pendiente» (D1, D2) | tras fase 3 |
| 5 | `appkit`/`ddl` | aviso de columnas sobrantes al arrancar en producción (D5) | lista para plan (usa `ddl` v0.0.45) |
| 6 | `mjosefa-cms` | `timezone` en `config/removed.go` (D8), aparece en DATABASE, se borra con «Sí», capturas y test de la app | última |

## Bloqueos
Ninguno.

## Relación con otras olas
- Sale de DATA_IMPORT: absorbe P47 (columna `timezone`), D53 y D64. DATA_IMPORT sigue sin esperarla.
- DEV_INSTANCES: comparte la pestaña MCP y la regla «lo que hace el dev en la TUI, el agente lo hace por MCP».
- IMPORT_SCREEN: la importación por pantalla podría vivir en DATABASE; se decide allá.
