# Orden de ejecución local (usabilidad, búsqueda, módulos configurables, módulos sin `appkit`) — master plan
> **Status:** IN PROGRESS · 2026-11-16 · bloque A ✅ publicado; bloque B: pasos 8–9 ✅; **espera al dueño**: revisar el plan [record_search_phase2/PLAN.md](record_search_phase2/PLAN.md) (paso 10, Sonnet se detiene ahí)

Orquesta, en orden, la ejecución **local** de cuatro masters: [PATIENT_DIRECTORY_USABILITY](PATIENT_DIRECTORY_USABILITY_MASTER_PLAN.md), [RECORD_SEARCH](RECORD_SEARCH_MASTER_PLAN.md), [ARCHITECTURE](ARCHITECTURE_MASTER_PLAN.md) (solo D62) y [CONFIGURABLE_MODULES](CONFIGURABLE_MODULES_MASTER_PLAN.md). Aquí no se decide nada: cada paso apunta a la decisión (`D<n>`) del master de su dominio, que es la especificación.

**Protocolo del ejecutor (Sonnet) — leer completo antes de cada paso:**
1. Leer las skills **master-plan** («Local execution across repos»), **testing**, **zero-debt**, **bug-fix-flow** y las que nombre el paso. Las skills son la única fuente de cómo trabajar (`webtyp/devskills/skills`).
2. Tomar el **primer** paso de «Fases» cuyo estado sea ⬜ y cuyas dependencias estén ✅. Si el paso es de **Opus** o del **Dueño**, o es una puerta 🔵, **detenerse** y avisar en una línea; no saltarlo.
3. Antes de tocar código: escribir al inicio de su `Estado` `[Sonnet · AAAA-MM-DD HH:MM] 🟡` y guardar (`gopush --no-cascade` en `webtyp/docs`). Marca de otro agente con menos de 90 min → no tocar.
4. Leer la decisión citada en el master de dominio y el código que nombra (`archivo:línea`). Trabajar **solo** en los repos del paso. Un consumidor que necesita el cambio usa `replace` a la copia local; **no publicar** (`gopush`, tags) hasta la puerta 🔵.
5. Verificar: `gotest` en cada repo tocado. Si el paso dice «pantalla»: MCP `webtyp` → `restart_development` del módulo, `browser_save_screenshot` a 972 px y en `iPhone SE` (320 px), y mirar la captura.
6. Al terminar: `Estado` = `✅ local · <cómo se verificó, una línea>`, borrar la marca, actualizar la fila del master de dominio con lo mismo, guardar (`gopush --no-cascade` en `webtyp/docs`).
7. Algo que la decisión no dice, una prueba que no se puede poner en verde, o un hallazgo de otro dominio → **no inventar**: una línea en «Bloqueos» con fecha, el paso queda 🟡 con `⚠`, y parar.
8. **Puerta 🔵**: el dueño revisa los ✅ del bloque. Solo con su «publica»: publicar **de abajo hacia arriba en el orden que lista la puerta** (si ese orden contradice los `go.mod`, manda el `go.mod`: primero lo que otros importan) (`gopush` en cada repo), quitar cada `replace`, verificar que ningún `go.mod` conserve `replace`, y pasar el repo al párrafo «Cerrado» de su master de dominio con su versión.

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

Ninguna propia: cada paso ejecuta una decisión de su master de dominio.

## Fases

Marcas: ⬜ por hacer · 🟡 en curso · ✅ hecho en local · 🔵 puerta del dueño (revisar y decir «publica») · 🧠 lo hace Opus (diseño o plan; Sonnet se detiene) · 🚫 bloqueado. Rutas: `webtyp/<repo>`, `veltylabs/modules/<repo>`.

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| **A** | | **Bloque A — usabilidad de `patient_directory`, segunda ola (sin dependencias)** | |
| 1 | `input` | PATIENT D18: traducción de las opciones de sexo en `input/lang.json`; skills **langc**, **translations** | ✅ local · `Male`/`Female` en `input/lang.json` (8 idiomas), `langc sync` sin avisos, `gotest` verde |
| 2 | `form`, `layout/crudview` | PATIENT D16 (el «+» enfoca el primer control enfocable; para un grupo de radios, la opción marcada o la primera) y D23 (elegir un radio con el puntero lleva al siguiente campo visible; flechas no saltan). Una sola función «siguiente control enfocable». Prueba con un grupo de radios primero. Pantalla. Skills **form-codegen**, **dom-elements**, **ui-consistency** | ✅ local · `form.focusFrom` única regla (Focus y radio por puntero), pruebas shared + wasm, `gotest` verde en `form` y `layout` (`replace` a `../form` en `layout/go.mod`, quitar en puerta A); captura de pantalla pendiente al paso 6 |
| 3 | `components/searchbar`, `layout/crudview` | PATIENT D15: Enter busca, vaciar no busca, aviso una vez. Pantalla. Skill **components** | ✅ local · Enter busca (`searchbar` + `widget.TypingAware`), aviso una vez en `crudview` (lang 8 idiomas), pruebas wasm y shared, `gotest` verde en `components` y `layout`; `replace` locales a `widget`/`components`/`form` (quitar en puerta A); captura al paso 6 |
| 4 | `form`, `components/agedate`, `components/searchbar` | PATIENT D19: el panel Issues de Chrome queda vacío (etiquetas, `aria-labelledby` + `role=group`, `autocomplete`). Pantalla | ✅ local · etiquetas, `role=group`+`aria-labelledby` en control propio (el título del grupo es `span`, no `label`), lupa de `searchbar` con `for`, `autocomplete` por tipo/`off`; `gotest` verde en `form`, `components`, `layout`; Issues de Chrome se mira en paso 6 |
| 5 | `components/tapinfotip` | PATIENT D20: zona táctil ≥ 24 px sin cambiar lo que se ve. Pantalla | ✅ local · receta nueva `style.HitArea()` en `widget/style` (zona 24 px por padding+margen negativo) usada en `tapinfotip`; `gotest` verde en `widget` y `components`; captura al paso 6 |
| 6 | `patient_directory` | PATIENT D22 (orden número → nombre → tipo → país) y D17 (verificar que el error del número se vea al guardar un pasaporte con tipo RUT). `replace` a 1–5. Pantalla | ✅ local · D22 (corregida por el dueño: número → tipo → país → «Nombre completo» con ayuda) en `definition.go` + `definitionc`; defecto de framework cerrado: `form` ignoraba el `Permitted` del campo (rechazaba «-» en el número; ahora `model.Field.PermittedOver` + `input.SetPermitted`, prueba `field_permitted_test.go`); D17 prueba nueva; `gotest` verde; pantalla 972/320 px en `patient_directory/docs/img/blockA-*.png`: «Pasaporte» lleva el foco al país emisor, DOM sin controles sin etiqueta/`autocomplete` |
| 7 | — | ✅ Puerta A publicada 2026-11-16: `model` v0.2.23 · `input` v0.2.4 · `widget` v0.6.49 · `components` v0.8.75 · `form` v0.4.68 · `layout` v0.3.126 · `patient_directory` v0.1.114 (sin `replace`; se quitaron también los previos de `crud`/`storage` con `crud` v0.0.16 y `storage` v0.2.7 porque `crud` tiene un plan corriendo sin tag) | ✅ |
| **B** | | **Bloque B — búsqueda de registros (RECORD_SEARCH)** | |
| 8 | `fmt` | RECORD_SEARCH D2: la función que pasa un texto a su forma de búsqueda `ToSearchText()` (RECORD_SEARCH D9), con pruebas de cada ejemplo de D2 y D5 | ✅ local · `fmt/search_text.go`: palabras de dígitos se juntan solo si las separan espacios, un signo («·») corta el campo (concilia D2 y D5); pruebas + idempotencia; `gotest` verde |
| 9 | — | Plan de la fase 2 de RECORD_SEARCH: [record_search_phase2/PLAN.md](record_search_phase2/PLAN.md) (verificado: todas las escrituras pasan por `orm.DB`; la columna vive fuera del modelo; reconstrucción en Go en `ddl.Sync`; D7 sin código en `dbsync`; dos defectos de motor hallados en `storage/mem` e `indexdb`) | ✅ Opus · código de 10a/10c/10d/10f/10h compilado y probado en copia temporal |
| 10 | — | 🔵 **El dueño revisa el plan** (su `## Design gate`: nombres nuevos y la firma de `Search` sin `text`) y escribe 🔵 aquí. Sin 🔵 Sonnet se detiene | ⬜ espera al dueño |
| 10a | `model` | [STAGE_10A_MODEL](record_search_phase2/STAGE_10A_MODEL.md): `FieldTextSearch`, `SearchableText`, `TextSearchFields/Of`, `Definition.ListsOnlyAfterSearch`, `SearchFirst` (D3–D5, D8) | ⬜ depende de 10 |
| 10b | `input` | [STAGE_10B_INPUT](record_search_phase2/STAGE_10B_INPUT.md): `Text`, `Phone`, `Email`, `Rut` entran en `text_search` (D4) | ⬜ depende de 10a |
| 10c | `storage` | [STAGE_10C_STORAGE](record_search_phase2/STAGE_10C_STORAGE.md): defecto de `mem` (lee por esquema, no por columnas pedidas) + cláusula de conformidad; `RewriteTextSearch` (D2) | ⬜ depende de 10a |
| 10d | `orm` | [STAGE_10D_ORM](record_search_phase2/STAGE_10D_ORM.md): `Create`/`Update`/`UpdateFields` mantienen `text_search` (D1, D2) | ⬜ depende de 10c |
| 10e | `indexdb` | [STAGE_10E_INDEXDB](record_search_phase2/STAGE_10E_INDEXDB.md): leer columnas pedidas; editar no borra `text_search` (D7) | ⬜ depende de 10c |
| 10f | `ddl` | [STAGE_10F_DDL](record_search_phase2/STAGE_10F_DDL.md): columna en `Sync`, reconstrucción en Go cuando cambia la lista de campos (D1, D2) | ⬜ depende de 10c |
| 10g | `definitioncheck`, `definitionc` | [STAGE_10G_GENERATOR](record_search_phase2/STAGE_10G_GENERATOR.md): nombre reservado (D3), genera `ListsOnlyAfterSearch()` (D8), `Search` sin `text` (D6) | ⬜ depende de 10a |
| 10h | `crud` | [STAGE_10H_CRUD](record_search_phase2/STAGE_10H_CRUD.md): búsqueda por defecto en `text_search`, la regla solo agrega (D6), lista vacía sin texto (D8) | ⬜ depende de 10a, 10c **y de que se cierre `crud/docs/PLAN.md`** (Jules, subida de archivos) |
| 10i | `view`, `layout/crudview` | [STAGE_10I_SCREEN](record_search_phase2/STAGE_10I_SCREEN.md): «Escribe para buscar» (D8) | ⬜ depende de 10a |
| 10j | `chat_room`, `editorial`, `room_layout`, `service_catalog` | [STAGE_10J_MODULES](record_search_phase2/STAGE_10J_MODULES.md): sus reglas `Search` pierden `text` (D6) | ⬜ depende de 10g, 10h |
| 11 | `indexdb`, `sqlite`, `postgres` | RECORD_SEARCH fase 3: prueba de conformidad, misma búsqueda y mismo resultado en los tres (incluye la cláusula nueva de 10c en `sqlite` y `postgres`) | ⬜ depende de 10a–10j |
| 12 | `patient_directory` | PATIENT fase 14 / RECORD_SEARCH fase 4: `ListsOnlyAfterSearch: true` y `Rules.Read: model.HaveNoBusinessRule` en `definition.go`, borrar `rule_patient_search.go` (con `isPhone`/`isDocument`/`digitsOnly` si quedan sin uso), `definitionc` local (10g); la migración rellena `text_search` de las 45 973 filas; «lucia» encuentra «Lucía», «13047980» encuentra «13.047.980-0». Pantalla: «Escribe para buscar» al abrir | ⬜ depende de 11 |
| 13 | — | 🔵 Puerta B. Publicar en orden: `fmt` → `model` → `input` → `storage` → `ddl` → `definitioncheck` → `crud` → `orm` → `definitionc` → `indexdb`/`sqlite`/`postgres` → `view` → `layout` → `chat_room`/`editorial`/`room_layout`/`service_catalog` → `patient_directory`; luego borrar `docs/record_search_phase2/` | 🔵 depende de 12 |
| **C** | | **Bloque C — módulos sin `appkit` (ARCHITECTURE D62) y lo que esperaba a `appkit`** | |
| 14 | `appkit` | PATIENT D21: terminar los cambios locales sin publicar (`App.Countries` → `Deps.Countries`/`BrowserDeps.Countries`; `app.go`, `serve.go`, `module_server.go`, `build_modules.go`, `screens.go`) hasta `gotest` verde; si necesita versiones locales sin publicar de otra librería (`mcp`, `server`), `replace` y anotarlo para la puerta C | ⬜ depende de 7 |
| 15 | `patient_directory`, `view`, `appkit` | PATIENT D24: `Deps.DefaultDocument`, el formulario nuevo parte con ese tipo | ⬜ depende de 14 |
| 16 | — | 🧠 Opus: plan de ARCHITECTURE D62 con su `## Design gate` (dónde va el contrato de construcción; compuerta `gocheck`); revisión del dueño 🔵 antes de ejecutar | 🧠 depende de 14 |
| 17 | `model`, contrato de construcción, `appkit`, `gocheck`, 8 módulos | ARCHITECTURE D62 según el plan del paso 16, un módulo por vez | ⬜ depende de 16 |
| 18 | — | 🔵 Puerta C. Orden de publicación: lo dice el plan del paso 16 | 🔵 depende de 15, 17 (publica también `appkit` del paso 14 y lo que este necesite, antes que los módulos) |
| **D** | | **Bloque D — módulos configurables (CONFIGURABLE_MODULES)** | |
| 19 | — | 🧠 Opus: plan de las fases 1–2 (rubros, íconos, maneras de uso, roles, vínculo cuenta → ficha: D1–D6) y capturas de D7 (formulario de otro módulo) y de «Ver como» rehecho con las maneras | 🧠 depende de 18 |
| 20 | `model`, `rbac`, `access`, `crud`, `view`, `appkit` | CONFIGURABLE fases 1–2 según el plan del paso 19 | ⬜ depende de 19 |
| 21 | `devtools`; `view`, `layout`, `appointment_booking` | CONFIGURABLE fases 3 y 5 («Ver como»; D7), tras aprobar sus capturas | ⬜ depende de 20 |
| 22 | `patient_directory` → `client_directory` y consumidores | CONFIGURABLE fase 4: renombrar repo, módulo, entidad y tabla (`gonew` para el repo nuevo; skill **api-design** paso 0) | ⬜ depende de 13, 21 |
| 23 | — | 🔵 Puerta D: orden de publicación según el plan del paso 19 | 🔵 depende de 22 |

## Bloqueos

- Ninguno.

## Relación con otras olas

- Ejecuta las decisiones de PATIENT_DIRECTORY_USABILITY, RECORD_SEARCH, ARCHITECTURE (D62) y CONFIGURABLE_MODULES; cada paso cerrado se refleja también en la fila de su master.
