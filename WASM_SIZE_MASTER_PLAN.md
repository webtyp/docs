# Tamaño del binario wasm — master plan
> **Status:** PENDING · 2026-11-17 · sin preguntas; sigue la fase 1 (análisis, sin código): la hace el modelo que planifica y sus resultados los revisa el dueño antes de tocar una librería.

**Qué es:** bajar lo que pesa `browser.wasm` encontrando código repetido y librerías que hacen demasiado, y partiéndolas para que su código se reutilice. Sale de BUILD_DESTINATIONS (P13 de allí): separar backend y front por paquetes ordena el código pero no baja el tamaño.

**Cómo se mide (igual en cada pasada):** en `veltylabs/mjosefa-cms`, `tinygo build -target wasm -opt=z -panic=trap -tags browser -size full -o /tmp/b.wasm ./web/browser` da el peso por paquete; con `-no-debug` y `gzip -9` se obtiene el binario que se publica. Punto de partida (2026-11-17, 8 módulos, TinyGo 0.41.1): 1,44 MB, 472 KB gzip; 1,15 MB de código y datos.

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 | **El tamaño se baja encontrando código repetido y librerías que hacen demasiado, y partiéndolas para reutilizar** (dueño, 2026-11-17). Nunca un wasm por área de la app (duplica todo y complica) ni presupuestos de tamaño. Cada librería que cambie anota su peso antes y después | fase 1 (análisis), luego cada librería |
| D2 | **Quitar la sección `name` del wasm de producción**: son los nombres de las funciones, que `-no-debug` no borra; 133 KB de 1,44 MB (19 KB gzip), sin tocar código de ninguna librería | el compilador único de `browser.wasm` (BUILD_DESTINATIONS D22) |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| 1 | `mjosefa-cms` (medición), `room_layout`, `dom`, `form`, `input`, `view`, `layout` | 🧠 Análisis, sin código. Lo ya medido: la stdlib **no** es el problema (`unicode` 2,5 KB por `strings`; `sync`, `errors`, `sort` bajo 300 bytes). Por revisar, de mayor a menor peso: (a) `room_layout/ui` 107 KB, 2,4 veces el `ui/` siguiente (`appointment_booking/ui` 45 KB), 2 861 líneas en 4 etapas escritas a mano: qué repite que ya exista en `components` o `layout`; (b) «métodos de interfaz» 67 KB + tipos 28 KB: `dom.Reference` (`dom/reference.go`) tiene 32 métodos; ver qué parte usa cada consumidor y si interfaces más chicas lo bajan (medir antes de decidir); (c) qué repiten entre sí `dom` 62 KB, `form` 50 KB, `layout/platformd` 34 KB, `fmt` 32 KB, `layout/crudview` 31 KB, `components/scheduleeditor` 30 KB, `view` 28 KB, `input` 25 KB, `json` 23 KB; (d) 269 KB que TinyGo no atribuye a ningún paquete (186 KB son datos): nombrar de quién son; (e) herramientas de desarrollo en el `main` de producción (`devtools.Menu()` en `web/browser/main.go`: `layout/apiexplorer` 18 KB, `layout/schemaview` 6 KB, `erd`). Resultado: una pregunta o decisión por hallazgo, aquí; los defectos de diseño de una librería, en ARCHITECTURE_MASTER_PLAN | ⬜ por hacer |
| 2 | compilador de `browser.wasm` | D2 | ⬜ espera BUILD_DESTINATIONS fase 5 |

## Bloqueos
- 2026-11-17 · D2 espera que BUILD_DESTINATIONS D22 deje un solo lugar que compila `browser.wasm`.

## Relación con otras olas

- **BUILD_DESTINATIONS** (origen; D22 es el lugar donde se construye D2); **STDLIB_MIGRATION** y `gocheck/docs/STDLIB_WEIGHT.md` (peso de la stdlib, ya medido allí); **NO_REFLECTION** (tamaño por reflexión, olas 1–3 cerradas); **ARCHITECTURE** recibe los defectos de diseño que salgan de la fase 1.
