# Consistencia visual entre módulos — master plan
> **Status:** PENDING · 2026-10-14 · D1–D10 cerradas; abiertas P11–P16; ningún PLAN.md escrito

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
- **D9** (cierra P9) — **`chipgrid` implementa `form.Renderer`**: es el control "uno entre pocas opciones"
  dentro de un formulario (mismo marco de campo, etiqueta arriba); el radio queda para listas largas o booleanas.
- **D10** (cierra P10) — **Todo campo con widget declara `Label`** (inglés, se traduce) y `form` emite un
  **diagnóstico ruidoso en desarrollo** si falta, en vez de mostrar el nombre del campo (hoy se ve "Name"
  en `#personal`; `model.Field.Label` existe en [model/field.go:83-92](../model/field.go#L83-L92)).

## 3. Preguntas abiertas

### P11 — El "rasgo de identidad" (pediste explicación de la P1)

Al quitar el chip de color (D2), los formularios quedan con aspecto neutro: etiqueta en texto y caja con
borde fino. El chip era lo único que hacía reconocible "un campo de esta app". La pregunta es si
conservamos **un** detalle de marca en el campo, y cuál:

- **A. Color primario solo en el foco**: el campo que se está editando pinta su borde y su etiqueta en el
  color primario; el resto, neutro. **Recomendada**: la marca aparece donde aporta (dice "estás aquí"), no
  compite con el botón primario, y no agrega elementos.
- **B. Ninguno**: foco con el anillo estándar del navegador/tema. Más simple, pero la app pierde el único rasgo propio de sus formularios.
- **C. Etiqueta siempre en color primario.** Mantiene identidad, pero con 8 campos son 8 manchas de color que vuelven a competir con el botón (el defecto del chip).

### P12 — Jerarquía de encabezados (propuesta según D6)

Con el título del módulo en el header, dentro del contenido ya no hay título de pantalla. Propuesta:

- **A. Un único nivel dentro del contenido.** Nivel 1 = nombre del módulo en el header (`platformd`,
  semánticamente el `<h1>` de la página, único). Dentro del contenido solo existe **título de sección**
  (`<h2>`): paso del acordeón, cabecera de tarjeta, grupo de campos, panel final de la reserva. Una sola
  receta `style.Heading()` en `widget/style` que todos los componentes usan; ningún componente ni módulo
  fija tamaño/peso de un título a mano. **Recomendada**: dos tamaños de título en toda la app, uno por zona; imposible equivocarse.
- **B. Dos niveles dentro del contenido** (sección y subsección). Solo si algún módulo tiene secciones
  anidadas; hoy ninguno (el más profundo es el acordeón de la reserva, de un nivel).

### P13 — Fondo primario del header en escritorio

D6 pide fondo primario en la esquina superior izquierda (marca + módulo). Opciones:

- **A. Toda la barra del header en color primario** (como ya es en móvil), con usuario y estado en blanco
  a la derecha. **Recomendada**: un solo header en ambos anchos; un bloque de color que termina a mitad de la barra crea un borde que hay que alinear con el rail y el contenido en cada ancho.
- **B. Solo el bloque izquierdo** `[icono] [app] / [módulo]` en primario; el resto del header neutro, como hoy.

### P14 — Qué nombre se muestra cuando el módulo es hijo de un grupo

En `item_catalog`, el grupo es "Catálogo" y la pantalla activa "Servicios".

- **A. `[app] / Servicios`** (solo la pantalla activa), igual que móvil, que muestra "Servicios". **Recomendada**: el grupo ya se ve resaltado en el rail; repetirlo alarga el header sin informar.
- **B. `[app] / Catálogo / Servicios`** (ruta completa). Más orientación en grupos grandes, más largo.

### P15 — Dónde va un control que filtra la lista (no es un campo del registro)

D6 manda los controles al formulario. Hay dos tipos y conviene separarlos:

1. **Valor del registro**: en `item_catalog` el selector "Especialidad" junto al título es en realidad el
   campo `specialty_id` de `CatalogItem` ([item_catalog/model.go:69](../../veltylabs/modules/item_catalog/model.go#L69));
   como no es campo del form, el módulo envuelve al presenter para inyectarlo al guardar
   (`requireSpecialty`, [item_catalog/ui/browser.go:14-60](../../veltylabs/modules/item_catalog/ui/browser.go#L14-L60)).
   → **Va al formulario** como `input.Select()` y se llena con `crudview.SetOptions`; se borran
   `requireSpecialty` y `specialtyPicker`. (Consecuencia directa de D5/D6, no pregunta.)
2. **Filtro de alcance**: en Horario y Servicios de Personal, el selector de profesional
   ([scheduleview.go:196-205](../../veltylabs/modules/appointment_booking/ui/scheduleview.go#L196-L205),
   [serviceconfigview.go:57,74](../../veltylabs/modules/appointment_booking/ui/serviceconfigview.go#L57))
   decide **qué lista** se ve, no un valor que se guarda. Para estos:
   - **A. En la cabecera de la lista** (`rp__aside-header`, junto al buscador): el filtro queda pegado a
     lo que filtra; en móvil la lista es la primera vista, así que se ve primero. **Recomendada.**
   - **B. Primer campo del formulario.** Mezcla "qué estoy viendo" con "qué estoy guardando": cambiar el profesional parecería editar el registro.

   El "Máx. sobrecupo/día" de Horario es un campo de `CalendarConfig` → va a un formulario de ese modelo (D5).

### P16 — Alcance de la migración en esta ola

- **`decktabs`**: además de Personal lo usan `chat_room` ([ui/browser.go:88](../../veltylabs/modules/chat_room/ui/browser.go#L88), [ui/groups.go:27](../../veltylabs/modules/chat_room/ui/groups.go#L27)) y `business_calendar`.
  - **A. Migrar los tres a `NewGroupUIModule` y borrar `components/decktabs`** (api-design: una forma de agrupar pantallas). **Recomendada.**
  - **B. Migrar solo Personal**; `decktabs` queda para pestañas *dentro* de una pantalla.
- **Inputs a mano fuera de `form`** (E7): `payment_manager` (comprobante, monto de reembolso) y
  `chat_room` (casillas de participantes) son formularios → **entran a la ola** (D5). `room_layout`
  (nombres y medidas editados sobre el plano) es edición sobre un lienzo, pieza propia del módulo → **queda fuera** (D7). ¿De acuerdo?

## 4. Evidencia

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

## 5. Compuerta api-design (borrador; se completa al cerrar P11–P16)

1. **Prior art.** Flutter `Scaffold` / Android `TopAppBar` (el título de pantalla es del armazón, la
   pantalla solo aporta `body`); Shopify Polaris `Page` + `FormLayout` (el producto compone, no estiliza
   campos); Django admin / Rails scaffold (el formulario se genera del modelo). Aquí `form.New(model)` y
   `UIModule.Label()` ya existen: falta que sean el **único** camino para campo y título.
2. **Novice-name.** Sin nombres nuevos en `form` ni en `platformd` (el título sale de `Label()`). Nuevo:
   `style.Heading()` (P12). Se borran nombres, no se agregan.
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
   `crudview.Config.Context`; `components/decktabs` (si P16 = A); `form` `labelChars` + truncado, chip
   `OnEdge`, `Grid(2, …)`; en módulos: `textField`, `PartFlowField/Input/Check`, `requireSpecialty`,
   `specialtyPicker`, el `modaldialog` del paso 4, `personal.go` con `decktabs`.

## 6. Librerías comprometidas y orden

Cada fila tendrá su `docs/PLAN.md` (inglés, skill **plan-authoring**) cuando se cierren sus preguntas;
despacho con `codejob`. Las hojas esperan el tag de su base (api-design → «fix at the root»).

| Orden | Librería | Cambio | Depende de | Decisiones / preguntas | PLAN.md |
|---|---|---|---|---|---|
| 1 | `webtyp/widget` (style) | Receta `Heading()` | — | P12 | pendiente |
| 2 | `webtyp/form` | Formato único (D2–D4), foco (P11), `Label` obligatorio (D10) | 1 | D2, D3, D4, D10, P11 | pendiente |
| 3 | `webtyp/components` | `chipgrid` como `form.Renderer` (D9); `stepaccordion`/`contentcard`/`modaldialog` usan `Heading()`; `actionbutton.Variant` tipado; borrar `decktabs` (P16) | 1, 2 | D9, P12, P16 | pendiente (hay un `docs/PLAN.md` en cola: se convierte en cola de ejecución) |
| 4 | `webtyp/layout` | `platformd`: título del módulo en el header en escritorio (D6, P13, P14); `rightpanel`: sin fila de título ni `HeadControls`, filtro de alcance en la cabecera de la lista (P15); `crudview`: sin `Context` | 1 | D6, P13, P14, P15 | pendiente |
| 5 | `veltylabs/modules/item_catalog` | **Referencia**: `specialty_id` como select en el form; borrar `requireSpecialty` y `specialtyPicker` | 2, 4 | D5, D6 | pendiente |
| 6 | `veltylabs/modules/appointment_booking` | Personal → `NewGroupUIModule`; reserva con `form.New` (D5), registro en línea (D8), `chipgrid` en form (D9); Horario/Servicios sin `HeadControls` | 2, 3, 4 | D5, D6, D8, D9, P15 | pendiente |
| 7 | `staff_manager`, `patient_directory` y demás con `crudview` | `Label` en todos los campos con widget (D10) | 2 | D10 | pendiente |
| 8 | `chat_room`, `business_calendar`, `payment_manager` | `decktabs` → grupo; inputs a mano → `form` | 2, 3, 4 | P16 | pendiente |

## 7. Relación con otras olas

- **Independiente** de SECURITY_BY_DEFAULT y API_SURFACE.
- **Comparte `form`/`dom`** con NO_REFLECTION_MASTER_PLAN (su ola 1 tocó `form`; hoy `form/docs/` no
  tiene PLAN.md en cola): antes de escribir el PLAN.md de `form` se verifica que esa ola esté publicada.
- **Extiende SOURCE_SELECTION** (D10: claves en inglés, `Label`/`Help` en los modelos).
- **Cambia API publicada** de `layout` (`rightpanel.Title`, `HeadControls`, `crudview.Config.Context`) y
  `components` (`decktabs`): sin consumidores externos fuera de `veltylabs`, se renombra/borra en esta ola (api-design gate 2).
