# Recursos de lo que no se ve: el framework los libera, una vez y para todos — master plan
> **Status:** IN PROGRESS · 2026-11-21 · F1–F4 🟢 completas en local, sin publicar; espera revisión del dueño para publicar en cascada (`widget`, `layout`, `devtools`, `devbrowser`, `room_layout`, `devskills`) y quitar `replace`; F4 confirmó que «Espacios» sigue lento (31,5 ms p50 / 70 ms p95) → decisión de planta en un solo fondo pasa a `ROOM_MAP_MASTER_PLAN.md`

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

Ninguna.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D2 (P1) | **`platformd` monta la vista de una pantalla al abrirla y la desmonta al salir** (terminada la animación de cambio): se liberan su DOM, señales y suscripciones; los datos quedan en presenters y `view.Records` | `layout/platformd` |
| D3 (P2) | **`widget/style.SkipOffscreen()`**: `content-visibility: auto` con su tamaño estimado; la usan los contenedores del framework (tarjeta de piso, fila de lista, panel), nunca cada pantalla | `widget/style`, `layout`, `room_layout` |
| D4 (P3) | **Medir antes de apagar el recolector**: el MCP de webtyp reporta la memoria pedida por acción (abrir pantalla, desplazar, guardar); el skill **wasm** prohíbe pedir memoria en lo que se repite | `devtools`/`app`, skill **wasm** |
| D1 | **La vida y el costo de lo que se ve los maneja el framework, una vez**: ningún componente ni módulo decide montar/desmontar, ni dibujar o no lo que está fuera de la vista (dueño) | skills **framework-map**, **components**, **wasm** |

## Fases

Marcas: 🔴 pendiente, sigue · ⚪ pendiente, espera otra fase · 🟡 en curso · 🟢 hecho en local y verificado, sin publicar.

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| F1a | `layout/platformd` | D2: cada pantalla se monta solo mientras está activa o saliendo (`screenPanel`, `release`, `p.leaving`, `leaveTimer`, `leaveMillis = 250`, que `leave_millis_test.go` ata a `css.DurationBase`, porque `webtyp.com/css` no compila a wasm). `dom` desmonta limpiando todo y vuelve a llamar `Init` al regresar; los 12 `Init` de pantallas se pueden repetir (revisado). Skill **components** ya lo dice (local, con línea STATUS) | 🟢 en vivo en mjosefa-cms: 7.102 → 2.330 elementos, solo la pantalla activa queda montada, ir a «Recepción» y volver la muestra con sus datos, sin errores de JavaScript; `go test` y el build wasm pasan |
| F1b | `layout` | correr `gotest` en `layout` con el cambio de F1a | 🟢 vet, style, race, tests y wasm pasan; cobertura 70,8 % |
| F1c1 | `layout` | `layout.Deck` (`layout/deck.go`): `NewDeck(current)`, `Show(id, build)`, `Open(id)`; `leaveMillis` y `leave_millis_test.go` pasaron a `layout`; `platformd` usa `p.deck` (ya no tiene `leaving`, `leaveTimer` ni `release`); pruebas en `layout/deck_test.go` | 🟢 `gotest` en `layout` pasa (la prueba de `Deck` encontró una carrera del temporizador, corregida con `d.mu`); en vivo sigue 2.330 elementos, solo la pantalla activa |
| F1c2 | `room_layout`, `layout` | «Ocupación» usa `layout.NewStepDeck(ActiveStageSig)` (`ui/rootview.go`: campo `stages`, `GoToStage` → `stages.Open`, `Render` → `stages.Show`). Nuevo en `layout/deck.go`: `NewStepDeck`, que suelta el paso anterior en el momento, porque los pasos no se animan y con `NewDeck` se verían dos apilados durante 250 ms; tiene su prueba. `room_layout/go.mod`: `require webtyp.com/layout v0.3.138` + `replace ../../../webtyp/layout`. `Deck` se queda en `layout`: es el chasis y a un wasm solo entran los paquetes importados | 🟢 `gotest` en `room_layout` y en `layout` pasan; en vivo, Operación → Planta → Espacios con un solo paso montado y sin errores de JavaScript |
| F2 | `widget/style`, `layout`, `room_layout` | D3: `style.SkipOffscreen()` (`widget/style/{sheet,except,emit_decls,emit_primitives}.go`, `guard_test.go`, `skip_offscreen_test.go`): emite `content-visibility: auto` y `contain-intrinsic-block-size: auto var(--control-height, …)` en `@layer primitives` y `On(...)`. Aplicado a contenedores sin `Flyout`/`Backdrop(Viewport)`/`OnEdge`: `layout/landing/css.go` (`partSection`), `layout/platformd/css.go` (`nav-item`), `room_layout/ui/css.go` (`PartFloor`, `PartItem`; no `PartPanel`, porque contiene `selectsearch`). `layout/go.mod` suma `replace webtyp.com/widget => ../widget`. Skill **widget-styling** actualizada (con línea STATUS) | 🟢 `gotest` pasa en `widget` (86,1 %), `layout` (70,9 %, wasm) y `room_layout` (64,0 %, wasm/browser); en vivo en `mjosefa-cms` (`#room_layout.occupancy` y «Espacios»: 2.251 elementos, `.room-map__floor`, `.room-map__item` y `.pd__nav-item` con `content-visibility: auto; contain-intrinsic-block-size: auto 56px`, 0 errores JS) |
| F3 | `devtools`, `devbrowser`, skill **wasm** | D4: `devtools` (`memstats_wasm.go` `//go:build dev && wasm`, `memstats_stub.go`, `menu.go`) expone `window.__webtypMemStats()` reutilizando buffer/objeto (0 allocs en producción); `devbrowser` (`devbrowser.go`, `mcp-performance.go`, `tests/mcp-performance_test.go`) lee `wasmMem` y reporta `WASM Mem` + `Delta` por acción (`browser_get_performance`); `mjosefa-cms/go.mod` suma `replace webtyp.com/devtools => ../../webtyp/devtools`; skill **wasm** actualizada (con línea STATUS). Para ver `WASM Mem`/`Delta` en el demonio MCP falta recompilar `webtyp` con `replace devbrowser` | 🟢 `gotest` pasa en `devtools` (100,0 %) y `go vet` + `-race` en `devbrowser`; en vivo en `mjosefa-cms`: desplazar «Espacios» = +0,1 KB (5 allocs, 0 GC), abrir «Configurar → Espacios» = +32,1 MB (540.936 allocs, +4 GC) |
| F4 | `room_layout` | medir de nuevo «Espacios» con F1 y F2 (pestaña visible, 41 cuadros de desplazamiento en `.pd__panel`, 2.251 elementos): `p50 = 31,5 ms`, `p95 = 70,0 ms`, `max = 80,7 ms`, 15/41 cuadros > 50 ms; sigue lento porque los dos pisos visibles pintan ~1.672 celdas DOM a la vez (y abrirlas pide 32,1 MB / 540.936 allocs) → pasa a `ROOM_MAP_MASTER_PLAN.md` | 🟢 medido en vivo; confirmado rediseño de la planta como un solo fondo en `ROOM_MAP_MASTER_PLAN.md` |

## Bloqueos

Ninguno.

## Relación con otras olas

- ROOM_MAP: el desplazamiento lento de «Espacios» espera F1/F2 (F4).
