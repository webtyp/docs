# Diagrama de la base de datos — master plan
> **Status:** IN PROGRESS · 2026-10-28 · librerías cerradas (`erd` v0.0.4, `appkit` v0.0.7, `app` v0.7.22, `layout` v0.3.81); falta que mjosefa-cms lo monte (plan escrito, sin despachar) y responder P12–P14, que deciden cuánto ruido muestra el diagrama

**Meta:** toda app webtyp con base de datos trae, sin escribirlo, un grupo «Developer tools» con el
explorador de API y un enlace «Base de datos» que dibuja tablas y relaciones y marca en rojo los errores.

## Preguntas abiertas

Probé `erd` con las migraciones reales de mjosefa-cms (5 bandas, 18 tablas): **0 errores y 30 avisos**, y
casi todos son ruido de las tres causas de abajo. Con 30 avisos nadie mira el diagrama.

**P12 — `tenant_id` y `project_id` salen como «referencia no declarada» en 13 tablas.** No apuntan a ninguna
tabla de esta base: son el alcance (qué clínica, qué proyecto), un valor fijo. A) `erd` reconoce las columnas de
alcance por nombre (constante `erd.ScopeColumns = tenant_id, project_id`) y no las marca ni las dibuja.
B) Cada modelo las marca (`Scope: true` en el campo, campo nuevo en `model`). **Recomiendo A**: es una
convención de todo el ecosistema (`modules.TenantID`, `rbac` usa `project_id` en su clave), B toca 15
módulos para decir lo mismo. Respuesta:

**P13 — referencias «blandas» a propósito salen como avisos.** `device.zone_id` no tiene clave foránea para que
quede vacío (`device_manager/model.go:45`) pero su selector sale de `zone` (`Options: &ZoneModel`); `rbac`
une sus 4 tablas sin claves foráneas (`rbac/models.go:8`).
```go
{Name: "zone_id", Type: input.Select(), Options: &ZoneModel}   // hoy: aviso undeclared_reference + zone aislada
// A: erd dibuja zone ← device con línea punteada, sin aviso      B: igual que hoy (el aviso es correcto)
```
A) `Options: &X` cuenta como relación blanda: línea punteada, sin aviso. `rbac` declara las suyas igual o queda
con aviso. B) Sin cambio. **Recomiendo A**: el modelo ya dice adónde apunta; marcarlo como error enseña a
ignorar los avisos. Respuesta:

**P14 — el tipo de cada columna es el del formulario, no el de la base.** El diagrama dice `is_active checkbox`,
`type radio`, `specialty_id select`, porque dibuja `Type.Name()` (`erd/read.go:148`). A) Dibujar el tipo de
la base (`bool`, `text`, `int`, `real`), y el de formulario solo cuando dice algo de los datos (`email`,
`unix_time`, `nano_time`). B) Solo el de la base. C) Como hoy. **Recomiendo A**: es un diagrama de la base,
pero `unix_time` frente a `nano_time` es justo el error de unidades que hay que ver. Respuesta:

## Decisiones por implementar
| # | Decisión | Dónde se implementa |
|---|---|---|
| D12 (P9, P11) | El repo `ddlc` se archiva: ya nadie lo importa (`app` v0.7.22 tiene el botón) | GitHub (tú, a mano) |

## Fases
| Fase | Repo | Qué | Estado |
|---|---|---|---|
| E1–E7a | `ddl`, `ormc`, `erd`, `icons`, `orm`, `appkit`, `layout`, `app` | `Capture`, `Models`, diagrama + CLI, glifos, `/_schema`, grupo «Developer tools» + pantalla, dibujo al guardar | 🟢 cerradas: ddl v0.0.32, ormc v0.1.27, erd v0.0.4, icons v0.0.38, orm v0.12.19, appkit v0.0.7, layout v0.3.81, app v0.7.22 |
| E7b | GitHub `webtyp/ddlc` | archivar el repo | ⚪ listo (tú) |
| E8 | 13 módulos `veltylabs` | por módulo, cuando cierre su plan de REACTIVE_RECORDS: `go install webtyp.com/erd/cmd/erd@latest && erd` en la raíz; borrar `docs/diagrams/database.md`; en `docs/ARCHITECTURE.md` cambiar su enlace por `![Base de datos](diagrams/database.svg)`; `gopush`. Al terminar: quitar las líneas `STATUS` de **module-layout** y **documentation** | ⚪ listo; mejor tras P12–P14 (si no, cada SVG se regenera dos veces) |
| E9 | `veltylabs/staff_manager` | declarar sus claves foráneas, que hoy no tiene ninguna: `staff_device.staff_id` → `staff_member`, `staff_device.device_id` → `device` (ya importa `device_manager`), `staff_member.user_id` → `user` (ya importa `auth`) | ⚪ escribir plan |
| E10 | `webtyp/erd` | P12–P14 | ⚪ tras tus respuestas |
| Z | `mjosefa-cms` | `docs/PLAN.md`: `appkit.MountServer` con `Migrations: config.MigrationSteps()` (`[]ddl.Step`, se borra `MigrationStep`), test `/_schema` 200 con permiso / 403 sin él. El admin ya lo ve por `*:all` (`config/auth.go:186`). Luego: captura del grupo y del diagrama | ⚪ escrito, sin despachar |

## Bloqueos
- 2026-10-28 · `rbac`, `sqlt`, `goflare` siguen con `ddl` v0.0.15 y `agentmemory` con v0.0.20, por pruebas rojas suyas (`rbac`: `TestMigrateDetectsPreexistingDuplicates`; `goflare`: falta `web/edge/main.go`). No bloquean esta ola.

## Relación con otras olas
- APP_COMPOSITION: Z de aquí es la mitad «contratos» de su B2; `config/server.go` sigue con el transporte.
- REACTIVE_RECORDS: F5 se diseña sobre `appkit` v0.0.7 (ya con `Migrations`).
- MVP mjosefa-cms: la prueba local (fase M2b del roadmap) es donde se ve esto funcionando.
