# Pestañas de `webtyp dev` por etapa, con migración propia — master plan
> **Status:** IN PROGRESS · 2026-11-05 · fases 1, 1b, 2, 3a y 5 cerradas; sigue 3b (`app`: «¿se fue a X?», `config/removed.go`, MCP); luego 4 y 6

**Qué duele hoy.** Al quitar `timezone` del modelo de `business_calendar`, la columna siguió en la base porque tenía un
dato (P47 de DATA_IMPORT). Nadie lo supo: `ddl.Sync` lo anota en un log que no se ve y sigue (`ddl/sync.go:208-211`).

## Preguntas abiertas
Ninguna.

## Decisiones por implementar
| # | Decisión | Dónde se implementa |
|---|---|---|
| D14 | El navegador de desarrollo se abre solo cuando ninguna etapa tiene `Pending()` (WIZARD, SECRETS, DATABASE con fallo o columna sin decidir); al resolverse lo último pendiente se abre solo. Hoy se abre al arrancar el servidor y tapa la consola (pedido del dueño, 2026-11-05) | `app` (fase 4) |
| D1 (P1) | Etapas de izquierda a derecha: `WIZARD` → `SECRETS` → `DATABASE` → `BUILD` → `DEPLOY` → `MCP` | `app/start.go` |
| D2 (P2) | Una lista `h.stages` en ese orden; al arrancar se abre la primera con `Pending()`, si ninguna BUILD; se borran los `SetActiveTab` sueltos (`start.go:219,246`, `section-deploy.go:84,105`). Pendiente nuevo durante el trabajo: no salta, marca la pestaña (`DATABASE •`) y una línea en BUILD | `app` |
| D4 (P4) | El módulo declara en su migración a dónde fueron los datos (nota o columna destino con copia antes de borrar); sin declaración, solo datos + pregunta. Forma: D7 | `ddl` + `app` (genera) |
| D7 (P7) | El «antes» es la base local: columnas de la tabla (`ModelName()`) vs campos del modelo que genera `ormc` al guardar; sin git. Si en un guardado sale un campo y entra otro del mismo tipo, DATABASE propone «¿se fue a X?»; la respuesta queda en una lista `Removed` generada (`MovedTo` copia; `Note` la agrega el agente por MCP) que producción repite sin borrar | `ddl` (`Removed`), `app` (genera) |
| D8 (P8) | La lista `Removed` siempre vive en la app: `config/removed.go` generado (`var Removed []ddl.Removed`), que el `migrate` generado pasa junto a `config.Migrations`; aunque el modelo sea de un módulo (se baja del caché, no se escribe ahí). Otra app del mismo módulo contesta su propia pregunta | `app` (genera), `migrate` (lo recibe) |
| D6 (P6) | El `migrate` generado acepta `-plan` (una línea por columna sobrante) y `-drop tabla.columna`; MCP `db_migrate_plan` y `db_drop_column` llaman lo mismo | `migrate`, `app` |

## Fases
| Fase | Repo | Qué | Estado |
|---|---|---|---|
| 1 | `ddl` | `Removed`, `Leftovers`, `DropColumn`, `CopyColumn`, `OpCopyColumn` + conformance (D7, D9, D10) | cerrada v0.0.45 (ejecutada en local) |
| 1b | `postgres`, `sqlt` | escriben `OpCopyColumn`; corren los dos casos nuevos de `ddl/conformance` (D10); `sqlt` además sube `storage` a v0.1.18 (NULL → cero en su conexión de prueba) | cerrada (`postgres` v0.4.36, `sqlt` v0.0.14; conformance real en Postgres local) |
| 2 | `migrate`, `server` | `-plan`/`-drop`, `config.Schema`, `config/removed.go` creado una vez (D6, D8, D9, D12) ; P13=A: al cambiar lo generado, `go get` @latest de appkit/ddl/devtools/migrate en la app | cerrada (`migrate` v0.1.0, `server` v0.2.114, `ddl` v0.0.46) |
| 3a | `app` | pestaña DATABASE (D3, D13): la migración sale de BUILD, informe completo, un campo que cambia según el estado (sobrante: Sí/No; fallo: Actualizar librerías/Reintentar; nada: Migrar ahora) | cerrada (`app` v0.7.71, `server` v0.2.115 `UpgradeGeneratedLibraries`) |
| 3b | `app` | «¿se fue a X?» al guardar (D7), escribir `config/removed.go` con `MovedTo`/`Note` (D8), MCP `db_migrate_plan` y `db_drop_column` (D6) | siguiente |
| 4 | `app` | `h.stages` y «primera pendiente» (D1, D2); el navegador espera a que no quede nada pendiente (D14) | tras fase 3 |
| 5 | `devtools`, `appkit`, `server` | aviso de columnas sobrantes al arrancar (D5, D11); `config.Schema` en las tres (D12) | cerrada (`devtools` v0.0.7, `appkit` v0.2.17, `server` v0.2.113; probada con una app generada sobre sqlite) |
| 6 | `mjosefa-cms` | `timezone` en `config/removed.go` (D8), aparece en DATABASE, se borra con «Sí», capturas y test de la app | última |

## Bloqueos
Ninguno.

## Relación con otras olas
- Sale de DATA_IMPORT: absorbe P47 (columna `timezone`), D53 y D64. DATA_IMPORT sigue sin esperarla.
- DEV_INSTANCES: comparte la pestaña MCP y la regla «lo que hace el dev en la TUI, el agente lo hace por MCP».
- IMPORT_SCREEN: la importación por pantalla podría vivir en DATABASE; se decide allá.
