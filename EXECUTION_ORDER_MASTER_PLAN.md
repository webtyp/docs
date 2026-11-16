# Orden de ejecución local (usabilidad, búsqueda, módulos configurables, módulos sin `appkit`) — master plan
> **Status:** IN PROGRESS · 2026-11-16 · bloques A y B ✅ publicados (puerta B); falta confirmar «Escribe para buscar» en pantalla y el relleno real de `text_search`; bloque E (4 módulos) listo para despachar con antigravity; bloque C: pasos 14 y 15 ✅ local, el 16 espera respuesta P54 (ARCHITECTURE)

Orquesta, en orden, la ejecución **local** de cuatro masters: [PATIENT_DIRECTORY_USABILITY](PATIENT_DIRECTORY_USABILITY_MASTER_PLAN.md), [RECORD_SEARCH](RECORD_SEARCH_MASTER_PLAN.md), [ARCHITECTURE](ARCHITECTURE_MASTER_PLAN.md) (solo D62) y [CONFIGURABLE_MODULES](CONFIGURABLE_MODULES_MASTER_PLAN.md). Aquí no se decide nada: cada paso apunta a la decisión (`D<n>`) del master de su dominio, que es la especificación.

**Protocolo del ejecutor (Sonnet) — leer completo antes de cada paso:**
1. Leer las skills **master-plan** («Local execution across repos»), **testing**, **zero-debt**, **bug-fix-flow** y las que nombre el paso. Las skills son la única fuente de cómo trabajar (`webtyp/devskills/skills`).
2. Tomar el **primer** paso de «Fases» cuyo estado sea ⬜ y cuyas dependencias estén ✅. Si el paso es de **Opus** o del **Dueño**, o es una puerta 🔵, **detenerse** y avisar en una línea; no saltarlo. Ningún paso espera a un agente externo (no hay: Jules cerró; todo es local); otro plan del mismo repo no bloquea, se hace uno tras otro.
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
| 9 | — | Plan de la fase 2 de RECORD_SEARCH: `record_search_phase2/PLAN.md` (borrado en la puerta B) (verificado: todas las escrituras pasan por `orm.DB`; la columna vive fuera del modelo; reconstrucción en Go en `ddl.Sync`; D7 sin código en `dbsync`; dos defectos de motor hallados en `storage/mem` e `indexdb`) | ✅ Opus · código de 10a/10c/10d/10f/10h compilado y probado en copia temporal |
| 10–11 | `model` v0.2.24 · `input` v0.2.5 · `storage` v0.2.8 · `orm` v0.12.42 · `indexdb` v0.7.20 · `ddl` v0.0.57 · `definitioncheck` v0.0.10 · `definitionc` v0.1.49 · `crud` v0.0.17 · `view` v0.6.86 · `layout` v0.3.127 · `sqlite` v0.3.43 · `postgres` v0.4.46 | RECORD_SEARCH fase 2 y 3 (plan aprobado por el dueño; los 4 módulos con regla `Search` pasan al bloque E) | ✅ publicado 2026-11-16 (puerta B) |
| 12 | `patient_directory` v0.1.115 | PATIENT fase 14 / RECORD_SEARCH fase 4: `ListsOnlyAfterSearch: true`, `Rules.Read: HaveNoBusinessRule`, `rule_patient_search.go` y sus ayudas borrados; prueba nueva «lucia»/«13047980»/teléfono; la migración de `Migrate` reconstruye `text_search` con `ddl.Sync` (prueba de migración usa `mem`) | ✅ publicado · pantalla 972/320 px en `patient_directory/docs/img/blockB-*.png`: abre vacía; «Escribe para buscar» aparece cuando el demonio dibuje con `layout` v0.3.127 (el módulo no importa `layout`); falta confirmar el relleno real de las 45 973 filas al correr `Migrate` en la base |
| 13 | `fmt` v1.0.3 y los de arriba | 🔵 Puerta B publicada 2026-11-16 (`gopush --no-cascade`, de abajo hacia arriba; ningún `go.mod` conserva `replace`; los consumidores no listados —`appkit` y demás— toman las versiones nuevas en su próxima publicación) | ✅ |
| **C** | | **Bloque C — módulos sin `appkit` (ARCHITECTURE D62) y lo que esperaba a `appkit`** | |
| 14 | `appkit` | PATIENT D21: terminar los cambios locales sin publicar (`App.Countries` → `Deps.Countries`/`BrowserDeps.Countries`; `app.go`, `serve.go`, `module_server.go`, `build_modules.go`, `screens.go`) hasta `gotest` verde; si necesita versiones locales sin publicar de otra librería (`mcp`, `server`), `replace` y anotarlo para la puerta C | ✅ local · `App.Countries` → `Deps.Countries`/`BrowserDeps.Countries`; `go.mod` sube `server` v0.2.127 y `mcp` v0.3.35 (publicados, traen `RawQuery`), `crud` v0.0.15, `router` v0.5.0; `replace` locales a `country` y `view` (quitar en puerta C, publicar antes `country` y `view`); `gotest` verde |
| 15 | `patient_directory`, `view`, `appkit` | PATIENT D24: `Deps.DefaultDocument`, el formulario nuevo parte con ese tipo | ✅ local · `view.WithNewRecord` + `Presenter.NewRecord()` (`crudview` lo usa si el host no pasa `Config.NewRecord`); `DefaultDocument` en `view.BrowserDeps`/`view.Demo`, `appkit` (`App`, `Deps`) y `patient_directory.Deps` (obligatorio, validado); `module/server.go` ahora también pasa `Countries`; `gotest` verde en `view`, `layout`, `appkit`, `patient_directory`; `replace` locales (quitar en puerta C): `layout`→`view`; `patient_directory`→`view`,`appkit`,`country`,`layout`; sin pantalla aún |
| 16 | — | 🧠 Opus: plan de ARCHITECTURE D62 con su `## Design gate` | 🧠 espera respuesta del dueño a P54 de [ARCHITECTURE](ARCHITECTURE_MASTER_PLAN.md); el plan se escribe después |
| 17 | `appkit`, 8 módulos, `access`, `mjosefa-cms` | ARCHITECTURE D62 según el plan del paso 16, un repo por vez | ⬜ depende de 16 |
| 18 | — | 🔵 Puerta C. Orden de publicación: «lo dice el plan del paso 16 | 🔵 depende de 15, 17 |
| **D** | | **Bloque D — módulos configurables (CONFIGURABLE_MODULES)** | |
| 19 | — | 🧠 Opus: plan de las fases 1–2 (rubros, íconos, maneras de uso, roles, vínculo cuenta → ficha: D1–D6) y capturas de D7 (formulario de otro módulo) y de «Ver como» rehecho con las maneras | 🧠 depende de 18 |
| 20 | `model`, `rbac`, `access`, `crud`, `view`, `appkit` | CONFIGURABLE fases 1–2 según el plan del paso 19 | ⬜ depende de 19 |
| 21 | `devtools`; `view`, `layout`, `appointment_booking` | CONFIGURABLE fases 3 y 5 («Ver como»; D7), tras aprobar sus capturas | ⬜ depende de 20 |
| 22 | `patient_directory` → `client_directory` y consumidores | CONFIGURABLE fase 4: renombrar repo, módulo, entidad y tabla (`gonew` para el repo nuevo; skill **api-design** paso 0) | ⬜ depende de 13, 21 |
| 23 | — | 🔵 Puerta D: orden de publicación según el plan del paso 19 | 🔵 depende de 22 |
| **E** | | **Bloque E — migrar a mano de antigravity los 4 módulos que 10j no pudo verificar (son otro dominio: `crud` actual, no búsqueda)** | |
| 24 | `chat_room`, `editorial`, `room_layout`, `service_catalog` | Tags de la puerta B ya escritos en los 4 planes (`grep '{{'` solo encuentra la frase de aviso). Antes de despachar: el `AGENTS.md` de cada repo (modificado, sin commit) debe estar en `main` | ✅ tags rellenados · falta el commit de `AGENTS.md` y despachar |
| 25 | `chat_room` | [PLAN](../../veltylabs/modules/chat_room/docs/PLAN.md): RuleAnswer en 12 reglas + 2 `Search` sin `text` (sin test que cambiar) | ⬜ plan escrito 2026-11-16 · despacha el dueño con antigravity · depende de 24 |
| 26 | `editorial` | [PLAN](../../veltylabs/modules/editorial/docs/PLAN.md): RuleAnswer en 4 reglas, `Search` sin `text`, prueba de `Migrate` lee por `storage.Conn` | ⬜ plan escrito 2026-11-16 · despacha el dueño con antigravity · depende de 24 |
| 27 | `service_catalog` | [PLAN](../../veltylabs/modules/service_catalog/docs/PLAN.md): RuleAnswer en 11 reglas, `Search` sin `text`, prueba de `Migrate`, sube `server`/`mcp` por `RawQuery` | ⬜ plan escrito 2026-11-16 · despacha el dueño con antigravity · depende de 24 |
| 28 | `room_layout` | [PLAN](../../veltylabs/modules/room_layout/docs/PLAN.md): RuleAnswer en 19 reglas, 4 `Search` sin `text`, `layout.Notifier/Auto/Duration` → `view`, sube `server`/`mcp` | ⬜ plan escrito 2026-11-16 · despacha el dueño con antigravity · depende de 24 |
| 29 | — | 🔵 Puerta E. El dueño revisa los 4 PR; `codejob close` de cada uno (no dependen entre sí) | 🔵 depende de 25–28 |

## Bloqueos

- 2026-11-16 · **10j resuelto por el bloque E**: los 4 módulos pinaban `crud` v0.0.11/12 (reglas con `return 0`, `ApplyAndPublish` sin paréntesis; `room_layout` además usa `layout.Notifier` ya movido a `view`; los que importan `appkit` arrastran `server`/`mcp` sin `RawQuery`). Verificado en copia temporal: con los cambios de cada plan, `gotest` verde en todos los destinos de los cuatro.
- 2026-11-16 · **Hallazgo de `ddl` (10f)**: `Sync` ahora exige un `storage.Conn` para una tabla con campos de búsqueda, así que la prueba de `Migrate` de cada módulo con un `Execer` que solo traga DDL falla (`editorial`, `service_catalog`); los planes 26 y 27 la arreglan con `mem.New()`. Si el dueño prefiere que `ddl` ofrezca ese doble de prueba en vez de repetirlo en cada módulo, es un cambio de 10f (otro paso de `ddl` antes de la puerta 13).

## Relación con otras olas

- Ejecuta las decisiones de PATIENT_DIRECTORY_USABILITY, RECORD_SEARCH, ARCHITECTURE (D62) y CONFIGURABLE_MODULES; cada paso cerrado se refleja también en la fila de su master.
