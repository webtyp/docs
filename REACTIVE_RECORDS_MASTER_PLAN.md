# Lo que se guarda se ve en todas partes: registros reactivos — master plan
> **Status:** IN PROGRESS · 2026-10-27 · **corrección D11 en curso**: G1 (`model` → `ormc` → `view`) la ejecuta Antigravity en local; luego despacha G2 y G3 y manda la corrección a los planes en vuelo (G4). F5 se escribe tras `dbsync` y G1

**Disparador (usuario, 2026-10-27):** al crear un registro aparece primero en la lista, pero al **editarlo** la fila
no cambia aunque salga «Guardado con éxito»; lo que depende de ese registro (las opciones de un selector) tampoco.
**Regla del dueño:** lo que se edita queda ligado campo a campo a lo que se muestra, de forma nativa, fácil para un
dev junior; ruptura sin retrocompatibilidad. Precedente propio: pa100t repartía cada resultado CRUD por websocket a
todos los clientes con nivel de acceso a ese recurso (`pa100tv5/v1/.../wsocket2/respond-broadcast.go`); `crudp`
(`~/Dev/Archive/crudp`) quiso ser ese protocolo: el mismo paquete (acción c/r/u/d + registro) en front y back.

**Causa (2026-10-27):** cada pantalla y cada selector guarda su propia copia de los registros
(`view/presenter.go:26-90`); los selectores se llenan una sola vez desde otra copia (`layout/app/menu.go:100-125`);
la fila sale de `Item()`, una función que el framework no puede leer; y el módulo repite a mano de qué lista sale
un selector (`view.Lookup` en `device_manager/ui/browser.go:39`) aunque el modelo podría decirlo. Ningún test prueba
«edito → la fila cambia». En la app, Equipos no guarda (400 sin decir el campo).

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 | **Lo que se guarda se ve en todo lugar que lo muestra**, sin recargar a mano ni código del módulo; ruptura sin retrocompatibilidad; misma pasada que M de APP_COMPOSITION. Regla ya escrita en **ui-consistency** | todas |
| D2 | **Test de comportamiento obligatorio**: «edito un campo válido → la fila, los selectores y todo lo que lo muestra cambian» (**testing**) | F0, Z |
| D3 (P1) | **Una sola copia de cada registro en el navegador** (modelo + id), compartida por listas, selectores y formularios; reemplaza el `index` por pantalla de `view` y la recarga completa tras guardar (`crudview.go:755-772`) | F2, F3 |
| D4 (P2, P7, D11) | **El enlace campo → ítem de lista es obligatorio y lo escribe `ormc`**: el módulo marca el campo con `Item: model.ItemLabel` / `ItemDescription` (varios se unen con `model.JoinItem`, « · », en orden de campo) / `ItemBadge`, y declara de qué lista sale un selector (`Ref: &M` con llave foránea, `Options: &M` sin ella). `ormc` genera `func (m *X) Item() model.Item` tipado (lee los campos directo; una fecha va en segundos) y rechaza lo inválido con `model.ValidateItem`. Agregar un dato a la lista = una línea en ese campo | G1–G5 |
| D11 | **Corrección de D4 (2026-10-27, pedida por el usuario)**: la primera versión publicó dos tipos para una idea (`model.Row` y `view.Item` + `itemOf`), un tercer vocabulario (`RowTitle`) y una proyección en ejecución (`model.RowOf`) que `view` repetía O(N²) por lista solo para leer ids (`view/records.go:37,40,59,70`, `presenter.go:56,85`). Queda **un** tipo, `model.Item`; `view.Record` = `model.Model` + `model.Itemizer` (un modelo sin `Item()` generado no compila como registro listado); `view.Records` calcula cada ítem **una vez** al guardar. Se borran `model.Row`, `RowSlot`, `RowOf`, `ValidateRow`, `Field.Row`, `view.Item`, `itemOf` | G1–G5 |
| D5 (P3) | **La fila cambia cuando el guardado se confirma**: guardar devuelve el registro guardado y el almacén lo aplica. **Offline igual, mismo código**: con `dbsync`, `client.Caller()` responde desde la base local por la misma interfaz `router.Caller`, y su pull trae los cambios del servidor por la misma puerta que el empuje (D7) | F2, F4 |
| D6 (P4) | **Una fila muestra solo campos de su propio registro** (no hay caso de uso para datos de otro módulo en la fila); los selectores sí son vivos por D3. Una copia congelada a propósito (`PatientNameSnapshot`) es un campo normal y no cambia nunca | F2 | `view` | `view.Records` (copia única), `Lister.Records`, `Presenter.OnChange`, `Item.Badge`, fila con `model.RowOf`, guardar pone la respuesta; borra `Itemizer`, `Lookup`, `Lead*` | 📝 `view/docs/PLAN_REACTIVE.md` (v0.6.47): pasa a `docs/PLAN.md` cuando cierre R0c (v0.6.46); necesita model v0.2.7 |
| D7 (P5, P8) | **El cambio de un usuario llega a todos los que lo están viendo**, sin código en el módulo (la idea de `crudp` y de pa100t). Transporte: **SSE** (`webtyp/sse`); las escrituras siguen por rpc. Cómo se capta y filtra: D9 | F2b, F5 |
| D9 (P9) | **El aviso a los demás sale del `changelog`**, como en `dbsync`: el servidor anota cada escritura en la capa de base de datos (sin código en el módulo, ve también semillas y migraciones); por SSE solo avisa «hay cambios hasta la versión N»; el navegador pide `/pull`, que ya filtra por permiso de lectura y por fila (`Rule.Resource`, `Rule.Scope`), y aplica las filas al almacén. **Un solo camino online y offline.** `appkit` arma las reglas desde los modelos y sus recursos; la app no escribe `config/sync.go` Hoy el tenant es de la instalación (`mjosefa-cms/modules/tenant.go:7`; las operaciones caen a él, `device_manager/ops.go:67-69`): un servidor = un tenant; si algún día un servidor atiende varios, falta el tenant de la sesión en `auth`, y eso toca a todo | F2b, F5 |
| D10 (P10) | **`appkit` centraliza la sincronización; la app elige la base de datos.** La app le pasa a `appkit` su conexión (el puerto `storage`, sobre el motor que ella eligió: postgres, sqlite, indexdb, `storage/mem`); `appkit` la envuelve en el `changelog`, monta el servidor de `dbsync` con reglas armadas desde los modelos y sus recursos, y en el navegador arma `view.NewRecords()` y el `dbsync.Client`. `appkit` puede usar los contratos agnósticos (`storage`, `orm`, `changelog`) pero **nunca importa un motor concreto**. Corrige la lectura de D14: «sin `orm`» significaba «`appkit` no abre ni elige la base», no «no usa el contrato». Regla escrita en la skill **project-layout** | F5 | `appkit`, `mjosefa-cms` | `appkit` recibe la conexión de la app y arma `changelog` + servidor `dbsync` (reglas desde los modelos) + en el navegador `view.NewRecords()` y `dbsync.Client`; `mjosefa-cms` solo le pasa su conexión (D10) | ⚪ se escribe tras F2b (necesita la API publicada de `dbsync` v0.3.11) |
| G1 | `model` → `ormc` → `view` (**puertas, ejecución local de Antigravity**, en ese orden) | D11: `model.Item`, `ItemSlot`, `JoinItem`, `ValidateItem` (v0.2.8); `ormc` genera `Item()` (v0.1.26); `view.Record`, ítems calculados una vez, sin `view.Item` (v0.6.48) | 📝 `docs/PLAN.md` en los tres |
| G2 | `components` (v0.8.57) → `layout` (v0.3.79) | alias `Item = model.Item`; crudview y menú con `model.Item`/`view.Record` | 📝 `docs/PLAN.md`; despachar tras G1 (layout tras components) |
| G3 | 13 módulos publicados con `Row:` (alert, business_calendar, device, editorial, network, patient_directory, payment, pharmacy, room_layout, staff, supply, work_schedule, cash) + `webtyp/auth` | `Row:` → `Item:`, `ormc` regenera, `RowOf` → `.Item()` | 📝 `docs/PLAN.md` en cada uno; despachar tras G1 (y G2 si importan `components`/`layout`) |
| G4 | en vuelo: `service_catalog` (review), `appointment_booking`, `chat_room`, `clinical_encounter`, `access`, `dbsync` | la misma corrección, por `codejob reply` tras G1 (`dbsync`: `Records.Put` recibe `view.Record`) | ⚪ tras G1 |
| G5 | `mjosefa-cms` | se adapta en F5 | ⚪ con F5 |
| D8 (P6) | **El almacén vive en `webtyp/view`** (`view.NewRecords()` + `records.Caller(...)`), y `appkit` lo arma una vez por página; los módulos no cambian su firma | F2, F5 |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| F0 | `layout/crudview` | test de D2 con backend que responde después; pasa hoy (crudview sí actualiza la fila; tu caso no se reprodujo: Equipos 400, Catálogo en blanco) | 📝 en `layout/docs/PLAN.md` (R0b) |
| F1a | `model` | `Field.Row`, `Field.Options`, `RowOf`, `ValidateRow` | 🟢 cerrada v0.2.7 |
| F1b | `ormc` | rechaza filas inválidas al generar | 🟢 cerrada v0.1.25 |
| F2 | `view` | almacén compartido (D3, D8), fila con `model.RowOf`, selectores desde `Ref`/`Options`, guardar devuelve el registro (D5); borra `Item()`, `Itemizer`, `Lookup` | 🟢 cerrada v0.6.47 |
| F2b | `dbsync`, `sse` | SSE «el head se movió» desde `changelog.OnChange`; el cliente hace pull y pone las filas en `view.Records` (`ClientConfig.Records` obligatorio) | [Antigravity · 2026-10-27 18:44] despachado (v0.3.11) |
| F3 | `components`, `layout` | widgets leen `Item.Badge` (fecha/hora traducidas); crudview sigue `OnChange` sin recargar; selectores desde `Ref`/`Options` | 🟢 cerradas: components v0.8.56, layout v0.3.78 |
| F4 | 16 módulos con `Item()` + `webtyp/access` + `webtyp/auth` (+ `cash_manager` y los 3 de M) | `Row:`/`Options:` en `model.go`, borra `Item()` y `Lookups`, guardar responde el registro; los 3 de M también `Screens(...)` con `Component` | [Antigravity · 2026-10-27 23:33] 15 módulos + auth cerrados (cash_manager v0.2.7, staff_manager v0.1.154, supply_manager v0.2.35, alert_manager v0.1.17, auth v0.0.82, network_manager v0.1.85, business_calendar v0.5.17, payment_manager v0.4.16, patient_directory v0.1.63, device_manager v0.2.73, editorial v0.0.35, pharmacy_manager v0.1.37, room_layout v0.3.50, service_catalog v0.6.12, work_schedule v0.1.30); 4 en vuelo (`access`, `appointment_booking`, `chat_room`, `clinical_encounter`) |
| F5 | `appkit`, `webtyp/server`, `mjosefa-cms` | `appkit` arma `view.NewRecords()` y el `dbsync.Client` del navegador; el servidor de sync según P10; `mjosefa-cms` no escribe nada | ⚪ espera P10; se escribe tras F2b |
| Z | `mjosefa-cms` | dos sesiones: una edita un servicio y una especialidad; en ambas cambian la lista y el selector de Servicios; capturas | ⚪ al final |

## Bloqueos

- 2026-10-27 · **Revisión: la coordina Antigravity.** Este chat se cerró; orden de despacho (cada paso cuando sus tags estén publicados): model v0.2.7 (ya en vuelo) → `ormc`; view v0.6.46 (en vuelo) → `view/PLAN_REACTIVE.md` (v0.6.47) → `components` (v0.8.56) y `dbsync` (v0.3.11) → `layout/PLAN_REACTIVE.md` (v0.3.78) → F4 en todos los módulos, `access` y `auth` → F5. Un plan aparcado (`PLAN_REACTIVE.md`) se renombra a `docs/PLAN.md` solo cuando el `PLAN.md` en curso de ese repo cierre.
- 2026-10-27 · Equipos no guarda (`upsert_device` 400) y el aviso no dice el campo: impide ver D1 en la app (pendiente de UI_CONSISTENCY).
- 2026-10-27 · Catálogo sigue en blanco hasta APP_COMPOSITION R3.

## Relación con otras olas

- APP_COMPOSITION: F4 viaja en el plan de M de cada módulo; F5 en `appkit`; R3 abre Catálogo.
- UI_CONSISTENCY: su pendiente «`crudview.autoSaveAction` pinte el campo que falló» es la otra mitad del aviso.
- dbsync (sin master): su base local y su pull entran por la misma puerta del almacén (D5).
