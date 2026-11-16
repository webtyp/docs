# Orden de ejecución local (usabilidad, búsqueda, módulos configurables, módulos sin `appkit`) — master plan
> **Status:** IN PROGRESS · 2026-11-16 · sin preguntas; listo para Sonnet desde el paso 1 (bloques A y B sin bloqueos; pasos 🧠 los hace Opus)

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
| 1 | `input` | PATIENT D18: traducción de las opciones de sexo en `input/lang.json`; skills **langc**, **translations** | ⬜ |
| 2 | `form`, `layout/crudview` | PATIENT D16 (el «+» enfoca el primer control enfocable; para un grupo de radios, la opción marcada o la primera) y D23 (elegir un radio con el puntero lleva al siguiente campo visible; flechas no saltan). Una sola función «siguiente control enfocable». Prueba con un grupo de radios primero. Pantalla. Skills **form-codegen**, **dom-elements**, **ui-consistency** | ⬜ |
| 3 | `components/searchbar`, `layout/crudview` | PATIENT D15: Enter busca, vaciar no busca, aviso una vez. Pantalla. Skill **components** | ⬜ |
| 4 | `form`, `components/agedate`, `components/searchbar` | PATIENT D19: el panel Issues de Chrome queda vacío (etiquetas, `aria-labelledby` + `role=group`, `autocomplete`). Pantalla | ⬜ depende de 2, 3 |
| 5 | `components/tapinfotip` | PATIENT D20: zona táctil ≥ 24 px sin cambiar lo que se ve. Pantalla | ⬜ |
| 6 | `patient_directory` | PATIENT D22 (orden número → nombre → tipo → país) y D17 (verificar que el error del número se vea al guardar un pasaporte con tipo RUT). `replace` a 1–5. Pantalla | ⬜ depende de 1–5 |
| 7 | — | 🔵 Puerta A. Publicar en orden: `input` → `form` → `components` → `layout` → `patient_directory` | 🔵 depende de 6 |
| **B** | | **Bloque B — búsqueda de registros (RECORD_SEARCH)** | |
| 8 | `fmt` | RECORD_SEARCH D2: la función que pasa un texto a su forma de búsqueda `ToSearchText()` (RECORD_SEARCH D9), con pruebas de cada ejemplo de D2 y D5 | ⬜ |
| 9 | — | 🧠 Opus: plan de la fase 2 de RECORD_SEARCH (verificar dónde pasan **todas** las escrituras, `orm.Store` vs `crud`; paso de relleno en Go para `ddl`; D6–D8), en etapas para Sonnet, en este master como pasos 10a… | 🧠 depende de 8 |
| 10 | `model`, `definitionc`, `definitioncheck`, `ddl`, `orm`/`crud`, `dbsync`, `crudview` | RECORD_SEARCH D3–D8 según el plan del paso 9 | ⬜ depende de 9 |
| 11 | `indexdb`, `sqlite`, `postgres` | RECORD_SEARCH fase 3: prueba de conformidad, misma búsqueda y mismo resultado en los tres | ⬜ depende de 10 |
| 12 | `patient_directory` | PATIENT fase 14 / RECORD_SEARCH fase 4: `ListsOnlyAfterSearch: true`, borrar `rule_patient_search.go`; «lucia» encuentra «Lucía». Pantalla | ⬜ depende de 11 |
| 13 | — | 🔵 Puerta B. Publicar en orden: `fmt` → `model` → `ddl` → `definitioncheck` → `definitionc` → `crud` → `orm` → `sqlite`/`postgres`/`indexdb` → `dbsync` → `layout` → `patient_directory` | 🔵 depende de 12 |
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
