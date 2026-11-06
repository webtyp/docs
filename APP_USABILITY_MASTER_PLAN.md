# Usabilidad de las apps — master plan
> **Status:** IN PROGRESS · 2026-11-06 · abierta con los datos reales de Pa100T ya importados en `mjosefa-cms` (DATA_IMPORT cerrada); primero U4: «Pacientes» cuelga la app

Ajustar las vistas y el uso de las apps a los datos reales: la base local de `mjosefa-cms` tiene ya 45.973
pacientes, 171.170 reservas y 64.356 fichas de Pa100T (cómo se importó: `mjosefa-cms/docs/PASTDB_MAPPING.md`;
`go run ./migrate/pastdb` vuelve a importar sin duplicar).

## Preguntas abiertas

Ninguna todavía (se escriben al abrir la ola).

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| U1 | Principio del dueño: **un módulo aporta un solo enlace** al menú (responsabilidad única); varios enlaces se justifican por escrito (p. ej. seguridad: personas, roles, asignaciones). La pantalla del módulo muestra de forma simple lo más que se puede hacer ahí, para no saltar entre módulos | `layout`, `appkit`, todos los módulos |
| U2 | **La app agrupa** los enlaces de sus módulos en el menú según su preferencia; los demos de cada módulo prueban con sus propios menús | `appkit.App`, `layout` |
| U3 | En listas largas el **filtro va primero** (p. ej. servicios por especialidad) y la búsqueda después; para filtrar ya existe `components/selectsearch` | `crudview`, módulos con listas largas |
| U4 (D84) | **Listas grandes, declaradas por el módulo.** `view.Page{First, Size, More}`: `First` = filas al abrir (0 = espera una búsqueda; `view.All` = todo con tope), `Size` = de a cuántas por búsqueda y por scroll, `More` = el scroll trae las siguientes hasta «No hay más resultados para esta búsqueda»; sin `Page` rige `view.DefaultPage` (50). La búsqueda la hace el servidor (`view.ListArgs{Search, After, Limit}`); el servidor recorta con `view.Cap` (`MaxPage` 500); se pagina por «después de la última fila». Filtro ≠ página. `patient_directory` `Page{First: 0, Size: 20, More: true}`; `staff_manager` `Page{First: view.All}` | 🟡 `view` v0.6.63 publicado; la cascada rechazó (y dejó en su versión anterior) a los que usan `Filter`/`List` viejo: `layout` primero (casi todos fallan por `crudview`), luego `appkit`, `access`, `auth`, `devtools`, `appointment_booking`, `clinical_encounter`, `business_calendar`, `room_layout`, `payment_manager`, `agreement_manager`, `chat_room`, `work_schedule`, `iam`, `mjosefa-cms` (logs `/tmp/gopush-gate-<repo>.log`); planes por despachar: `layout/crudview` (Search/More; botón «Mostrar más resultados» al final mientras no haya U5), `patient_directory` (búsqueda por RUT con `input.NormalizeRUT`, teléfono o nombre; sin `Or` del `orm`: sus OR no se agrupan y romperían el filtro de tenant), `appointment_booking` (sus 3 listers a la firma nueva) |
| U5 | Scroll automático: cargar las siguientes al llegar al final de la lista. Hoy `dom` solo tiene `OnScrollCapture` de la página; falta saber que un elemento llegó a la vista | `dom` → `components/targetlist` → `crudview` |
| U6 | Búsqueda sin distinguir mayúsculas: `orm`/`storage` solo tienen `Like` (sensible a mayúsculas en Postgres); los nombres de Pa100T están en minúsculas, los nuevos como se escriben | `storage`, `postgres`, `orm` |
| U7 (D77) | «Horario» marca el bloque que cae fuera del horario del establecimiento (74 importados así a propósito, para ajustarlos ahí) | `appointment_booking` |
| U8 (D83) | «Horario» se corta por la izquierda en ancho de celular (400 px) | `appointment_booking` |
| U9 (D66) | El tenant sale de todos los `*Args` (servidor usa `Deps.TenantID`) en `appointment_booking`, `clinical_encounter`, `patient_directory`; se borra el «si viene vacío pongo el mío»; la prueba de conformidad de `router` rechaza un `tenant_id` en un modelo montado con `.Accepts`/`MountSave` | esos 3 + `router` |
| U10 (D68) | `ddl.Sync` no compara tipo ni largo de una columna existente: ensanchar → `ALTER COLUMN … TYPE`; angostar o cambiar tipo con datos → preguntar en la pestaña de migración (dev: `patient.rut` y `business_hours.*_min` se corrigieron a mano) | `ddl`, `postgres` |
| U11 (D70) | Borrar `Deps.ValidateRUT` de `patient_directory` y `staff_manager`: llaman `input.NormalizeRUT` directo (un validador inyectado permitió el falso que ocultó la búsqueda rota) | esos 2 |
| U12 (D80) | `CatalogReader` de `appointment_booking` con un solo método (`ServiceCard`, activo explícito) en vez de `ServiceExists` + `ServiceCard`, que se cruzaban | `appointment_booking`, `service_catalog` |
| U13 (D51) | Las respuestas 400/404 de `/api/*` no salen en el log del servidor | `rpc` o `appkit` |
| U14 (D61) | El error de `appkit.Find` nombra el contrato con un texto de quien llama (~20 llamadas) | `appkit`, módulos |
| U15 (D49) | Faltan traducciones (es) en `appointment_booking` y `clinical_encounter` (`langc sync .`) | esos 2 |
| U16 (H1) | `model.FieldError` sin `Unwrap`; `NotNull` en bool rechaza `false` | `model` |
| U17 (D48) | `agedate.AgeDate` marca sola `birthdate_estimated` (necesita que `form` deje a un control escribir otro campo); la importación ya marca estimada toda fecha «AAAA-01-01» | `form`, `components`, `patient_directory` |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| A1 | `layout` | (Bloquea uso diario) El menú lateral se corta en escritorio (y en móvil al abrir «Usuarios y roles»): no tiene desplazamiento | por diseñar |
| A2 | `appointment_booking`, `staff_manager` | Dos «Personal» en el menú (fichas vs horario y servicios por profesional) | por diseñar, con U1 |
| A3 | todos los módulos | Revisar cuántos enlaces aporta cada uno y justificar los que sean más de uno | por diseñar |
| A4 | `appkit`, `layout` | Agrupación del menú declarada en la app (U2): qué cambia en `appkit.App` | por diseñar |
| A5 | `service_catalog` y otros | Filtro por especialidad antes que búsqueda (U3) | por diseñar |
| B1 | `layout` | U4: `crudview` con `Search`/`More` y «Mostrar más resultados» | [Claude · 2026-11-06 18:00] despachado (Jules 486127966435176406); revisa Antigravity |
| B2 | ola 1, sin `layout`: `auth`, `alert_manager`, `cash_manager`, `editorial`, `network_manager`, `payment_manager`, `pharmacy_manager`, `supply_manager`, `work_schedule` | U4: la operación de lista acepta `view.ListArgs`, `Page` declarado (receta `view/docs/LIST_PAGING.md`) | [Claude · 2026-11-06 18:00] despachados (Jules); revisa Antigravity |
| B3 | ola 2, con `layout`: `access`, `agreement_manager`, `appointment_booking`, `business_calendar`, `chat_room`, `clinical_encounter`, `device_manager`, `patient_directory` (búsqueda por RUT/teléfono/nombre, abre vacío, 20 por página), `room_layout`, `service_catalog`, `staff_manager` | lo mismo | `docs/PLAN.md` escrito en cada repo, **sin despachar**: se despacha cuando B1 cierre (`go get webtyp.com/layout@latest` del plan toma esa versión); `room_layout` tiene cambios locales sin publicar del dueño (mapa R2): no tocarlos |
| F | `mjosefa-cms` | Capturas de cada pantalla con datos reales y el menú nuevo; publicar `mjosefa-cms` (cambios de la importación sin publicar: `migrate/pastdb`, `service_map.csv`, `catalog.csv`, docs) | al final |

## Coordinación Claude ↔ Antigravity

Este master es el único canal. Antigravity revisa los PR cada 20 min y Claude lee este master cada 30 min. Escribe
cada uno abajo, con fecha y hora, y nunca en el chat. Reglas: **plan-review** (máximo 2 intentos del agente, luego
corrección local), **codejob** (`codejob pull` / `codejob close`), marca de dueño en la fila de la fase antes de
actuar (**master-plan** «Varios agentes»), y `replace` local + `gotest` de `mjosefa-cms` antes de cerrar un repo que
la app usa.

### Mensajes

- [Claude · 2026-11-06 18:00] Ola 1 despachada (B1, B2). Cuando cierres `layout` (B1), despacha la ola 2 (B3):
  `cd <repo> && codejob dispatch` en cada uno; si ya hay 15 planes corriendo, espera. Al cerrar cada repo que usa
  `mjosefa-cms`, deja aquí una línea con la versión publicada.

## Bloqueos

- Ninguno.

## Relación con otras olas

- DATA_IMPORT (`mjosefa-cms`): cerrada 2026-11-06; sus pendientes viven aquí (U4–U17). Ola 3 aparte: adjuntos y gastroscopía con `webtyp/pdf` (una tabla por especialidad unida 1 a 1 a la ficha); falta publicar `webtyp/docs/ORIGIN.md`.
- UI_CONSISTENCY: reglas visuales de formularios y colores; esta ola trata navegación y flujo.
