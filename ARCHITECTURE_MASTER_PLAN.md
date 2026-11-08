# Mejoras de arquitectura — master plan
> **Status:** IN PROGRESS · 2026-11-08 · D1–D18 cerradas; `work_schedule` (`v0.1.58`) y `editorial` (`v0.0.65`) cerradas y publicadas; 15 sesiones en Jules (`appointment_booking` y `room_layout` despachados); `access` espera `auth` · Diagrama: [diagrams/TRANSPORT_CRUD_ARCHITECTURE.md](diagrams/TRANSPORT_CRUD_ARCHITECTURE.md)

Defectos de diseño entre librerías (responsabilidades mezcladas, dependencias que sobran, vocabulario inconsistente).
Reglas: **api-design** (puerta 4), **api-harness**, **core-principles**, **module-layout**, **zero-debt**.
La firma de `crud.Registry` queda fijada por D7–D13; cambiarla después obligaría a migrar los 18 módulos dos veces.

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

| ID | Decisión | Fase |
|---|---|---|
| D1 | **Contrato de cable en `webtyp.com/crud` (0 UI, 0 genéricos, 0 `op string`, lote transaccional):** `crud` extrae de `view` las 4 operaciones estándar y sus mensajes (`ReadArgs`, `CreateArgs`, `UpdateArgs`, `DeleteArgs`, `Cap`, `MaxPage`), `crud.Op(m, action)` → `"<ModelName>_c|_r|_u|_d"` (`crud/op.go:24`), `.Requires(Resource(ModelName()), action)` automático, validaciones en `validate.go` y lote `model.ModelSlice` para `db.Tx` | A1 (v0.0.2 publicada; se amplía con D7–D12) |
| D2 | **`List` solo en la UI, `Read` en `crud` y backend:** `view.Lister.List(args crud.ReadArgs, done)` queda en `webtyp/view`; en módulos (`ops.go`, `crud.go`, `queries.go`) se renombra `List*`/`Get*` → `Read*`, `Save*`/`Upsert*` → `Create*`, `Patch*`/`Deactivate*` → `Update*`, `Remove*` → `Delete*` | A1–A4 |
| D3 | **`view.Ops` sin strings de operación y `view.go` con `//go:build browser`:** `view` elimina `mount.go`, `payload.go` y `ListArgs`/`Cap`/`MaxPage` de `page.go`; `view.Ops` usa `Allow model.Action` y `crud.Op`; `auth/view.go` y `access/view.go` llevan `//go:build browser` | A2, A3 |
| D4 | **Una sola validación de esquema en el servidor:** `crud/validate.go` es el único punto que ejecuta `model.ValidateFields`; los módulos borran los `if` que repiten el `Schema` (`patient_directory/module.go:119-135,168-191`) y solo validan reglas de negocio (`ValidateRUT`, unicidad por tenant, transiciones de estado). El cierre de `reg.Operation`/`.Public()` está en D7 + D8 | A1–A4 |
| D5 | **`crud` deja de depender de `rpc`:** `rpc.WriteError` (`rpc/errors.go:52-71`) pasa a `router.WriteError`, junto a `router.Problem`/`StatusOf`; así `rpc` y `mcp` pueden importar `crud` sin ciclo (`model` → `router` → `crud` → `rpc`/`mcp`/`view`/`appkit`) | A1 |
| D6 | **Rechazar en vez de recortar:** `MountRead` (`crud/read.go:86-94`) recorta en silencio `Search`, `After` e `ID` demasiado largos; pasa a responder `400` (`ErrInvalidID` para `ID`; error nuevo `ErrArgTooLong` para `Search`/`After`) como `validateIDs` (**zero-debt**: sin fallbacks silenciosos). El comentario de `Cap` deja de mostrar al módulo llamándolo | A1 |
| D7 (P1) | **Contrato cerrado `crud.Module` / `crud.Registry` / `crud.Route`:** `appkit`, `rpc.Mount`, `mcp.HarvestOps` y `crud.Loopback` exigen `crud.Module` (`MountOperations(reg crud.Registry)`); `Registry` solo tiene `Create`, `Read`, `Update`, `Delete` (los `crud.Mount*` pasan a métodos); `Route` solo `RequiresMFA`/`Describe`; se elimina `router.OperationModule` y el módulo deja de ver `router.Context`/`router.Route` | A1, A3, A4 |
| D8 (P2) | **`MountAPI` solo por declaración de la app:** `appkit.App` separa `Modules` (su `Build` devuelve `crud.Module`, ya no `model.ModuleNaming` asertado en `appkit/serve.go:58-62`) e `Infra []appkit.Infra` (único lugar con `router.Router`: auth, archivos), escrito en `config/`; un módulo de dominio con `MountAPI` no se monta | A1, A3 |
| D9 (P3) | **`crud.Caller{UserID string}` como primer argumento de las 4 funciones**, rellenado por `crud` desde la sesión ya verificada; reemplaza `ctx.UserID()` en `chat_room/ops.go:96-238` y `payment_manager/ops.go:126`, y elimina `CreatorUserId` del JSON en `appointment_booking` (`model_orm.go:836`) | A1, A4 |
| D10 (P5) | **Lecturas públicas solo por `App.PublicReads []model.ModuleNaming` en `config/`:** `crud` monta esas lecturas como públicas (solo `Read`, misma validación y `Cap`); `room_layout` deja de llamar `.Public()` (`ops.go:529`) y la app de referencia declara `PublicSlotModel` | A1, A4 |
| D11 (P6) | **`model.Field.ServerOnly`:** `crud` rechaza en `Create` y `Update` cualquier campo marcado con `ErrInvalidField` (`balance`, `created_by`, `updated_at`); en `Create` el `tenant_id` también es `ServerOnly`; la UI (`form`) oculta esos campos con la misma marca | A0, A1, A4 |
| D12 (P7) | **`ormc` genera `model_crud.go` (`//go:build !browser`) por entidad:** `Mount<Entity>Create/Read/Update/Delete(reg, fn)` con firmas tipadas (`func(c crud.Caller, list PatientList) error`); el módulo nunca escribe `list.(*PatientList)` | A1, A4 |
| D13 (P4) | **Filtro tipado opcional en `Read`, nunca `storage.Query` por el cable:** `Registry.Read(m, newFilter func() model.Fielder, read func(c Caller, args ReadArgs, f model.Fielder) (model.ModelSlice, error))` (`newFilter` nil = sin filtro); `crud` decodifica el filtro en `ReadArgs` (clave `filter`), lo valida con `model.ValidateFields` y lo entrega; el módulo lo traduce en `queries.go` a `storage.Condition` y agrega siempre `tenant_id`; una sola operación `_r` por entidad; `ormc` genera `Mount<Entity>Read(reg, newFilter, fn)` tipado | A1, A2, A4 |
| D14 (D7) | **Cómo se monta por dentro:** `router.OperationRegistry` sigue existiendo solo como primitiva de transporte (nunca llega a un módulo); `crud.Mount(reg router.OperationRegistry, public []model.ModuleNaming, m crud.Module) error` arma un `crud.Registry` sobre ella y llama `m.MountOperations`; todo el que hoy llama `MountOperations` pasa por `crud.Mount`: `rpc.Routes`/`rpc.Mount`, `mcp.HarvestOps`, `dbsync.harvest` (`dbsync/client.go:359`, `Client.Modules` devuelve `[]crud.Module`) y `crud.Loopback(mods...)`/`crud.LoopbackActingAs(userID, mods...)` (reemplaza `router/loopback.New/WithTenant/ActingAs`, que se borra; sin inyección de tenant: el tenant es configuración del módulo). `auth/authority`: `OpMe` → `Infra` (`MountAPI`); usuarios y LAN → `crud` sobre `auth.User` y la entidad LAN (recurso = `ModelName`, adiós `Requires("users", …)`). `appkit` monta como operaciones solo los valores construidos (y sus `Parts`) que son `crud.Module`, y falla al arrancar (`ErrNotCrudModule`) si un módulo no es `crud.Module` ni `Parts`; `access.Infra` monta las rutas de sesión (logout, `/me`). Recursos que cambian de nombre: `users` → `user`, lista de personas `access_assignment` → `access_person` | A1, A3 |
| D15 (P8) | **La demo de un módulo no escribe chasis:** el módulo declara `seed.Demo() (view.Demo, error)` (`storage/mem` + seed + `Screens`, solo contratos); el demonio `webtyp/app` detecta `seed.Demo` sin `web/browser/main.go` y genera en memoria `func main() { layoutapp.Demo(seed.Demo) }` en un `go.mod` temporal fuera del repo (`replace <módulo> => <ruta>` + la versión de `layout` que fija el demonio); `layoutapp.Demo` monta `crud.LoopbackActingAs` + `BuildMenu` + `platformd`; se borran los `web/browser/main.go` (15 demos, 938 líneas) y las demos rotas de `patient_directory`/`staff_manager` vuelven a mostrar el menú; un `web/browser/main.go` escrito a mano sigue como salida de emergencia | L4, L5 |
| D16 (P9) | **`view` es el único repo de contratos entre módulo y chasis; `layout` solo lo elige la app:** a `view` pasan `Notifier`/`UsesNotifier`/`Auto`/`Persistent`/`For`/`Duration` (de `layout/capabilities.go`; `Badged`, `Described`, `UIModule`… quedan en `layout`: ningún módulo los implementa salvo `chat_room`, ver L6), `BrowserDeps` (sin `Brand`, ver D17), `ModuleBrowser`, `EntryScreens`, `Demo`; la fuente de chat plana (reemplazo de `chatview.Source`) va en L6; `view` conserva `NewCallerLister`; `appkit.Browser` (`appkit/browser.go`) → `layout/app`, y `appkit` deja de requerir `layout`; el `ui/` de un módulo declara datos y el chasis elige `crudview`/`chatview`/`rightpanel` (se borra la excepción de **app-modules**); una pantalla a medida puede usar `components` | L1–L3, L5, L6 |
| D17 (P10) | **La pantalla de ingreso la dibuja el chasis:** `access` entrega `view.EntryScreens{Login, Setup view.Entry; IdleSeconds int}` con `view.Entry{Subtitle lang.Text; Form *dom.Element; Message *dom.SignalString}` (solo formulario y mensaje); `layout/app.Browser` los envuelve con `layout/login` + `App.Brand`; `access` deja de importar `layout/login` (`access/module/browser.go:15`) y de leer `d.Brand` | L1, L3, A3b |
| D18 (P11) | **Un permiso distinto = un modelo propio, nunca una operación con nombre:** si una acción hoy exige un recurso distinto al de su entidad, pasa a ser una operación CRUD sobre un modelo cuyo `ModelName` es ese recurso. Transición de revisor → **crear** un registro (`editorial`: enviar/retirar = `post_u` sobre `state`; aprobar/pedir cambios = `post_transition_c` con `to_state`, `reason`; `actor_id`, `from_state`, `at` `ServerOnly`; el módulo valida la transición y cambia `Post.state` en la misma `db.Tx`; `ResourcePostReview` se borra). Reporte o lectura derivada → `Read` de un modelo de cable con su propio nombre (`alert_manager` `Health`/`Summary`, `cash_manager` funciones). RBAC concede cada recurso por separado | A4.2 (respondido a las sesiones en curso), plantilla de los módulos siguientes |

## Fases

**Mensaje a Antigravity — reparto:** ejecuta **local** solo lo pequeño de `webtyp` (L1 `view`, L2 `appkit`, L4a `server`) y **publica cada uno apenas pasa `gotest`** (`gopush`, de abajo hacia arriba); **nunca `replace`**. **Despacha** (`codejob dispatch`, `EXECUTOR: jules`) todo módulo y todo repo de trabajo grande, en oleadas para llenar los **15 cupos simultáneos** (límite **100 trabajos/día**): un plan despachado no ve carpetas hermanas, por eso cada oleada sale solo cuando lo que necesita está publicado. Al despachar, pregunta al dueño por el temporizador de revisión (**codejob**) y revisa cada PR con **plan-review**.
- **Oleada 0 (local, publicando):** L1 → L2 (L4a va después de que se publique L3).
- **Oleada 1 y 2 (15 cupos concurrentes en Jules llenos al máximo):**
  1. `webtyp/auth` (sesión 8600479642742704008)
  2. `webtyp/layout` (sesión 6353811879617294185)
  3. `veltylabs/modules/patient_directory` (sesión 14656212430721096454)
  4. `veltylabs/modules/staff_manager` (sesión 15975789775639801906, reply enviado)
  5. `veltylabs/modules/service_catalog` (sesión 3199782381933916853, reply: gotest pasa, abriendo PR)
  6. `veltylabs/modules/device_manager` (sesión 4954513444987103874)
  7. `veltylabs/modules/business_calendar` (sesión 14857109302422191573)
  8. `veltylabs/modules/supply_manager` (sesión 15140373304869652996, reply D18 enviado)
  9. `veltylabs/modules/pharmacy_manager` (sesión 8674256041346153039, reply enviado)
  10. `veltylabs/modules/agreement_manager` (sesión 18370178096779508356, reply enviado)
  11. `veltylabs/modules/alert_manager` (sesión 14672456625882336877, reply Caller struct enviado)
  12. `veltylabs/modules/cash_manager` (sesión 14578098458978629659, reply D1 crud.Op enviado)
  13. `veltylabs/modules/room_layout` (sesión 15931334370260317991)
  14. `veltylabs/modules/appointment_booking` (sesión 5077271452224787563, despachado al cerrarse `work_schedule`)
  15. `webtyp/core` (`app`, sesión 5351990320140399274, cerrada localmente como `v0.8.3`; cuenta en Jules hasta que termine ciclo)
  - **Módulos cerrados y publicados**: `work_schedule` (`v0.1.58`), `editorial` (`v0.0.65`).
  - A medida que se liberen cupos al completar PRs, se despachan los restantes (`clinical_encounter` [plan listo], `chat_room`, `payment_manager`, `network_manager`, `site_content`, `site_manager` y `access`).
- **Oleada 3:** L6 tras P12; A5 al final.

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| A0 | `webtyp/model` | marca `ServerOnly` (D11) | cerrada `v0.2.14` |
| A1 | `webtyp/router`, `crud`, `rpc`, `mcp`, `ormc`, `dbsync`, `appkit` | D5–D14 | cerrada: `router` `v0.4.19`, `crud` `v0.0.3`, `rpc` `v0.2.17`, `mcp` `v0.3.23`, `ormc` `v0.1.36`, `dbsync` `v0.3.47`, `appkit` `v0.2.27` |
| A2 | `webtyp/view` | contrato de cable desde `crud`, `view.Ops` con `model.Action` | cerrada `v0.6.66` |
| A1b | `webtyp/ormc` | bug: `crud_gen.go` omitía `Mount*` en entidades con campos `input.*` (casi todas); fixture real en `tests/testdata/fixturecrud` | cerrada `v0.1.37` |
| A3 | `webtyp/auth` | `auth/authority` en `crud`, `OpMe` → `Infra`; D15–D17 no lo tocan | en Jules (sesión 8600479642742704008, trabajando `IN_PROGRESS`) |
| L1 | `webtyp/view` | contratos módulo↔chasis (D16, D17): `Notifier`…, `BrowserDeps`, `ModuleBrowser`, `EntryScreens`, `Entry`, `Demo` | cerrada `v0.6.67` |
| L2 | `webtyp/appkit` | borra `browser.go` y los tipos movidos; `Screens`/`Entry` sobre `view`; `go.mod` sin `layout` | cerrada `v0.2.28` |
| L3 | `webtyp/layout` | dobles de test en `crud.ReadArgs` (ex A3), contratos desde `view`, `LocalTools` sobre `crud.Module`, `layout/app.Browser` (de `appkit`) y `layout/app.Demo`, pantallas de ingreso con `App.Brand` | despachado · Jules (sesión 6353811879617294185, oleada 1) |
| A3b | `webtyp/access` | `crud` (D7–D14) + pantallas de ingreso como datos (D17) + `seed.Demo` (D15) | plan listo · despacho · oleada 1, tras `auth` publicado |
| L4a | `webtyp/server` | main de navegador generado con `layoutapp.Browser` + `layout` en `generatedLibraries` | plan listo · local · tras L3 publicado |
| L4b | `webtyp/app` | el demonio genera la demo desde `seed.Demo` en un módulo temporal fuera del repo | cerrada `v0.8.3` |
| A4.1 | `veltylabs/modules/patient_directory` (plantilla) | contrato `crud` + `seed.Demo` + `module/*` sobre `view` | despachado · Jules (sesión 14656212430721096454, oleada 1); el dueño revisa la PR como plantilla |
| A4.2+L5 | 17 módulos `veltylabs/modules/*` + `veltylabs/site_manager`, `site_content` | un plan por repo copiado de la plantilla A4.1 aprobada (su diff como referencia en `_temp/`); `appointment_booking` sin `CreatorUserId` (D9); `room_layout` con `PublicReads` (D10) y `view.Notifier`; su demo pasa a `seed.Demo` | oleada 2: 19 planes copiados de la plantilla aprobada; despacho vía `codejob` |
| L6 | `webtyp/view`, `webtyp/layout`, `chat_room`, `clinical_encounter`, `appointment_booking` | el `ui/` de estos 3 aún arma `crudview`/`chatview`/`rightpanel` y `chat_room` implementa `Badged` con `countbadge`: diseño pendiente (fuente de chat plana, contador, lista a medida de `clinical_encounter/ui/browser.go:100`); abrir P12 con código antes de su plan; en A4.2 esos 3 conservan esas importaciones | pendiente |
| A5 | `mjosefa-cms` (app de referencia) | corre con el contrato cerrado; capturas de cada pantalla revisadas; test: admin y usuario sin permisos recorren el menú, ningún enlace abre pantalla vacía; `web/browser/main.go` regenerado con `layoutapp.Browser` | espera A4.2, L4 |

## Bloqueos

- 2026-11-07 · A3b (`access`) espera `auth` publicado (Jules).
- 2026-11-07 · Oleada 2 espera la aprobación del dueño de la PR de A4.1.

## Relación con otras olas

- APP_USABILITY: su fase F (publicar `iam`) espera A1–A3 (`crud`, `view`, `auth`, `access`).
