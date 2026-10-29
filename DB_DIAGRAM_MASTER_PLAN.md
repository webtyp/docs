# Diagrama de la base de datos — master plan
> **Status:** IN PROGRESS · 2026-10-28 · sin preguntas; E10, E11 y Z hechos en local y vistos en mjosefa-cms (claro y oscuro), sin publicar: espera tu orden de publicar (orden en Bloqueos)

**Meta:** toda app webtyp con base de datos trae, sin escribirlo, un grupo «Developer tools» con el
explorador de API y un enlace «Base de datos» que dibuja tablas y relaciones y marca en rojo los errores.

## Preguntas abiertas

Ninguna.

## Decisiones por implementar
| # | Decisión | Dónde se implementa |
|---|---|---|
| D12 (P9, P11) | El repo `ddlc` se archiva: ya nadie lo importa (`app` v0.7.22 tiene el botón) | GitHub (tú, a mano) |

## Fases
| Fase | Repo | Qué | Estado |
|---|---|---|---|
| E1–E7a | `ddl`, `ormc`, `erd`, `icons`, `orm`, `appkit`, `layout`, `app` | `Capture`, `Models`, diagrama + CLI, glifos, `/_schema`, grupo «Developer tools» + pantalla, dibujo al guardar | 🟢 cerradas: ddl v0.0.32, ormc v0.1.27, erd v0.0.4, icons v0.0.38, orm v0.12.19, appkit v0.0.7, layout v0.3.81, app v0.7.22 |
| E7b | GitHub `webtyp/ddlc` | archivar el repo | ⚪ listo (tú) |
| E8 | 13 módulos `veltylabs` | por módulo, cuando cierre su plan de REACTIVE_RECORDS: `go install webtyp.com/erd/cmd/erd@latest && erd` en la raíz; borrar `docs/diagrams/database.md`; en `docs/ARCHITECTURE.md` cambiar su enlace por `![Base de datos](diagrams/database.svg)`; `gopush`. Al terminar: quitar las líneas `STATUS` de **module-layout** y **documentation** | ⚪ tras publicar E10 |
| E9 | `veltylabs/staff_manager` | declarar sus claves foráneas, que hoy no tiene ninguna: `staff_device.staff_id` → `staff_member`, `staff_device.device_id` → `device` (ya importa `device_manager`), `staff_member.user_id` → `user` (ya importa `auth`) | ⚪ escribir plan |
| E10 | `widget`, `ormc`, `erd` | sin CSS a mano: recetas SVG `Paint`/`Ink`/`Line`/`Dashed`/`Typeface` y `erd/css.go` (el diagrama sigue el tema claro/oscuro; el archivo suelto lleva la hoja generada); solo lo que declara el modelo (se borró la regla que adivinaba por `_id`: 30 avisos → 7, todos reales); `Options` = línea punteada; tipo de la base o unidad del instante; `ormc.Models` conserva `Options` y unidades | 🟡 hecho en local, sin publicar (`replace` en `erd`, `layout`, `mjosefa-cms`) |
| E11 | `webtyp/layout` (`schemaview`) | el diagrama no se veía (el lienzo medía 2 px: la lista de avisos no se encogía): avisos en columna lateral (`style.Sidebar`), cada zona con su scroll | 🟡 hecho en local, sin publicar |
| Z | `mjosefa-cms` | `docs/PLAN.md`: `appkit.MountServer` con `Migrations: config.MigrationSteps()` (`[]ddl.Step`, se borra `MigrationStep`), test `/_schema` 200 con permiso / 403 sin él. El admin ya lo ve por `*:all` (`config/auth.go:186`). Luego: captura del grupo y del diagrama | 🟡 ejecutado en local (sin publicar): `/_schema` responde y la pantalla dibuja las 18 tablas; falta publicar |

## Bloqueos
- 2026-10-28 · Publicar en este orden (cada `gopush` quita el `replace` del siguiente): `widget` → `ormc` → `sitec` → `erd` → `layout` → `appkit` → `app` → `mjosefa-cms`. Espera tu orden.
- 2026-10-28 · `rbac`, `sqlt`, `goflare` siguen con `ddl` v0.0.15 y `agentmemory` con v0.0.20, por pruebas rojas suyas (`rbac`: `TestMigrateDetectsPreexistingDuplicates`; `goflare`: falta `web/edge/main.go`). No bloquean esta ola.

## Relación con otras olas
- APP_COMPOSITION: Z de aquí es la mitad «contratos» de su B2; `config/server.go` sigue con el transporte.
- REACTIVE_RECORDS: F5 se diseña sobre `appkit` v0.0.7 (ya con `Migrations`).
- MVP mjosefa-cms: la prueba local (fase M2b del roadmap) es donde se ve esto funcionando.
