# Módulos configurables (rubro y rol) — master plan
> **Status:** PENDING · 2026-11-16 · P1–P9 cerradas (D1–D7); espera ARCHITECTURE D62 (fase 10: módulos sin `appkit`) (módulos sin `appkit`); renombrar `patient_directory` espera además a que PATIENT_DIRECTORY_USABILITY publique

Dominio: que **un** módulo sirva a varias apps sin copiarlo: se adapta al **rubro** de la app (nombre, ícono, palabras) y al **rol** de quien lo usa (recepcionista ve buscar/editar a todos; el cliente solo su ficha), y que ambos se prueben en la demo del módulo sin montar la app.

**Para retomar:** skills `api-design` (paso 0: búsqueda en el ecosistema antes de crear vocabulario), `app-modules`, `module-layout`, `project-layout`, `dev-autologin`. Hoy: `patient_directory/module/module.go` (`appkit.Module{Name:…}` sin rubro), `definition.go:21-22` (`Allow`; ninguna ficha ligada a una cuenta), `rbac/rbac.go:110` (permiso = recurso + acción, sin «solo la propia»), `appointment_booking/interfaces.go` (`PatientDirectory`: `FindPatient`/`CreatePatient`), `crud/loopback.go:33` (`LoopbackActingAs`).

## Preguntas abiertas

Ninguna. Antes de planificar D7 (formulario de otro módulo) y la fase 3 («Ver como») se presentan sus capturas.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (P1) | **Contrato de rubro.** La app declara su rubro; cada módulo declara los rubros que soporta, con su etiqueta (en `lang.json`) e ícono (D6); tipos en `model` (dónde: D4; cuáles: D5; el módulo no importa `appkit`: ARCHITECTURE D62); `appkit` rechaza al arrancar un módulo sin soporte para el rubro de la app («clinical_encounter does not support industry retail»). `patient_directory` → **`client_directory`** (repo, módulo, entidad `client`); la tabla `patient` se **renombra a `client` con migración** (respondiste A sin elegir la sub-decisión (b): se toma la recomendada, nombre de tabla = nombre de entidad; dilo si no). | `model`, `appkit` (solo la app), `client_directory`, consumidores |
| D2 (P2) | **El vínculo cuenta → ficha vive en `access`** (una cuenta puede tener varias fichas: madre e hijos, tutor y animales); al crear su primera ficha quien solo ve lo suyo, `access` anota el vínculo; su pantalla va directo a su ficha (crear la primera vez, luego editar), sin buscador ([hoy](../../veltylabs/modules/patient_directory/docs/img/roles-1-today.png) · [solo su ficha](../../veltylabs/modules/patient_directory/docs/img/roles-2-own-record.png)). El módulo marca lo interno con `HiddenFromRecordOwner: true` (notas, activo). Cómo se le da «solo lo suyo» a un rol: D3 | `access`, `model`, `crud`, `view` |
| D3 (P4) | **Roles por maneras de usar un módulo.** `model` define una lista cerrada (`model.ModuleUse`): `ManagesEverything` (admin), `WorksWithAllRecords` (recepción), `SeesOwnRecords` (cliente: usa el vínculo de D2), `VisitsWithoutAccount` (visitante). Cada módulo declara las que soporta (1–4) y dibuja su pantalla según la manera; la app declara sus roles (`model.Role`: código `model.RoleCode` constante de la app, nombre visible, manera, y excepciones por módulo solo cuando difiere, p. ej. recepción en `cash_manager` → `SeesOwnRecords`, su turno de caja). Los tipos viven en `model` (contrato que leen `access`, `rbac`, `crud`, `view`, `devtools`), nunca en `appkit`; por debajo cada manera se traduce a permisos de `rbac` (tabla + acción + qué fichas) que nadie escribe a mano. «Ver como» de la demo lista las maneras del módulo; en una app, los roles de la app | `model`, `rbac`, `access`, `crud`, `view`, `devtools` |
| D4 (P5) | **Los rubros son constantes en `model`** (`model.Industry`; un tipeo no compila) y **sus palabras viven en el `lang.json` de cada módulo** (`langc`), no en un JSON propio: las traducciones son datos, el rubro cambia el comportamiento (bloquea el arranque). Lista y nombres: P6 | `model`, módulos (`lang.json`) |
| D5 (P6, P7) | **Rubro como eje** (palabras, ícono y qué módulo calza) con **4 rubros**, solo los que hoy cambian algo: `IndustryHumanHealth`, `IndustryAnimalCare`, `IndustryEducation`, `IndustryGeneralBusiness` (peluquería, tienda, restaurante, servicios profesionales, taller…); uno nuevo se separa cuando un módulo lo trate distinto. `model.AnyIndustry()` es **función** (una variable de paquete la podría cambiar cualquiera); un módulo genérico la declara y solo los especializados listan los suyos: `clinical_encounter` solo `IndustryHumanHealth` hasta que su sujeto sea una referencia. Palabras por defecto («Clients») y el `lang.json` del módulo solo agrega las del rubro que cambia («Patients», «Owners», «Students»); un negocio mixto elige su rubro principal | `model`, módulos |
| D6 (P8) | **Ícono por defecto del módulo y uno por rubro solo si cambia**, junto a sus rubros en `model.ModuleDefinition` (tras ARCHITECTURE D62): `Icon`, `IconFor []model.IndustryIcon` (ícono = `IconSvg()` de `webtyp/icons`, skill `ui-icons`) | `model`, módulos, chasis del menú |
| D7 (P9) | **`view` ofrece «abrir la pantalla de otro módulo y recibir el resultado»**: la reserva pide «un cliente», se abre el formulario de `client_directory` y al guardar vuelve con su id; los módulos no se importan, el chasis los une; contrato en `view`, nunca en `appkit`. Capturas del flujo antes de planificar | `view`, `layout`, `appointment_booking`, `client_directory` |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| 1 | `model`, `appkit` | Contrato de rubro e íconos (D1, D4–D6) | ✅ local (`gotest` verde; falta publicar en puerta 23 de EXECUTION_ORDER) |
| 2 | `model`, `rbac`, `access`, `crud`, `view` | Maneras de uso y roles (D3); vínculo cuenta → ficha y `HiddenFromRecordOwner` (D2) | ✅ local (`gotest` verde; falta publicar en puerta 23 de EXECUTION_ORDER) |
| 3 | `devtools` | «Ver como»: maneras del módulo en la demo, roles en la app ([maqueta](../devtools/docs/img/view-as-2.png)) | 🔴 espera fase 2 |
| 4 | `client_directory` (antes `patient_directory`), `appointment_booking`, `clinical_encounter`, `mjosefa-cms` | El repo ya está renombrado en GitHub y en local (2026-11-17); falta el interior: módulo, entidad y tabla; consumidores. Orden: después de RECORD_SEARCH (que agrega `text_search` a la tabla `patient`), para migrar la tabla una sola vez con su nombre final | 🔴 espera que PATIENT_DIRECTORY_USABILITY y RECORD_SEARCH publiquen |
| 5 | `view`, `appointment_booking`, `client_directory` | Abrir el formulario de otro módulo y volver con el resultado (D7, [maqueta](../../veltylabs/modules/appointment_booking/docs/img/d7-client-form.png)) | 🔴 espera plan |

## Bloqueos

- 2026-11-16 · La declaración de módulo (rubros, maneras) espera ARCHITECTURE D62: hoy los módulos importan `appkit`.

## Relación con otras olas

- PATIENT_DIRECTORY_USABILITY: termina antes; el renombre se hace sobre su última versión.
- RECORD_SEARCH: la búsqueda no se declara (sus D2, D4); se construye antes del renombre (fase 4).
- ACCESS_REUSE: `access` guarda el vínculo cuenta → ficha (D2) y los roles de la app (D3).
- EXECUTION_ORDER: ordena la ejecución local de este master; Sonnet marca allí y aquí cada paso.
