# Consistencia visual entre módulos — master plan
> **Status:** IN PROGRESS · 2026-10-14 · D1–D17 cerradas; widget, components, form e item_catalog publicados; layout retenido

Índice de olas: [MASTER_PLANS.md](MASTER_PLANS.md). Forma de trabajo: skill **master-plan** (este documento es el estado; el chat no resume).

## 1. Objetivo

**`platformd` es el responsable de cómo se ve la app; un módulo solo declara *qué* contenido tiene.**
El alcance es la consistencia **mínima** que el usuario percibe al cambiar de pantalla: **los
formularios** (cada módulo es prácticamente un formulario) y **el encabezado del módulo**, que sube al
header de la app. Las piezas visuales propias de un módulo, que nadie más reutiliza, siguen viviendo y
estilizándose dentro del módulo (D7).

Módulo de referencia: **`veltylabs/modules/item_catalog`** — el más simple; ya agrupa sus pantallas con
`platformd.NewGroupUIModule`. Todo cambio de esta ola se prueba primero ahí.

Fuera de alcance: rail y drawer de `platformd`, tema de color, iconografía, piezas visuales internas de un módulo.

## 2. Decisiones

- **D1** (cierra P11 original) — Este master plan va en el idioma del usuario; los `docs/PLAN.md` que se despachan con `codejob` van en inglés. Regla en **dev-protocols** y **master-plan**.
- **D2** (cierra P1) — **Un solo formato de campo: etiqueta arriba del control**, igual en móvil y escritorio:

  ```
   Nombre completo *              ← <label> texto, traducido, sin truncar
  ┌──────────────────────────────┐
  │ Dra. Ana Rojas               │ ← control a ancho completo, alto táctil ≥ 44px
  └──────────────────────────────┘
   Formato: 12.345.678-9           ← ayuda (solo si el modelo trae Help)
   ⚠ RUT inválido                  ← error; reemplaza a la ayuda mientras exista
  ```

  Se elimina el chip sobre el borde ([form/css.go](../form/css.go) `PartLabel` `OnEdge`) y el truncado
  a 14 bytes ([form/render_input.go:19](../form/render_input.go#L19), `labelChars`). El markup de `form` ya
  es `label → help → control → error` ([render_input.go:128-200](../form/render_input.go#L128-L200)): el
  cambio es de hoja de estilos; todo módulo con `crudview` lo hereda sin tocar código. Respaldo: Baymard,
  NN/g, GOV.UK Design System, Polaris, Material 3 (etiqueta arriba = más rápido de llenar, sin ambigüedad).
- **D3** (cierra P2) — **Una sola columna** siempre, ancho máximo legible (~40rem). Se elimina `PartForm` `Grid(2, …)`.
- **D4** (cierra P3) — **Ayuda y error debajo del control**; el error reemplaza a la ayuda; se valida **al
  salir del campo y al enviar**, no en cada tecla.
- **D5** (cierra P4) — **Todo formulario es `form.New` sobre un modelo existente.** Reserva paso 4: buscar
  = `FindPatientByRutArgs` (`SubmitLabel("Search")`); registro = `Patient` con `ShowField(rut, name, phone,
  email)`; online = `LinkPatientArgs`. Se borran `textField` y los `PartFlow*` de campo
  ([booking_flow.go:370-374](../../veltylabs/modules/appointment_booking/ui/booking_flow.go#L370-L374)).
- **D6** (cierra P5) — **El título del módulo sube al header de la app.**
  - Móvil: **sin cambios** ([img/app-mod-header-mobile.png](img/app-mod-header-mobile.png)) — se ve bien y ocupa poco.
  - Escritorio: desaparece la banda de título dentro del contenido (`rp__header`, ver
    [img/app-mod-header-desktop-remove.png](img/app-mod-header-desktop-remove.png)); la esquina superior
    izquierda del header muestra `[icono-app] [nombre-app] / [nombre-módulo]` (dinámico), fondo color
    primario, texto e icono blancos. Hoy ([img/app-mod-header-desktop.png](img/app-mod-header-desktop.png))
    el nombre del módulo solo aparece en móvil ([platformd/css.go:62-71](../layout/platformd/css.go#L62-L71)).
  - El formulario empieza más arriba.
  - **Los controles que hoy van junto al título van dentro del formulario**, no en el header
    (`rightpanel.HeadControls`, `crudview.Config.Context`).
  - **`decktabs` sale de las pantallas de módulo**: ocupa demasiado espacio; agrupar pantallas es
    `platformd.NewGroupUIModule`, como ya hace `item_catalog`
    ([item_catalog/ui/browser.go:96-104](../../veltylabs/modules/item_catalog/ui/browser.go#L96-L104)).
    "Personal" (Datos y dispositivos / Horario / Servicios) pasa a ser un grupo.
- **D7** (cierra P7) — **No se impide que un módulo estilice sus propias piezas** (mapa de planta,
  editor de horario, etc.): hay piezas que no se reutilizan y mueren en su módulo. **No hay bloqueo en el
  build** (`sitec` sale de la ola). La frontera de consistencia es: **campos → `form`**, **encabezado del
  módulo → `platformd`**.
- **D8** (cierra P8) — **Registro de paciente revelado en línea**: si el RUT no existe, el formulario de
  registro aparece en el mismo paso con el RUT ya cargado; se elimina el `modaldialog` del paso 4.
- **D9** (cierra P9; **reemplazada por D17**) — ~~`chipgrid` implementa `form.Renderer`~~: obligaba a los modelos de dominio a importar UI.
- **D10** (cierra P10) — **Todo campo con widget declara `Label`** (inglés, se traduce) y `form` emite un
  **diagnóstico ruidoso en desarrollo** si falta, en vez de mostrar el nombre del campo (hoy se ve "Name"
  en `#personal`; `model.Field.Label` existe en [model/field.go:83-92](../model/field.go#L83-L92)).

- **D11** (cierra P11) — **Foco con color primario**: el campo que se edita pinta su etiqueta (y el marco, con el tratamiento de foco de `Field`) en primario; el resto neutro. Es el único rasgo de marca del campo.
- **D12** (cierra P12) — **Un solo nivel de título dentro del contenido**: `<h1>` = nombre del módulo en el header (`platformd`); dentro del contenido solo `<h2>` de sección con la receta `style.Heading()` (`widget/style`). Ningún componente ni módulo fija tamaño/peso de un título.
- **D13** (cierra P13) — **Toda la barra del header en color primario** en escritorio, igual que en móvil; usuario y estado en blanco.
- **D14** (cierra P14) — Hijo de un grupo: se muestra **`[app] / <pantalla activa>`** (lo que `activeLabel()` ya devuelve).
- **D15** (cierra P15) — Valor del registro → campo del formulario (`item_catalog.specialty_id` como `input.Select()`, se borran `requireSpecialty` y `specialtyPicker`). **Filtro de alcance** → cabecera de la lista sobre el buscador: `crudview.Config.Context` pasa a `Config.Scope`. "Máx. sobrecupo/día" → formulario de `CalendarConfig`.
- **D17** (cierra P17) — **El `Radio` de `form` se dibuja como chips** (misma apariencia que `chipgrid`: cajas táctiles en fila que se envuelven, la elegida en primario); `chipgrid` queda como selector fuera de formularios. Sin API nueva. Agregado al plan de `form` después del despacho y enviado a Jules con `codejob reply`.
- **D16** (cierra P16) — `decktabs` se borra; Personal, `chat_room` y `business_calendar` pasan a `NewGroupUIModule`. Entran a la ola los inputs a mano de `payment_manager` y `chat_room`; `room_layout` queda fuera (edición sobre lienzo, D7).

## 3. Evidencia

| # | Pantalla | Qué se ve | De dónde sale |
|---|---|---|---|
| E1 | `#personal` → Datos y dispositivos | Chip de color sobre el borde, dos columnas, error en la esquina, H1 "Funcionarios" | [form/css.go](../form/css.go), [form/render_input.go:162-169](../form/render_input.go#L162-L169), [layout/rightpanel/rightpanel.go:110-112](../layout/rightpanel/rightpanel.go#L110-L112) |
| E2 | `#appointment_booking` | Acordeón sin título de pantalla; campos a mano con etiqueta arriba y otra hoja de estilos; registro en un modal con un tercer estilo | [booking_flow.go:370-374](../../veltylabs/modules/appointment_booking/ui/booking_flow.go#L370-L374), [step_patient.go:37-46](../../veltylabs/modules/appointment_booking/ui/step_patient.go#L37-L46), [ui/css.go:39-55](../../veltylabs/modules/appointment_booking/ui/css.go#L39-L55) |
| E3 | idem | Los campos a mano duplican modelos que ya declaran sus widgets | [patient_directory/model.go:16-40](../../veltylabs/modules/patient_directory/model.go#L16-L40) |
| E4 | `#personal` | Etiqueta "Name" en inglés: el modelo no trae `Label` | [model/field.go:83-92](../model/field.go#L83-L92) |
| E5 | todos | 4 jerarquías de título: H1 degradado (`rightpanel`), H3 (`stepaccordion`), H3 sueltos (`payment_manager/ui/browser.go:150`), H1 sueltos (`room_layout/ui/stage*.go`) | — |
| E7 | 4 módulos | `<input>` a mano: appointment_booking, payment_manager, chat_room, room_layout | `grep -rn 'Input("' veltylabs/modules/*/ui` |
| E8 | components | `actionbutton.ActionButton.Variant` es `string`; `widget/style` no tiene receta de título | [components/actionbutton](../components/actionbutton/README.md) |

**Causa raíz**: `UIModule.View()` devuelve un `dom.Component` libre y el título vive *dentro* del
contenido (`rightpanel.Title`), así que cada módulo decide si lo tiene y cómo; y `form` no es el único
camino para un campo porque nada en el contrato lo hace más fácil que escribir `Input(...)`.

## 4. Compuerta api-design

1. **Prior art.** Flutter `Scaffold` / Android `TopAppBar` (el título de pantalla es del armazón, la
   pantalla solo aporta `body`); Shopify Polaris `Page` + `FormLayout` (el producto compone, no estiliza
   campos); Django admin / Rails scaffold (el formulario se genera del modelo). Aquí `form.New(model)` y
   `UIModule.Label()` ya existen: falta que sean el **único** camino para campo y título.
2. **Novice-name.** Sin nombres nuevos en `form` ni en `platformd` (el título sale de `Label()`). Nuevo:
   `style.Heading()` (D12); `crudview.Config.Scope` reemplaza a `Context` (D15); `actionbutton.Variant` tipado. Se borran nombres, no se agregan.
3. **Ledger.**
   ```
   Conceptos que aprender          −3  (Context, HeadControls, decktabs para agrupar)
   Archivos para un form nuevo     −1  (css.go del módulo para sus campos)
   Líneas en el módulo de reserva  −~120 (textField, modal, PartFlow*, css de campos)
   Formas de hacer lo mismo        campo 3→1 · título 4→1 · agrupar pantallas 2→1
   ```
4. **Dónde vive.** Forma del campo: `form`. Título de módulo: `platformd`. Receta de título de sección:
   `widget/style`. Agrupar pantallas: `platformd.NewGroupUIModule`. Piezas propias: el módulo (D7).
5. **Qué borra.** `rightpanel.Title` y su fila de cabecera; `rightpanel.HeadControls`;
   `crudview.Config.Context` (pasa a `Scope`); `components/decktabs`; `form` `labelChars` + truncado, chip
   `OnEdge`, `Grid(2, …)`; en módulos: `textField`, `PartFlowField/Input/Check`, `requireSpecialty`,
   `specialtyPicker`, el `modaldialog` del paso 4, `personal.go` con `decktabs`.

## 5. Librerías comprometidas y orden

Ola 1 no depende de nada; cada ola siguiente espera el tag publicado de la anterior.

| Ola | Librería | Cambio | Decisiones | PLAN.md |
|---|---|---|---|---|
| 1 | `webtyp/widget` | `style.Heading()` | D12 | **publicado v0.6.41** (PR #8) |
| 1 | `webtyp/form` | etiqueta arriba, una columna, ayuda/error abajo, validar al salir, foco primario, aviso sin `Label` | D2–D4, D10, D11, D17 | **publicado v0.4.41** (PR #26) |
| 1 | `webtyp/layout` | título del módulo en el header (barra primaria, `<h1>`), `rightpanel` sin banda de título, `crudview` `Context` → `Scope` en la cabecera de la lista | D6, D12–D15 | escrito, **retenido** (ver Pendientes) |
| 2 | `webtyp/components` | títulos con `Heading()`, `actionbutton.Variant` tipado, borrar `decktabs` | D12, D16 | **publicado v0.8.31** (PR #33) |
| 3 | `veltylabs/modules/item_catalog` (referencia) | `specialty_id` como select en el form; borrar `requireSpecialty`/`specialtyPicker`; `Label` en todos los campos con widget | D5, D10, D15 | **publicado v0.3.76** (PR #11) |
| 3 | `veltylabs/modules/appointment_booking` | Personal → grupo; reserva con `form.New`; registro en línea; `Scope` en Horario/Servicios; sobrecupo en form de `CalendarConfig` | D5, D6, D8, D15, D16 | pendiente: tags de form, layout, components |
| 3 | `staff_manager`, `patient_directory` y demás con `crudview` | `Label` en todo campo con widget | D10 | pendiente: tag de form |
| 3 | `chat_room`, `business_calendar`, `payment_manager` | `decktabs` → grupo; inputs a mano → `form` | D5, D16 | pendiente: tags de form y components |

## 6. Relación con otras olas

- **Independiente** de SECURITY_BY_DEFAULT y API_SURFACE.
- **Comparte `form`/`dom`** con NO_REFLECTION_MASTER_PLAN (su ola 1 tocó `form`; hoy `form/docs/` no
  tiene PLAN.md en cola): antes de escribir el PLAN.md de `form` se verifica que esa ola esté publicada.
- **Extiende SOURCE_SELECTION** (D10: claves en inglés, `Label`/`Help` en los modelos).
- **Cambia API publicada** de `layout` (`rightpanel.Title`, `HeadControls`, `crudview.Config.Context`) y
  `components` (`decktabs`): sin consumidores externos fuera de `veltylabs`, se renombra/borra en esta ola (api-design gate 2).

## 7. Preguntas abiertas

_(ninguna)_

## 8. Pendientes

- **Seguimiento cada 15 min: detenido el 2026-10-14 a pedido del usuario.** Lo que sigue en vuelo se revisa a mano. Intentos de feedback por fallo, máximo 2: widget cerrado sin fallos · form cerrado (1 intento usado; en la revisión quité 7 archivos sueltos del agente y resolví un conflicto con `main` en `docs/PLAN.md`) · components cerrado sin fallos · item_catalog cerrado (1 intento usado; en la revisión agregué el log de error al cargar especialidades) (los `codejob reply` enviados fueron cambios de alcance, no fallos). Fuera de la ola: `webtyp/devflow` (arreglo de `codejob close`) **publicado v0.4.138** (PR #48; agregué en la revisión la aserción de que no vuelve a mergear).

- **`webtyp/layout` tiene cambios sin commit que no son de esta ola** (`platformd.go`, `README.md`,
  `go.mod`: el selector de tema pasa al menú de usuario, siguiendo el PR #32 de components).
  `codejob dispatch` publica todo el árbol de trabajo, así que **no despaché layout**. Confirmar si esos
  cambios se publican (o se descartan) y luego `codejob dispatch` en `webtyp/layout`.
- **components PR #32 cerrado a mano**: estaba mergeado pero sin publicar y `codejob close` falló porque la rama del PR ya no existía (defecto de `devflow`: `Close` revisa los commits del PR antes de detectar que ya está mergeado). Borré su `docs/PLAN.md` y publiqué con `gopush` (v0.8.30). El arreglo de `devflow` está publicado (v0.4.138).
- **`webtyp/widget/docs/PLAN.md` era un plan ya ejecutado** (receta `AppBar`, commit `9207a13`): lo pasé a
  `docs/LAST_PLAN_EXECUTED.md` para liberar `PLAN.md`.
- **D16, segunda parte, se tomó por aceptada** (dejaste la pregunta sin responder): `payment_manager` y
  `chat_room` entran; `room_layout` queda fuera. Corrígelo aquí si no es así.
- Los planes de módulos (ola 3) se escriben cuando estén publicados los tags de su base, para fijar versiones exactas.

