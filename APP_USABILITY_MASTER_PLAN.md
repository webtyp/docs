# Usabilidad de las apps — master plan
> **Status:** IN PROGRESS · 2026-11-11 · la ola de listas grandes (U4) está cerrada y publicada; decididos el menú, la búsqueda y el filtro; faltan sus planes y la limpieza de contratos

Ajustar las vistas y el uso de las apps a los datos reales: la base local de `mjosefa-cms` tiene 45.973 pacientes, 171.170 reservas y 64.356
fichas de Pa100T (`mjosefa-cms/docs/PASTDB_MAPPING.md`; `go run ./migrate/pastdb` vuelve a importar sin duplicar).
**Cerrado:** U4 (listas grandes: `crudview` con `Search`/`More`, `view.Page`, receta `view/docs/LIST_PAGING.md`) en `layout` v0.3.108 y en los 20
módulos de las olas 1 y 2.

## Preguntas abiertas

Ninguna. (2026-11-11: P1–P4 respondidas A; sus decisiones están abajo.)

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| U1 | Principio del dueño: **un módulo aporta un solo enlace** al menú (responsabilidad única); varios enlaces se justifican por escrito (p. ej. seguridad: personas, roles, asignaciones). La pantalla del módulo muestra de forma simple lo más que se puede hacer ahí, para no saltar entre módulos | `layout`, `appkit`, todos los módulos |
| U5 | Scroll automático: cargar las siguientes al llegar al final de la lista. Hoy `dom` solo tiene `OnScrollCapture` de la página; falta saber que un elemento llegó a la vista | `dom` → `components/targetlist` → `crudview` |
| U7 (D77) | «Horario» marca el bloque que cae fuera del horario del establecimiento (74 importados así a propósito, para ajustarlos ahí) | `appointment_booking` |
| U8 (D83) | «Horario» se corta por la izquierda en ancho de celular (400 px) | `appointment_booking` |
| U9 (D66) | El tenant sale de todos los `*Args` (servidor usa `Deps.TenantID`) en `appointment_booking`, `clinical_encounter`, `patient_directory`, `editorial`, `pharmacy_manager`, `supply_manager`, `payment_manager`, `agreement_manager`; se borra el «si viene vacío pongo el mío»; la prueba de conformidad de `router` rechaza un `tenant_id` en un modelo montado con `.Accepts`/`MountSave` | esos módulos + `router` |
| U10 (D68) | `ddl.Sync` no compara tipo ni largo de una columna existente: ensanchar → `ALTER COLUMN … TYPE`; angostar o cambiar tipo con datos → preguntar en la pestaña de migración (dev: `patient.rut` y `business_hours.*_min` se corrigieron a mano) | `ddl`, `postgres` |
| U11 (D70) | Borrar `Deps.ValidateRUT` de `patient_directory` y `staff_manager`: llaman `input.NormalizeRUT` directo (un validador inyectado permitió el falso que ocultó la búsqueda rota) | esos 2 |
| U12 (D80) | `CatalogReader` de `appointment_booking` con un solo método (`ServiceCard`, activo explícito) en vez de `ServiceExists` + `ServiceCard`, que se cruzaban | `appointment_booking`, `service_catalog` |
| U13 (D51) | Las respuestas 400/404 de `/api/*` no salen en el log del servidor | `rpc` o `appkit` |
| U14 (D61) | El error de `appkit.Find` nombra el contrato con un texto de quien llama (~20 llamadas) | `appkit`, módulos |
| U15 (D49) | Faltan traducciones (es) en `appointment_booking` y `clinical_encounter` (`langc sync .`) | esos 2 |
| U16 (H1) | `model.FieldError` sin `Unwrap`; `NotNull` en bool rechaza `false` | `model` |
| U17 (D48) | `agedate.AgeDate` marca sola `birthdate_estimated` (necesita que `form` deje a un control escribir otro campo); la importación ya marca estimada toda fecha «AAAA-01-01» | `form`, `components`, `patient_directory` |
| U2 | **La app agrupa** los enlaces del menú: `appkit.App{Menu: []appkit.MenuGroup{{Label: "Clínica", Screens: []string{"patient", "clinical_encounter"}}}}`; un módulo no sabe en qué grupo cae; los demos de cada módulo prueban con sus propios menús | `appkit.App`, `layout` |
| U3 | En listas largas el **filtro va primero**: el módulo marca el campo, `model.Field{Name: "specialty", Filter: true}`; `crudview` dibuja un selector (`components/selectsearch`) sobre la lista y lo envía en `ReadArgs.Filter` | `model`, `crudview`, módulos |
| U6 | **`storage.ILike`** (Postgres `ILIKE`, SQLite `LIKE`): la búsqueda de texto de `crud`/`crudview` no distingue mayúsculas; los nombres de Pa100T están en minúsculas, los nuevos como se escriben | `storage`, `postgres`, `sqlite`, `mem`, `crud` |

## Fases

| Fase | Repo | Qué | Estado | Plan |
|---|---|---|---|---|
| A1 | `layout` | (Bloquea uso diario) El menú lateral se corta en escritorio y en móvil: no tiene desplazamiento | 🟡 Jules `17455336479303599613` | `layout/docs/PLAN.md` |
| A2 | `appointment_booking` | Dos «Personal»: el de agenda pasa a «Horario del personal» (dos enlaces, dos nombres) | decidido; espera que `appointment_booking` publique su migración | — |
| A3 | todos los módulos | Cuántos enlaces aporta cada uno; justificar los que sean más de uno (U1) | por escribir | — |
| A4 | `appkit`, `layout` | Agrupación del menú declarada en la app (U2) | por escribir | — |
| A5 | `model`, `crudview`, `service_catalog` | Filtro antes que búsqueda (U3) | por escribir | — |
| S1 | `storage`, `postgres`, `sqlt`, `indexdb`, `orm`, `crud` | `storage.ILike` (U6), en orden: `storage` + `mem` + conformidad (🟡 Jules `4940575217866102135`, `storage/docs/PLAN.md`); **la búsqueda va dentro de la conformidad**: cláusula `ilike_ignores_case` en `storage/conformance` (la corre cada backend: `mem`, `sqlt`, `postgres`, `indexdb`, en rojo hasta implementarla) y la misma cláusula en `crud/conformance` (cada `crud.Store` debe cumplir `ILike`); luego `sqlt` (SQLite: `ILIKE`→`LIKE`), `postgres` (`ILIKE` directo), `indexdb`, `orm` (`QB.ILike`), `crud` (usa `ILike` sin bajar a minúsculas el término) | por escribir cada uno al publicar `storage` | — |
| C1 | `model` | U16: `Unwrap` en `FieldError` y `NotNull` en bool | 🟡 Jules, sesión `11594222981559106768` | `model/docs/PLAN.md` |
| C2 | `patient_directory`, `staff_manager` | U11: `Deps.ValidateRUT` fuera; las pruebas usan RUT válidos (`11111111-1`, `12345678-5`, `7654321-6`) en vez del doble que acepta cualquiera | por escribir; `patient_directory` tiene cambios sin guardar del dueño | — |
| F | `mjosefa-cms` | Capturas de cada pantalla con datos reales y el menú nuevo; vive en la fase A5 de ARCHITECTURE (local, en etapas) | espera ARCHITECTURE A5 | `mjosefa-cms/docs/PLAN_STAGE_*.md` |

## Bloqueos

- 2026-11-07 · publicar `iam` espera P1 de [ARCHITECTURE_MASTER_PLAN.md](ARCHITECTURE_MASTER_PLAN.md) (`view` arrastra `dom` al servidor).

## Relación con otras olas

- DATA_IMPORT (`mjosefa-cms`): cerrada 2026-11-06; sus pendientes viven aquí (U1–U17). Ola 3 aparte: adjuntos y gastroscopía con `webtyp/pdf` (una tabla por especialidad unida 1 a 1 a la ficha); falta publicar `webtyp/docs/ORIGIN.md`.
- ARCHITECTURE: P1 (`view` sin `dom`) desbloquea `iam`; A5 (`mjosefa-cms` en local) cierra la fase F de esta.
- FILE_UPLOAD (`FILE_UPLOAD_DESIGN.md`): subida de archivos con `input.ImageFile` & co.
- UI_CONSISTENCY: reglas visuales de formularios y colores; esta ola trata navegación y flujo.
