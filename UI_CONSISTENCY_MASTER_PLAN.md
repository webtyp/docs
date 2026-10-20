# Consistencia visual entre módulos — master plan
> **Status:** IN PROGRESS · 2026-10-20 · ola 3 cerrada en casi todos los módulos (grupo, `Label`, `decktabs`→grupo); `business_calendar` con un plan despachado (reloj 24h en el editor de horario); `item_catalog`/`service_catalog` sigue con `replace` locales

Índice de olas: [MASTER_PLANS.md](MASTER_PLANS.md). Forma de trabajo: skill **master-plan** (este documento es el estado; el chat no resume).

## 1. Qué quedó fuera de este documento

Lo ya decidido y verificado pasó a **documentación y skills**; aquí solo vive lo que falta:

- **Reglas** (formulario, título único, roles de color, foco y selección ámbar, datos neutros): skill **ui-consistency** (`devskills/skills/ui-consistency`).
- **Razones, prior art, alternativas descartadas y el experimento del anillo en degradado:** [UI_CONSISTENCY_DESIGN.md](UI_CONSISTENCY_DESIGN.md).
- **Publicado antes:** `widget` v0.6.41 (`style.Heading()`), `form` v0.4.41 (etiqueta arriba, validar al salir, `Label` obligatorio, `Radio` en chips), `components` v0.8.31 (títulos con `Heading()`, `decktabs` borrado), `item_catalog` v0.3.76 (referencia: select de especialidad en el formulario, `Label` en todo campo).

**Objetivo que sigue en pie:** `platformd` es responsable de cómo se ve la app; un módulo solo declara *qué* contenido tiene. Módulo de referencia: `veltylabs/modules/item_catalog` (todo cambio se prueba primero ahí). Fuera de alcance: rail y drawer de `platformd`, la *elección* del tema de color, iconografía y piezas internas de un módulo.

## 2. Decisiones cerradas por implementar

**Formulario y color — hechos en local, sin publicar** (D3, D20, D21, D22, D23 y la accesibilidad de campos): el detalle de cada regla vive en la skill **ui-consistency** y en [UI_CONSISTENCY_DESIGN.md](UI_CONSISTENCY_DESIGN.md); lo que cambió en cada árbol está en §4.

**Estructura en los módulos (esperan el tag de `layout`):** «Personal» (Datos y dispositivos / Horario / Servicios) pasa a ser un grupo con `platformd.NewGroupUIModule`, como `item_catalog` ([browser.go](../../veltylabs/modules/item_catalog/ui/browser.go)).

- **D5** — todo formulario es `form.New` sobre un modelo existente. Reserva paso 4: buscar = `FindPatientByRutArgs` (`SubmitLabel("Search")`); registro = `Patient` con `ShowField(rut, name, phone, email)`; online = `LinkPatientArgs`. Se borran `textField` y los `PartFlow*` de campo ([booking_flow.go:370-374](../../veltylabs/modules/appointment_booking/ui/booking_flow.go#L370-L374)).
- **D8** — registro de paciente **en línea**: si el RUT no existe, el formulario de registro aparece en el mismo paso con el RUT cargado; se elimina el `modaldialog` del paso 4.
- **D15 (resto)** — en `appointment_booking`: el selector de profesional de Horario y Servicios pasa a `crudview.Config.Scope` (ya existe en `layout`, local); «Máx. sobrecupo/día» → formulario de `CalendarConfig`. (El select de especialidad en `item_catalog` ya está publicado.)
- **D16 (resto)** — Personal, `chat_room` y `business_calendar` pasan a `NewGroupUIModule`; entran los inputs a mano de `payment_manager` y `chat_room`; `room_layout` queda fuera (edición sobre lienzo). Se tomó por aceptada; corrígelo aquí si no.

## 3. Preguntas abiertas

_(ninguna)_

## 4. Librerías y orden

| Ola | Librería | Cambio | Estado |
|---|---|---|---|
| 1b | `webtyp/form` | **arreglo de «tenant_id required»:** el formulario conserva al guardar los campos del registro cargado que no muestra (sin widget); rejilla de 2 columnas, selección y check en ámbar, check de fila completa, radio con nombre accesible, select con guía y con `Picker()` (`PartSelect`), sin tooltip técnico | **publicado** `form` v0.4.43 |
| 1b | `webtyp/widget` | superficie `Inverse`, `DockedClear(space)`, superficie `AccentOutline` (anillo inset de acento que no repinta) y receta `Picker()` + `PartSelect` (popup del select con `base-select`, bajo `@supports`); tests de comportamiento | **publicado** `widget` v0.6.43 (`AccentOutline`, `Inverse`, `DockedClear`, `Picker()` documentados en `docs/SPECS.md`) |
| 1b | `webtyp/components` | `targetlist`: badge neutro sobre el borde inferior derecho (fuera del flujo, posición fija), hover/foco con `AccentOutline`; `chipgrid`, `cellgrid` y el día de `scheduleeditor` elegidos en ámbar; `usermenu`: el trigger lee sobre el header primario (`Button(Primary)`); se borra el test del badge sobre el borde | **publicado** `components` v0.8.34 |
| 2 | `webtyp/layout` | título en el header (`<h1>` único y grande junto a la marca; nombre y versión de la app en el panel que abre la marca, `Platform.AppVersion`; barra primaria), `rightpanel` sin banda de título, `crudview` `Context` → `Scope` en la cabecera de la lista; mensaje en superficie `Inverse` bajo el header en móvil, `Auto()` ≥ 4 s, errores persistentes y sin apilar, «Could not save: …»; detalle a ancho completo en móvil | **publicado** `layout` v0.3.51 (etapas 1–5 de su plan, guardado como `docs/LAST_PLAN_EXECUTED.md`); `chatview` ya no usa `decktabs` (borrado de `components` en v0.8.31): lleva su propio selector de dos pestañas |
| 3 | `veltylabs/modules/appointment_booking` | Personal → grupo; reserva con `form.New`; registro en línea; `Scope` en Horario/Servicios; sobrecupo en `CalendarConfig` | **cerrada** (landed junto con su migración de TIME_UNITS) |
| 3 | `staff_manager`, `patient_directory`, `business_calendar`, `device_manager`, `editorial`, `service_catalog` | `Label` en todo campo con widget | **cerrada**: `staff_manager` v0.1.120, `patient_directory` v0.1.33, `business_calendar` v0.4.87, `device_manager` v0.2.55, `editorial` v0.0.25, `service_catalog` v0.5.4; de paso, `editorial.slug` y `service_catalog.Specialty.slug` tenían `Label: "Slug"` (jerga, no explica nada): ahora `Label`+`Help` describen qué hace el campo |
| 3 | `chat_room`, `business_calendar`, `payment_manager` | `decktabs` → grupo; inputs a mano → `form` | **cerrada** |
| 3 | `business_calendar` | editor de horario mostraba minutos-desde-medianoche para escribir («480» en vez de «08:00»): se cambia a reloj 24h (`input.Hour()`) | **despachada** (`docs/PLAN.md`, Jules, sesión 15090588848331827000) |

Los planes de módulos (ola 3) se escriben cuando estén publicados los tags de su base, para fijar versiones exactas.

## 5. Pendientes

- **`item_catalog` conserva `replace` locales** a `form`, `components`, `widget` y `layout` (decisión del usuario: faltan detalles de UI). `gopush` los quita de los dependientes con commit de `go.mod` ('deps only'): tras cada `gopush` de esas librerías hay que volver a ponerlos (sin commit). Se quitan al cerrar `item_catalog`.
- Fallos de la cascada **ajenos a esta ola** (mocks que no implementan `router.Route.RequiresMFA` / `router.Context.Assurance`): `patient_directory`, `room_layout`, `dbsync`, `pwa`; son de SECURITY_BY_DEFAULT.
- Revisar `stepindicator`: su paso actual va en primario y el seleccionado en `AccentInverse`; D20 dice que «actual» es ámbar, pero la semántica de progreso es distinta y no se tocó.
- **Módulos despachados (2026-10-19, `codejob`, Jules):** `business_calendar`, `chat_room` y `appointment_booking` (su `docs/PLAN.md`: `decktabs` → grupo, API nueva de `layout`, replaces locales fuera). Revisión cada 15 min con `codejob` + `codejob pull`. **Estacionado:** `appointment_booking/docs/PLAN_BOOKING_FORMS.md` (D5, D8 y sobrecupo en formulario) espera un release de `patient_directory` con `Label` en sus campos con widget (su plan está por escribir; su `gotest` hoy falla por mocks de `RequiresMFA`/`Assurance`, de SECURITY_BY_DEFAULT).
- **Seguimiento de `mjosefa-cms`:** al publicarse `chat_room`, quitar `chatroomui.WithoutGroups()` de `modules/browser.go:69` (la opción se borra); su `go mod tidy` falla mientras los módulos con `decktabs` no se republiquen.
- Escribir los planes de los demás módulos (ola 3) cuando estén los tags; migrar `appointment_booking` (hoy no compila: `decktabs` borrado, `RightPanel.Title/HeadControls` y `Config.Context` quitados de `layout`).
- Borrar `widget.PartRadioChecked` / `PartRadioUnchecked` de `widget/field.go` (sin uso desde que el `Radio` es chips).
- **Traducciones** («field Name is required», «Select an option», «Could not save/update/delete», etiquetas; 21+ claves en `item_catalog/lang.json`): `langc sync` y traducir.
- Celdas huérfanas en la rejilla de 2 columnas: depende del orden de los campos del modelo; si molesta, el modelo declara `SpanFull`.
- En móvil el botón flotante ↺ solapa ~14 px el borde derecho del último campo al final del scroll: margen en `crudview` (va en el plan de `layout`).
- Icono de revelar contraseña activo (ámbar sobre fondo claro, ~1,7:1): vigilar el contraste.
- Método: la demo regenera `style.css` solo ante un cambio de **contenido** (de `go.mod` o de un archivo del módulo con `replace`); el MCP del navegador no envía teclas, ni hover, ni cambia de tema: para eso se usa un Chromium aparte con `--blink-settings=primaryHoverType=2,availableHoverTypes=2,primaryPointerType=4,availablePointerTypes=4` contra `https://localhost:8080`.

## 6. Relación con otras olas

Independiente de SECURITY_BY_DEFAULT y API_SURFACE. Comparte `form`/`dom` con NO_REFLECTION_MASTER_PLAN (su ola 1 ya está publicada). Extiende SOURCE_SELECTION (`Label`/`Help` en los modelos). Cambia API publicada de `layout` (`rightpanel.Title`, `HeadControls`, `crudview.Config.Context` → `Scope`) sin consumidores fuera de `veltylabs`.
