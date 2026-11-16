# Usabilidad de `patient_directory` — master plan
> **Status:** IN PROGRESS · 2026-11-16 · primera ola (edad, países, teléfono, documento) **publicada**; P15, P18, P19 cerradas (D15, D22–D24): segunda ola en ejecución local (EXECUTION_ORDER bloque A); búsqueda sin tildes espera que RECORD_SEARCH se construya

Dominio: lo que el funcionario ve y hace en `patient_directory`. Lo que no es de este dominio va al master que corresponda; **no se trabaja aquí**.

**Para retomar en un chat nuevo:** leer este archivo y las skills `ui-consistency`, `form-codegen`, `api-design` (nombres de 2–3 palabras: verbo + objeto). Entorno: `webtyp dev` en `veltylabs/modules/patient_directory` (MCP `restart_development`); la demo sale de `seed/demo.go`; el módulo no usa copias locales salvo `crud` y `storage` (son de la migración en curso). Probar con `gotest` en cada repo y, para pantalla, `browser_evaluate_js`/`browser_save_screenshot` con la app en 972 px y `iPhone SE` (320 px). Publicar solo con `gopush` y revisar después que ningún `go.mod` conserve `replace` (ya lo hace `gopush`, `devflow v0.4.195`). Si un hallazgo no es de este dominio: al master que corresponda, no a este.

**Cerrado (publicado, sin pendientes):** `fmt v1.0.2` (`RemoveAccents()`) · `model v0.2.22` (`Field.AlsoSetsField`, `Field.ShownWhen`/`model.When`) · `country v0.0.2` · `input v0.2.3` (`input.Phone()` único, `input.AlsoSetter`) · `widget v0.6.48` (`State.Hidden`, partes del teléfono) · `view v0.6.83` (`WithOptions`) · `components v0.8.74` (`agedate`: edad en años enteros ≥ 1 que marca sola «aproximada»; `tapinfotip`) · `form v0.4.67` (`ShownWhen`, `AlsoSetsField`, teléfono prefijo + número) · `layout v0.3.125` · `patient_directory v0.1.113` · `sitetheme v0.1.104`. El detalle de cada decisión vive en: `country/README.md`, `model/docs/API_FIELD.md`, `components/agedate/README.md`, `input/README.md` y las skills `ui-consistency` y `form-codegen`.

## Preguntas abiertas

Ninguna. P16–P17 (nombres sin tildes) pasaron a [RECORD_SEARCH_MASTER_PLAN.md](RECORD_SEARCH_MASTER_PLAN.md) P1–P2: es una decisión de framework, no de este módulo.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D15 (P15) | Enter dispara la búsqueda (`searchbar.go:96` deja `OnInput` por `OnKeyDown` Enter). **Vaciar la caja no busca** hasta Enter. Aviso «Press Enter to search» en el área de mensajes **una sola vez** por pantalla, la primera vez que se teclea, `view.Auto()` (4 s) | `searchbar`, `crudview` |
| D16 | El botón «+» deja el foco en el **primer control enfocable**, siempre y a nivel de framework. Causa probable (a verificar): `Form.Focus` apunta a `Inputs[0].GetID()` (`form.go:140`), que para un grupo de radios es el `div`, no un control; debe enfocar la opción marcada o la primera | `form`, `crudview` |
| D17 | Un pasaporte con el tipo RUT marcado da **error en el campo número** (regla del módulo `checkDocument`, `module.go:60-75`, que ya devuelve `FieldError{Field:"rut"}`); falta **verificar que el error se vea** al guardar | `patient_directory` |
| D18 | Las opciones de **sexo** (`Male`/`Female`, `input/gender.go`) no tienen traducción: agregarla en `input/lang.json` | `input` |
| D19 | `form` no deja avisos de HTML en Chrome. Auditoría (972 px): 4 `input` sin etiqueta (la fecha de `agedate` y 3 de `searchbar`: `aria-label`), 1 `label for` que apunta a un `div` (el control de `agedate`: usar `aria-labelledby` con `role=group`), 22 controles sin `autocomplete` (valor por tipo/nombre, `off` si no aplica). El panel Issues debe quedar vacío | `form`, `components` (`agedate`, `searchbar`) |
| D20 | Zona táctil del (i) de campo (14 px; mínimo 24 px, WCAG 2.5.8) sin cambiar lo que se ve | `components/tapinfotip` |
| D22 (P18, corregida 2026-11-16) | Orden: **número → tipo (RUT marcado) → país emisor → nombre**: el tipo va pegado al número porque lo valida; la etiqueta es **«Nombre completo»** con ayuda «nombres seguidos de apellidos» (un solo campo, nombre del paciente; neutro para el futuro `client_directory`). Tab va de número a tipo | `patient_directory` |
| D23 (P18) | **Elegir una opción de radio con el puntero lleva el foco al siguiente campo visible**, por defecto, sin configurar (lo dueño es `form`: dibuja el grupo en `render_input.go` y ya maneja el foco en `form.go:140`). Solo con clic/toque (`pointerType` mouse/touch/pen): las flechas del teclado siguen moviéndose **dentro** del grupo (patrón radio de WAI-ARIA) y no saltan. Si la opción revela campos (`ShownWhen`), el siguiente es el primero revelado; en el último campo no se mueve. Riesgo aceptado: WCAG 3.2.2 cuenta un cambio de foco como cambio de contexto; queda acotado al puntero | `form` |
| D24 (P19) | ~~`Deps.DefaultDocument` obligatorio~~ (reemplazado por ARCHITECTURE D74: el módulo lo deduce del país por defecto); el formulario nuevo parte con ese tipo; `view.New` expone el registro inicial (`crudview.CrudView.NewRecord`, `crudview.go:134`) | `patient_directory`, `view`, `appkit` |
| D21 | `appkit`: `App.Countries` → `Deps.Countries`/`BrowserDeps.Countries` (editado en local, sin publicar; EXECUTION_ORDER paso 14) | `appkit` |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| 9 | `components/searchbar`, `layout/crudview` | D15 | ✅ publicado `components` v0.8.75, `layout` v0.3.126 |
| 10 | `form`, `crudview` | D16 foco del «+» y D23 avance al elegir radio (misma lógica: «siguiente control enfocable»); prueba con grupo de radios primero | ✅ publicado `form` v0.4.68, `layout` v0.3.126 |
| 11 | `input` | D18: traducciones de sexo | ✅ publicado `input` v0.2.4 |
| 12 | `form`, `components` | D19: avisos de HTML; verificar con el script de auditoría y el panel Issues | ✅ publicado `form` v0.4.68, `components` v0.8.75 |
| 13 | `patient_directory`, `view`, `appkit` | D17 verificar; D22 orden; D24 tipo por defecto (`appkit` tras EXECUTION_ORDER paso 14) | D17 y D22 ✅ publicado `patient_directory` v0.1.114; D24 ✅ local (paso 15; publica la puerta C) |
| 14 | `patient_directory` | Adoptar la búsqueda de RECORD_SEARCH: `ListsOnlyAfterSearch: true` y **borrar** `rule_patient_search.go` (RECORD_SEARCH D6, D8: no le queda nada); EXECUTION_ORDER paso 12 | ✅ publicado `patient_directory` v0.1.115 (falta ver «Escribe para buscar» en pantalla) |
| 15 | `components/tapinfotip` | D20 | ✅ publicado `widget` v0.6.49, `components` v0.8.75 |
| 16 | `appkit` | D21 | ✅ local (EXECUTION_ORDER paso 14; `replace` a `country` y `view` hasta puerta C) |

## Bloqueos

- 2026-11-16 · «lucia» no encuentra «Lucía» hasta que RECORD_SEARCH publique su fase 3.

## Relación con otras olas

- RECORD_SEARCH: recibe P16–P17 (dato exacto y búsqueda, a nivel de framework); la fase 14 lo espera.
- CONFIGURABLE_MODULES: `patient_directory` → `client_directory`, rubro y roles; renombrar espera a que esta ola publique.
- EXECUTION_ORDER: ordena la ejecución local de este master; Sonnet marca allí y aquí cada paso.
