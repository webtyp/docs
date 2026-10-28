# Lo que se guarda se ve en todas partes: registros reactivos — master plan
> **Status:** IN PROGRESS · 2026-10-28 · sin preguntas abiertas. **G1, G2, G3 y G4 cerradas 100%**; **F2b cerrada** (`dbsync` v0.3.11). F5 listo para redactar (gate de api-design y plan-authoring).

**Disparador (usuario, 2026-10-27):** al **editar** un registro la fila no cambia aunque salga «Guardado con éxito»,
y lo que depende de él (las opciones de un selector) tampoco. **Regla del dueño:** lo que se edita queda ligado campo
a campo a lo que se muestra, de forma nativa y fácil para un dev junior; ruptura sin retrocompatibilidad. Precedente
propio: pa100t repartía cada resultado CRUD por websocket a los clientes con acceso a ese recurso; `crudp`
(`~/Dev/Archive/crudp`) quiso ser ese protocolo.

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 | **Lo que se guarda se ve en todo lugar que lo muestra**, sin recargar a mano ni código del módulo. Regla en **ui-consistency**; test en **testing** | todas, Z |
| D3 (P1, P6) | **Una sola copia de cada registro por página**: `view.Records` (modelo + id), compartida por listas, selectores y formularios; `appkit` la arma una vez (`view.NewRecords()` + `records.Caller(...)`); los presenters reciben `OnChange`. Guarda cada ítem **calculado una vez**, al poner el registro | G1, F5 |
| D4 (P2, P7) | **Un solo tipo de ítem de lista, `model.Item{ID, Label, Description, Badge}`, declarado en los campos y escrito por `ormc`.** El campo lleva `Item: model.ItemLabel` (uno), `ItemDescription` (varios, unidos con `model.JoinItem` « · » en orden de campo) o `ItemBadge` (uno; una fecha va en segundos). `ormc` genera `func (m *X) Item() model.Item` tipado y rechaza lo inválido con `model.ValidateItem`; un registro listado es `view.Record` (`model.Model` + `model.Itemizer`), así que un modelo sin marcas no compila como lista. Un selector dice de qué lista sale: `Ref: &M` (llave foránea) u `Options: &M` (sin ella). Agregar un dato a la lista = una línea en ese campo. Reemplaza lo publicado en model v0.2.7 / view v0.6.47 (`model.Row`, `RowOf`, `view.Item`: dos tipos para una idea y una proyección O(N²) en ejecución) | G1–G4 |
| D5 (P3) | **La fila cambia cuando el guardado se confirma**: guardar responde el registro guardado y el almacén lo aplica. Offline, el mismo código: `dbsync.Client.Caller()` responde desde la base local por la misma interfaz `router.Caller` | G4, F5 |
| D6 (P4) | **Un ítem muestra solo campos de su propio registro**; una copia congelada a propósito (`PatientNameSnapshot`) es un campo normal que no cambia | G3 |
| D7 (P5, P8, P9) | **El cambio de un usuario llega a todos los que lo están viendo, sin código en el módulo**: el servidor anota cada escritura en el `changelog`; por SSE (`webtyp/sse`) solo avisa «hay cambios hasta la versión N»; el navegador pide `/pull` (filtra por permiso de lectura y por fila: `Rule.Resource`, `Rule.Scope`) y pone las filas en `view.Records`. Un solo camino online y offline. Hoy un servidor = un tenant (`mjosefa-cms/modules/tenant.go:7`); si un servidor atendiera varios, falta el tenant de la sesión en `auth` | F2b, F5 |
| D10 (P10) | **`appkit` centraliza; la app elige la base de datos.** La app le pasa su conexión (puerto `storage`, sobre el motor que eligió); `appkit` la envuelve en el `changelog`, monta el servidor de `dbsync` con reglas armadas desde los modelos y sus recursos, y en el navegador arma `view.NewRecords()` y el `dbsync.Client`. `appkit` usa contratos agnósticos (`storage`, `orm`, `changelog`) y **nunca** importa un motor (invariante 6 de **project-layout**) | F5 |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| F0 | `layout/crudview` | test «edito → la fila cambia» con backend que responde después | 🟢 cerrada (layout v0.3.77) |
| F1 | `model`, `ormc` | primera versión de la marca de fila (`Row`, `RowOf`) | 🟢 cerradas v0.2.7 / v0.1.25 — **la reemplaza G1** |
| F2 | `view` | `view.Records`, `OnChange`, guardar responde el registro, selectores desde `Ref`/`Options` | 🟢 cerrada v0.6.47 (G1 cambia su tipo de ítem) |
| F3 | `components`, `layout` | `Badge` con fecha/hora traducidas; crudview sigue `OnChange` sin recargar | 🟢 cerradas v0.8.56 / v0.3.78 |
| F4 | 15 módulos + `auth` | marcas en los campos, sin `Item()` a mano ni `Lookups` | 🟢 publicados con la marca vieja `Row:` — **G3 la renombra** |
| G1 | `model` → `ormc` → `view` (**puertas: ejecución local de Antigravity**, en ese orden) | `model.Item`, `ItemSlot`, `JoinItem`, `ValidateItem`; `ormc` genera `Item()`; `view.Record`, ítems calculados una vez, sin `view.Item` ni `RowOf` | 🟢 cerradas: model v0.2.9, ormc v0.1.26, view v0.6.48 |
| G2 | `components` → `layout` | alias `Item = model.Item`; crudview y menú con `model.Item` / `view.Record` | 🟢 cerradas: `components` v0.8.57, `layout` v0.3.79 |
| G3 | `alert_manager`, `business_calendar`, `device_manager`, `editorial`, `network_manager`, `patient_directory`, `payment_manager`, `pharmacy_manager`, `room_layout`, `staff_manager`, `supply_manager`, `work_schedule`, `cash_manager`, `webtyp/auth`, `service_catalog` | `Row:` → `Item:`, `ormc` regenera `Item()`, `RowOf` → `.Item()` | 🟢 cerradas 15 librerías (14 módulos + `auth`; incluye `patient_directory` v0.1.64 y `staff_manager` v0.1.157) |
| G4 | `access` (v0.0.16), `appointment_booking` (v0.3.50), `chat_room` (v0.2.104), `clinical_encounter` (v0.2.24) | `Item:` en modelos; `dbsync` recibe D11 | 🟢 cerradas las 4 librerías |
| F2b | `dbsync`, `sse` | SSE «el head se movió» desde `changelog.OnChange`; pull → `view.Records` (`ClientConfig.Records` obligatorio) | 🟢 cerrada v0.3.11 |
| F5 | `appkit`, `mjosefa-cms` | D10: `appkit` recibe la conexión de la app y arma `changelog` + servidor `dbsync` + en el navegador `view.NewRecords()` y `dbsync.Client`; `mjosefa-cms` solo pasa su conexión | ⚪ se escribe tras F2b y G1 (gate de api-design) |
| Z | `mjosefa-cms` | dos sesiones: una edita un servicio y una especialidad; en ambas cambian la lista y el selector de Servicios; capturas revisadas | ⚪ al final |

## Bloqueos

- 2026-10-28 · Orden: G1 en local (`model` v0.2.8 → `ormc` v0.1.26 → `view` v0.6.48) → G2 (`components`, luego `layout`) → G3 y G4 → F5 → Z. Revisión y cierres: Antigravity (temporizador de 15 min).
- 2026-10-27 · Equipos no guarda (`upsert_device` 400) y el aviso no dice el campo: impide ver D1 en la app (pendiente de UI_CONSISTENCY).

## Relación con otras olas

- APP_COMPOSITION: los 3 módulos de M en vuelo reciben la corrección de G4; F5 vive en `appkit`.
- UI_CONSISTENCY: su pendiente «`crudview.autoSaveAction` pinte el campo que falló» es la otra mitad del aviso.
