# Mejoras de arquitectura — master plan
> **Status:** IN PROGRESS · 2026-11-07 · espera respuesta a P1 (contrato de interfaz `crud.Module` / `crud.Registry`) · Diagrama: [diagrams/TRANSPORT_CRUD_ARCHITECTURE.md](diagrams/TRANSPORT_CRUD_ARCHITECTURE.md)

Defectos de diseño entre librerías que aparecen al usarlas (responsabilidades mezcladas, dependencias que sobran, vocabulario inconsistente).
Reglas: **api-design** (puerta 4), **api-harness**, **core-principles** y **module-layout**.
Diagrama de arquitectura de transporte, validación y RBAC: [diagrams/TRANSPORT_CRUD_ARCHITECTURE.md](diagrams/TRANSPORT_CRUD_ARCHITECTURE.md).

## Preguntas abiertas

**P1 — Hacer obligatorio `webtyp/crud` por contrato de interfaz en compilación (sin depender de `gocheck`).** Hoy `appkit`, `rpc`, `mcp` y `loopback` aceptan `router.OperationModule` (`MountOperations(reg router.OperationRegistry)`), lo que entrega a los módulos `reg.Operation(name, h) router.Route` y les permitió crear operaciones ad-hoc con IDOR (`patient_directory/ops.go:83,97,111`) o rutas `.Public()` (`room_layout/ops.go:529`).
- **A) (Recomendado) `crud.Registry` + `crud.Module` + `crud.Route` (eliminando `router.OperationModule`):** `appkit`, `rpc.Mount`, `mcp.HarvestOps` y `loopback` exigen `crud.Module` (`MountOperations(reg crud.Registry)`). `crud.Registry` solo tiene `Create`, `Read`, `Update`, `Delete` (convirtiendo `crud.Mount*` en métodos de `reg`) y devuelve `crud.Route` (solo `RequiresMFA()` y `Describe(string)`, sin `.Public()`, `.Authenticated()` ni `.Requires()`). Un módulo con una o varias tablas registra sus entidades sin poder tocar `Operation(name, h)`, `router.Context` ni `router.Route`:
  ```go
  func (m *Module) MountOperations(reg crud.Registry) {
      reg.Create(func() model.ModelSlice { return &PatientList{} }, func(list model.ModelSlice) error {
          return m.CreatePatients(*list.(*PatientList))
      })
      reg.Read(&PatientModel, func(args crud.ReadArgs) (model.ModelSlice, error) {
          return m.ReadPatients(args.ID, args.Search, args.After, crud.Cap(args.Limit))
      })
      reg.Update(func() model.Model { return &Patient{} }, func(ids, fields []string, rec model.Model) error {
          return m.UpdatePatients(ids, fields, *rec.(*Patient))
      })
  }
  ```
- **B) Interfaces de capacidad por entidad (`crud.Creator`, `crud.Reader`, `crud.Updater`, `crud.Deleter`) sin `MountOperations`:** Cada entidad implementa directamente `Create`/`Read`/`Update`/`Delete`; en módulos multi-tabla (`room_layout`, `service_catalog`, `supply_manager`) obliga a crear un struct wrapper por tabla devuelto en `Parts()`.
**Recomiendo A**: cierra al 100% en tiempo de compilación `reg.Operation`, `router.Context` y `.Public()`/`.Authenticated()`, y soporta módulos de 1 o N tablas sin structs intermedios (análisis detallado de pros, contras y mitigaciones en [diagrams/TRANSPORT_CRUD_ARCHITECTURE.md](diagrams/TRANSPORT_CRUD_ARCHITECTURE.md#3-análisis-de-pros-contras-y-mitigaciones-de-la-propuesta-para-el-framework)). Respuesta:

## Decisiones por implementar

| ID | Decisión | Fase |
|---|---|---|
| D1 | **Contrato de cable en `webtyp.com/crud` (0 UI, 0 genéricos, 0 `op string`, lote transaccional, layout por dominio y validaciones en `validate.go`):** `crud` extrae de `view` el contrato de las 4 operaciones estándar, sus mensajes (`ReadArgs`, `CreateArgs`, `UpdateArgs`, `DeleteArgs`, `Cap`, `MaxPage`), `crud.Op(m, action)` → `"<ModelName>_<action>"`, `.Requires(Resource(ModelName()), action)` automático, validaciones en `validate.go` y lote `model.ModelSlice` para `db.Tx` | A1 (v0.0.2; se amplía con P1) |
| D2 | **Separación semántica estricta entre `List` (Frontend UI) y `Read` (`crud` + Backend):** `List` (`view.Lister`, `List(args crud.ReadArgs, done)`) permanece solo en `webtyp/view`. En `crud` y en el backend de módulos (`ops.go`, `crud.go`, `queries.go`), toda lectura y paginación se unifica bajo **`Read`** (`read.go`, `crud.ReadArgs` con `ID`, `Read<Entity>`), renombrando `List*`/`Get*` → `Read*`, `Save*`/`Upsert*` → `Create*`, `Patch*`/`Deactivate*` → `Update*`, `Remove*` → `Delete*` | A1–A4 |
| D3 | **`view.Ops` sin strings de operación y `view.go` con `//go:build browser`:** `view` elimina `mount.go`, `payload.go` y `ListArgs`/`Cap`/`MaxPage` de `page.go`; `view.Ops` usa `Allow model.Action` y `crud.Op`. `auth/view.go` y `access/view.go` llevan `//go:build browser` | A2, A3 |
| D4 | **`webtyp/crud` obligatorio por contrato de interfaz en compilación, fin de doble validación declarativa y cierre de IDOR/Public:** Los módulos no reciben `router.OperationRegistry`, `router.Context` ni `router.Route`, impidiendo en tiempo de compilación `reg.Operation` crudo, `.Public()` (detectado en `room_layout/ops.go:529`) y `args.TenantId` (IDOR en `patient_directory/ops.go`). `crud/validate.go` es el único punto del servidor que ejecuta `model.ValidateFields`; los módulos solo validan invariantes de dominio | A1–A4 |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| A1 | `webtyp/crud`, `webtyp/router`, `webtyp/rpc`, `webtyp/mcp`, `webtyp/appkit` | `crud` v0.0.2 publicado; tras responder P1, incorporar `crud.Registry`, `crud.Module` y `crud.Route` y eliminar `router.OperationModule` para que solo compile el montaje vía `crud` | espera P1 |
| A2 | `webtyp/view` | borra `mount.go`, `payload.go`, `tests/mount_test.go` y `ListArgs`/`Cap`/`MaxPage` de `page.go`; `Lister.List` usa `crud.ReadArgs`; `view.Ops` usa `Allow model.Action` y `crud.Op` + `crud.*Args` | `docs/PLAN.md` escrito; espera P1 |
| A3 | `webtyp/access`, `webtyp/auth`, `webtyp/layout` | `access` y `auth` migran `ops.go` y `view.go` (`//go:build browser`) a `crud` y renombran métodos backend a `Read*`/`Create*`/`Update*`/`Delete*`; `layout` actualiza `crudview/*_test.go` a `crud.ReadArgs` | esperan A1–A2 |
| A4.1 | `veltylabs/modules/patient_directory` (plantilla) | migra `ops.go`, `crud.go`, `queries.go`, `view.go` al contrato cerrado de `crud`, lote transaccional `db.Tx`, cierra IDOR de `args.TenantId` y elimina doble validación manual de esquema | `docs/PLAN.md` escrito; **se ejecuta en local tras A2** y, una vez aprobado, sirve de plantilla |
| A4.2 | 17 módulos restantes `veltylabs/modules/*` (`room_layout`, `pharmacy_manager`, `service_catalog`, `device_manager`, `staff_manager`, `supply_manager`, `work_schedule`, `business_calendar`, `alert_manager`, `appointment_booking`, `clinical_encounter`, `payment_manager`, `agreement_manager`, `cash_manager`, `chat_room`, `editorial`, `network_manager`) | aplican la plantilla aprobada de `patient_directory` (`crud`, `db.Tx`, renombrado `Create*`/`Read*`/`Update*`/`Delete*`, `//go:build browser` en `view.go`) | se despachan vía `codejob` una vez aprobado A4.1 (`patient_directory`) |

## Bloqueos

- 2026-11-07 · P1 espera al usuario.

## Relación con otras olas

- APP_USABILITY: su fase F (publicar `iam`) espera A1–A3 (`crud`, `view`, `auth`, `access`).
