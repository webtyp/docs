# Recursos de lo que no se ve: el framework los libera, una vez y para todos — master plan
> **Status:** IN PROGRESS · 2026-11-21 · P1–P3 cerradas (D2–D4); F1a, F1b y F1c1 🟢 en local (7.102 → 2.330 elementos); sigue F1c2 (`room_layout` usa `layout.Deck`)

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
| F1c2 | `room_layout` | `RootView` crea un `layout.Deck` sobre `ActiveStageSig`; `Render` usa `deck.Show` en vez de sus cuatro `dom.Show` (`ui/rootview.go:212-222`) y `GoToStage` usa `deck.Open`. Ya descartado: `Refresh` no reconstruye el paso (`dom/signal.go:72`). Verificar en vivo: cambiar de paso en «Ocupación» sin errores; `gotest` en `room_layout`. Luego quitar del skill **components** lo de «sub-steps still use a hand-written `dom.Show`» | 🔴 sigue (revisar antes si `room_layout/go.mod` ya requiere `webtyp.com/layout`; si no, agregarlo con `replace ../../../webtyp/layout`) |
| F2 | `widget/style`, `layout`, `room_layout` | D3: `SkipOffscreen()` | ⚪ tras F1c2 |
| F3 | `devtools`/`app` (MCP), skill **wasm** | D4: la memoria pedida por acción | ⚪ tras F2 |
| F4 | `room_layout` | medir de nuevo «Espacios» con F1 y F2; si sigue lento, la planta como un solo fondo (decisión en `ROOM_MAP_MASTER_PLAN.md`) | ⚪ tras F2 |

## Bloqueos

Ninguno.

## Relación con otras olas

- ROOM_MAP: el desplazamiento lento de «Espacios» espera F1/F2 (F4).
