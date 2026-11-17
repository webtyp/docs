# Módulos configurables, fases 1–2 (`model`, `rbac`, `access`, `crud`, `view`, `appkit`) — plan

> Read `AGENTS.md` at each repo root before any change.
> Implementa D1–D6 de [CONFIGURABLE_MODULES_MASTER_PLAN.md](../CONFIGURABLE_MODULES_MASTER_PLAN.md).
> Se ejecuta **en local** por [EXECUTION_ORDER paso 20](../EXECUTION_ORDER_MASTER_PLAN.md) (reglas de `replace`, `gotest` y puerta 23).
> Maquetas de referencia: [view-as-2.png](../../devtools/docs/img/view-as-2.png) (D3) y [d7-client-form.png](../../../veltylabs/modules/appointment_booking/docs/img/d7-client-form.png) (D7).

## Tools

```bash
go install webtyp.com/devflow/cmd/gotest@latest
go install webtyp.com/lang/cmd/langc@latest
export PATH="$PATH:$(go env GOPATH)/bin"
```

## Ecosystem search

Se buscó `Industry`, `ModuleUse`, `HiddenFromRecordOwner`, `RecordLink`, `LoopbackActingAs` en `webtyp/*` y `veltylabs/*`:
- `webtyp/model`: ya define `ModuleDefinition`, `Field`, `RoleCode`, `Grant`, `Action`, `Resource` (**extender** allí; `model` solo importa `webtyp.com/fmt`, no puede importar `lang` ni `svg`).
- `webtyp/rbac`: ya administra `Role`, `Permission`, `EnsureGrants` (**extender** con `GrantsForUse(def, use)`).
- `webtyp/access`: ya administra `RoleRecordModel`, `AssignmentModel`, `EnsureRole` (**extender** con la entidad `access_record_link` para el vínculo cuenta → ficha de D2).
- `webtyp/crud`: ya define `CurrentUser`, `LoopbackActingAs`, `ServerOnly` (**extender** con `ModuleUse` en `CurrentUser`, `LoopbackActingAsUse` y filtrado de `HiddenFromRecordOwner`).
- `webtyp/view`: ya define `BrowserDeps`, `Screen` (**extender** con `Use model.ModuleUse` y `LinkedRecordIDs []string`).
- `webtyp/appkit`: ya valida módulos y asegura `DefaultGrants` en `backend.Build` (**extender** con `App.Industry`, `App.Roles` y validación de rubro).

## Design gate

1. **Prior art:**
   - Django (`permissions` + `ModelAdmin.get_queryset`, https://docs.djangoproject.com/en/stable/topics/auth/default/): permisos por modelo/acción y filtrado por dueño en la vista. Aquí la manera de uso (`ModuleUse`) es un contrato cerrado en `model`, no código ad-hoc por vista.
   - Rails + Pundit (scopes por rol, https://github.com/varvet/pundit): políticas por registro en Ruby. Aquí `rbac` traduce la `ModuleUse` a `model.Grant` automáticamente.
   - Shopify App Bridge / OAuth scopes (https://shopify.dev/docs/api/admin-graphql): capacidades declaradas arriba. Aquí el módulo declara sus rubros e íconos en `model.ModuleDefinition`.
2. **Prueba del novato:**
   - `model.Industry` (`IndustryHumanHealth`, `IndustryAnimalCare`, `IndustryEducation`, `IndustryGeneralBusiness`) + `model.AnyIndustry()`.
   - `model.ModuleUse` (`ManagesEverything`, `WorksWithAllRecords`, `SeesOwnRecords`, `VisitsWithoutAccount`): 2–3 palabras literales, verbo + objeto.
   - `model.Role`, `model.ModuleUseOverride`, `model.IndustryIcon`, `model.Field.HiddenFromRecordOwner`.
   - `rbac.GrantsForUse(def, use)`.
   - `access.Module.LinkRecord(userID, module, recordID)` / `LinkedRecords(userID, module)`.
   - `crud.LoopbackActingAsUse(pub, userID, use, ownedIDs, mods...)`.
3. **Complexity ledger:**
   - Conceptos nuevos: +2 (`model.Industry`, `model.ModuleUse`).
   - Archivos por módulo consumidor: 0 nuevos (se declaran campos en `Definition` y `Field`).
   - Líneas en la app: +2 (`App.Industry`, `App.Roles`).
   - Formas de hacer lo mismo: -1 (reemplaza permisos escritos a mano por rol por `GrantsForUse`).
4. **¿Dónde vive?** Tipos de contrato en `model` (cero dependencias); traducción a permisos en `rbac`; vínculo cuenta → ficha en `access`; filtrado/contexto de operación en `crud`; contexto de pantalla en `view`; validación y wiring de app en `appkit`. Ningún módulo importa `appkit`.
5. **Qué borra:** la necesidad de escribir `Grants` a mano para cada rol estándar de la app y el buscador para usuarios con `SeesOwnRecords`.

## APIs used

| Símbolo | Firma / Forma | Repo / Archivo |
|---|---|---|
| `model.ModuleDefinition` | `type ModuleDefinition struct { Name string; Entities []Definition; … }` | `webtyp/model/module_definition.go` |
| `model.Field` | `type Field struct { … ServerOnly bool; Permitted }` | `webtyp/model/field.go` |
| `model.Grant`, `model.RoleCode` | `type RoleCode string`, `type Grant struct { Resource; Actions }` | `webtyp/model/rbac.go` |
| `rbac.Service.EnsureGrants` | `func (m *Service) EnsureGrants(projectID string, role model.RoleCode, grants []model.Grant) error` | `webtyp/rbac/rbac.go` |
| `access.Module.EnsureRole` | `func (m *Module) EnsureRole(code model.RoleCode, name, description string, emails []string) error` | `webtyp/access/module.go` |
| `crud.CurrentUser` | `type CurrentUser struct { UserID string }` | `webtyp/crud/contract.go` |
| `view.BrowserDeps` | `type BrowserDeps struct { Caller router.Caller; Zone time.Zone; IDs model.IDGenerator; Countries country.Catalog }` | `webtyp/view/browser.go` |
| `appkit.App` | `type App struct { Name, TenantID, Timezone string; Countries country.Catalog; SignIn SignInMethod; Modules []model.ModuleDefinition; Log func(...any) }` | `webtyp/appkit/app.go` |

Si una firma real difiere de esta tabla, detenerse y revisar el código antes de adaptar.

---

## Etapa 1 — `webtyp/model` (D1, D2, D3, D4, D5, D6)

1. Crear `model/industry.go`:
   - `type Industry string` y las 4 constantes:
     `IndustryHumanHealth Industry = "human_health"`, `IndustryAnimalCare Industry = "animal_care"`, `IndustryEducation Industry = "education"`, `IndustryGeneralBusiness Industry = "general_business"`.
   - `func AnyIndustry() []Industry` devuelve una copia nueva con los 4 rubros.
   - `type IndustryIcon struct { Industry Industry; Icon string }` (nombre de símbolo SVG; `model` no importa `svg`).
   - `func SupportsIndustry(def ModuleDefinition, ind Industry) bool`: verdadero si `ind == ""` o `len(def.Industries) == 0` o `ind` está en `def.Industries`.
   - `type ModuleUse uint8` con `ManagesEverything`, `WorksWithAllRecords`, `SeesOwnRecords`, `VisitsWithoutAccount` (empezando en `1` con `iota + 1` para que `0` signifique sin declarar).
   - `type ModuleUseOverride struct { Module string; Use ModuleUse }`.
   - `type Role struct { Code RoleCode; Name string; Use ModuleUse; Overrides []ModuleUseOverride }` y `func (r Role) UseFor(module string) ModuleUse` (devuelve el override si existe para `module`, si no `r.Use`).
2. `model/module_definition.go`: agregar a `ModuleDefinition`:
   - `Industries []Industry`
   - `Icon string`
   - `IconFor []IndustryIcon`
   - `Uses []ModuleUse`
   - Agregar `func (d ModuleDefinition) IconForIndustry(ind Industry) string` (devuelve el `Icon` de `IconFor` que coincida con `ind`, si no `d.Icon`).
3. `model/field.go`: agregar a `Field`:
   - `HiddenFromRecordOwner bool` (comentario: campos internos ocultos para quien usa el módulo en `SeesOwnRecords`, ej. notas internas o estado activo).
4. Pruebas en `model/tests/configurable_test.go` + filas en `model/docs/TESTS.md`:
   - `TestConfigurable_AnyIndustryReturnsFreshSlice`
   - `TestConfigurable_SupportsIndustry`
   - `TestConfigurable_IconForIndustryFallsBackToDefault`
   - `TestConfigurable_RoleUseForAppliesOverride`
5. `GOWORK=off gotest` verde en `model`.

## Etapa 2 — `webtyp/rbac` (D3)

1. `replace webtyp.com/model => ../model` en `rbac/go.mod`.
2. Crear `rbac/uses.go`:
   - `func GrantsForUse(def model.ModuleDefinition, use model.ModuleUse) []model.Grant`:
     - `model.ManagesEverything`: `model.AllActions` sobre cada entidad de `def.Entities` con `Allow != 0`.
     - `model.WorksWithAllRecords`: `e.Allow & (model.Create | model.Read | model.Update)` sobre cada entidad con resultado != 0.
     - `model.SeesOwnRecords`: `e.Allow & (model.Create | model.Read | model.Update)` sobre cada entidad con resultado != 0 (el alcance «solo lo suyo» lo aplica `crud` + `access`).
     - `model.VisitsWithoutAccount`: `e.Allow & model.Read` sobre cada entidad con resultado != 0.
     - cualquier otro valor (`0`): devuelve `nil`.
3. Pruebas en `rbac/tests/uses_test.go` + filas en `rbac/docs/TESTS.md`:
   - `TestUses_GrantsForUseCoversEachMode`
4. `GOWORK=off gotest` verde en `rbac`.

## Etapa 3 — `webtyp/access` (D2)

1. `replace webtyp.com/model => ../model` y `webtyp.com/rbac => ../rbac` (si lo usa) en `access/go.mod`.
2. `access/definition.go`: agregar `RecordLinkModel` (`Name: "access_record_link"`):
   - Campos: `id` (PK, `ServerOnly: true`), `user_id` (`model.Text()`, `NotNull: true`), `module` (`model.Text()`, `NotNull: true`), `record_id` (`model.Text()`, `NotNull: true`), `created_at` (`model.NanoTime()`, `ServerOnly: true`).
   - `Allow: model.Create | model.Read | model.Delete`, `Rules: model.Rules{Create: model.HaveNoBusinessRule, Read: model.HaveNoBusinessRule, Delete: model.HaveNoBusinessRule}`.
   - Incluir `RecordLinkModel` en `Definition.Entities`.
3. Regenerar `definition_gen.go` / actualizar `access/migrate` (y `access/crud.go`) para crear la tabla `access_record_link`.
4. `access/record_links.go`: métodos en `*Module`:
   - `func (m *Module) LinkRecord(userID, module, recordID string) error` (idempotente: no duplica si ya existe la tupla `(userID, module, recordID)`).
   - `func (m *Module) LinkedRecords(userID, module string) ([]string, error)` (devuelve todos los `record_id` del usuario para ese módulo; una cuenta puede tener varios).
5. Pruebas en `access/tests/record_link_test.go` + filas en `access/docs/TESTS.md`:
   - `TestRecordLink_LinksMultipleRecordsIdempotently`
   - `TestRecordLink_IsolatesByUserAndModule`
6. `GOWORK=off gotest` verde en `access`.

## Etapa 4 — `webtyp/crud` (D2, D3)

1. `replace webtyp.com/model => ../model` en `crud/go.mod`.
2. `crud/contract.go`: agregar a `CurrentUser`:
   - `Use model.ModuleUse`
   - `OwnedRecordIDs []string`
3. `crud/loopback.go`: agregar `LoopbackActingAsUse(pub events.Publisher, userID string, use model.ModuleUse, ownedIDs []string, mods ...Module) router.Caller` (mantiene `LoopbackActingAs` delegando con `use = 0`).
4. `crud/stored_entity_read.go`, `crud/stored_entity_create.go`, `crud/stored_entity_update.go`:
   - Cuando `c.Use == model.SeesOwnRecords`, los campos marcados `HiddenFromRecordOwner: true` no se devuelven en lectura ni se aceptan en creación/actualización (mismo trato que `ServerOnly` en escritura).
5. Pruebas en `crud/tests/record_owner_test.go` + filas en `crud/docs/TESTS.md`:
   - `TestRecordOwner_HiddenFieldsOmittedForSeesOwnRecords`
   - `TestRecordOwner_LoopbackActingAsUsePassesUseAndOwnedIDs`
6. `GOWORK=off gotest` verde en `crud`.

## Etapa 5 — `webtyp/view` (D2, D3)

1. `replace webtyp.com/model => ../model` en `view/go.mod`.
2. `view/browser.go`: agregar a `BrowserDeps`:
   - `Use model.ModuleUse`
   - `LinkedRecordIDs []string`
3. `view/fields.go`: `func VisibleFields(fields []model.Field, use model.ModuleUse) []model.Field` — excluye los campos con `HiddenFromRecordOwner: true` cuando `use == model.SeesOwnRecords`.
4. Pruebas en `view/tests/fields_test.go` + filas en `view/docs/TESTS.md`:
   - `TestFields_VisibleFieldsHidesInternalForRecordOwner`
5. `GOWORK=off gotest` verde en `view`.

## Etapa 6 — `webtyp/appkit` (D1, D3, D4, D5, D6)

1. `replace` en `appkit/go.mod` hacia `../model`, `../rbac`, `../access`, `../crud`, `../view`.
2. `appkit/app.go`: agregar a `App`:
   - `Industry model.Industry`
   - `Roles []model.Role`
3. `appkit/backend/build.go`:
   - Error nuevo `ErrUnsupportedIndustry` cuando `app.Industry != ""` y `!model.SupportsIndustry(mod, app.Industry)`, nombrando módulo y rubro (`"<module> does not support industry <industry>"`).
   - En `entry(...)`, si `len(app.Roles) > 0` y `app.SignIn != 0`, asegurar cada rol en `access` (`EnsureRole`) y sus permisos en `rbac` usando `rbac.GrantsForUse(mod, role.UseFor(mod.Name))` para cada módulo de `App.Modules`.
4. `langc sync .` en `appkit` si se agregan claves nuevas de error/UI.
5. Pruebas en `appkit/tests/configurable_test.go` + filas en `appkit/docs/TESTS.md`:
   - `TestBuild_RejectsUnsupportedIndustry`
   - `TestBuild_EnsuresAppRolesAndModuleUseGrants`
6. `GOWORK=off gotest` verde en `appkit`.

## Entrega

| Etapa | Repo | Qué | Verificación |
|---|---|---|---|
| 1 | `webtyp/model` | `Industry`, `AnyIndustry`, `ModuleUse`, `Role`, `Icon`, `HiddenFromRecordOwner` | `gotest` |
| 2 | `webtyp/rbac` | `GrantsForUse` | `gotest` |
| 3 | `webtyp/access` | `access_record_link`, `LinkRecord`, `LinkedRecords` | `gotest` |
| 4 | `webtyp/crud` | `CurrentUser.Use`, `LoopbackActingAsUse`, filtrado `HiddenFromRecordOwner` | `gotest` |
| 5 | `webtyp/view` | `BrowserDeps.Use`, `LinkedRecordIDs`, `VisibleFields` | `gotest` |
| 6 | `webtyp/appkit` | `App.Industry`, `App.Roles`, validación de rubro y permisos por uso | `gotest` |
