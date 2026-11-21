# Recursos de lo que no se ve: el framework los libera, una vez y para todos — master plan
> **Status:** PENDING · 2026-11-21 · esperan **P1, P2 y P3** del dueño

**Regla del dueño (2026-11-21):** como un auto en una ciudad abierta, donde los edificios del fondo se ven borrosos
porque no se cargan todas sus texturas, el framework libera lo que no se está viendo, **de forma centralizada**: nunca
un parche por componente. Y la memoria que se pide se mantiene al mínimo, para poder apagar el recolector de basura
en el futuro.

**Lo que se ve hoy:** en «Ocupación → Espacios» el desplazamiento va a tropezones. Medido: no es Go ni el wasm (los
eventos de desplazamiento cuestan 0,04–0,4 ms y ningún cuadro lento tiene tiempo de script) ni la disposición
(0,5 ms al moverse la barra del marco); queda el pintado. La página tiene **7.102 elementos** aunque la pantalla
muestra unos 1.950: `platformd` monta la vista de **todas** las pantallas al arrancar y solo muestra la actual
(`layout/platformd/platformd.go:700-735`, una sección por pantalla; `style.SlideDeck`: «todos los hijos siguen montados
en el DOM»). Las ocultas conservan su DOM, señales y suscripciones: memoria y trabajo que nadie ve.

Para retomar: skills **framework-map**, **dom-elements**, **widget-styling**, **wasm**, **api-design**; probar en
`veltylabs/mjosefa-cms` (MCP de webtyp; medir con la pestaña al frente: en segundo plano el navegador limita a 1 cuadro/s).

## Preguntas abiertas

**P1 — Qué pasa con una pantalla que no se está viendo.** A) **`platformd` monta la vista de una pantalla al abrirla
y la desmonta al salir** (terminada la animación de cambio): su DOM, señales y suscripciones se liberan; los datos
viven en los presenters y en `view.Records`, así que volver la rearma desde ahí. B) dejarla montada y que el navegador
no la pinte (`content-visibility: hidden`): no pinta, pero la memoria y las suscripciones siguen. **Recomiendo A**: es
la única que libera memoria (lo que pide apagar el recolector) y se hace una vez, en el marco; ningún componente
cambia. ¿Cuál evita el acoplamiento y es más mantenible? A: un solo lugar decide la vida de las vistas.
```go
// hoy — layout/platformd/platformd.go: cada pantalla montada para siempre
panel := Section().Set(clsPanel.AsAttr()).BindStateFunc(widget.Current, func() bool { return p.active.Get() == id })
panel.Child(m.View())
// A — la vista existe solo mientras su pantalla está activa (dom.Show ya desmonta y libera)
panel.Child(Show(p.isActive(id), func() *Element { return m.View().Render() }))
```
Respuesta:

**P2 — Lo que no se ve dentro de una pantalla grande** (dos plantas de 22×38, listas largas). A) **una receta de
`widget/style`, `SkipOffscreen()`, que emite `content-visibility: auto` y su tamaño estimado**: el navegador no
dibuja lo que está fuera de la vista (la textura borrosa del fondo) y lo dibuja al acercarse; la usan los contenedores
del framework (tarjeta de piso, fila de lista, panel) sin código en el módulo. B) virtualizar (crear solo las filas
visibles) en cada lista: más código y más memoria en movimiento. **Recomiendo A**: cero JS, una receta central; B solo
si una lista llega a miles de filas. ¿Cuál evita el acoplamiento y es más mantenible? A.
```go
// A — widget/style, una vez: lo usa el contenedor del framework, no cada pantalla
Part(PartFloor, style.Stack(style.Space3), style.SkipOffscreen())
```
Respuesta:

**P3 — Memoria al mínimo, camino a apagar el recolector.** A) **medir primero y fijar la regla**: el MCP de webtyp
reporta las asignaciones de memoria del wasm por acción (abrir pantalla, desplazar, guardar) y el skill **wasm** prohíbe
asignar en los caminos que se repiten (redibujar, manejar un evento): se reutilizan nodos y listas; con los números se
decide cuándo apagar el recolector. B) apagarlo ya y corregir lo que falle. **Recomiendo A**: sin medir no se sabe qué
asigna; apagarlo a ciegas agota la memoria. ¿Cuál evita el acoplamiento y es más mantenible? A.
```go
// A — devtools/MCP (solo en desarrollo): asignaciones de una acción
before := runtime.MemStats{}; runtime.ReadMemStats(&before) // … acción … ; reportar Mallocs y TotalAlloc del tramo
```
Respuesta:

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 | **La vida y el costo de lo que se ve los maneja el framework, una vez**: ningún componente ni módulo decide montar/desmontar, ni dibujar o no lo que está fuera de la vista (dueño) | skills **framework-map**, **components**, **wasm** |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| F1 | `layout/platformd` | P1 | tras P1 |
| F2 | `widget/style`, `layout`, `room_layout` | P2 | tras P2 |
| F3 | `devtools`/`app` (MCP), skill **wasm** | P3 | tras P3 |
| F4 | `room_layout` | medir de nuevo «Espacios» con F1 y F2; si sigue lento, la planta como un solo fondo (decisión en `ROOM_MAP_MASTER_PLAN.md`) | tras F1, F2 |

## Bloqueos

Ninguno.

## Relación con otras olas

- ROOM_MAP: el desplazamiento lento de «Espacios» espera F1/F2 (F4).
