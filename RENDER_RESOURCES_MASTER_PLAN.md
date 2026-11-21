# Recursos de lo que no se ve: el framework los libera, una vez y para todos — master plan
> **Status:** IN PROGRESS · 2026-11-21 · P1–P3 cerradas (D2–D4); F1 investigada y auditados los `Init` (paso 3 hecho), sin código: lo siguiente es el paso 1 en `platformd`

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

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| F1 | `layout/platformd` | D2. **Sin código todavía; investigado, listo para hacer**: (1) `platformd.go:700-735` monta `m.View()` en cada `Section` de pantalla para siempre; cambiar por `Show(<activa o saliendo>, …)` con un `p.leaving` que guarda la pantalla anterior hasta que termina la animación de `SlideDeck` (`MotionBase`) y luego la suelta. (2) `dom` ya lo soporta: `unmountRecursive` (`dom/dom_frontend.go:606-667`) limpia listeners, suscripciones y cleanups y saca el id de `initedIDs`, así que al volver se llama `Init` otra vez y luego `Render`. (3) Riesgo a revisar: todo `Init` debe poder repetirse sin acumular (suscripciones de larga vida van en el constructor). Verificado: `crudview` se suscribe a `OnChange` en `New` (`layout/crudview/crud.go:147`), no en `Init`. **Revisados los 12 `Init` de pantallas (2026-11-21): todos se pueden repetir.** Con guarda (no rehacen nada): `crudview`, `appointment_booking.BookingFlow`, `payment_manager.PaymentStep`; limpian con `ctx.OnCleanup`: `chat_room.screen` (temporizador); solo rehacen señales/buscador o releen datos: `apiexplorer`, `schemaview`, `chatview`, `ScheduleView`, `ServiceConfigView`, `membersTab`, `PaymentsView`, `room_layout.RootView`. Ningún `Init` se suscribe ni agrega listeners globales. (4) Verificar en vivo: total de elementos de la página baja de ~7.100 a lo de la pantalla visible; navegar entre pantallas sin errores; volver a una pantalla la muestra con sus datos | 🔴 siguiente |
| F2 | `widget/style`, `layout`, `room_layout` | D3 | tras F1 |
| F3 | `devtools`/`app` (MCP), skill **wasm** | D4 | tras F2 |
| F4 | `room_layout` | medir de nuevo «Espacios» con F1 y F2; si sigue lento, la planta como un solo fondo (decisión en `ROOM_MAP_MASTER_PLAN.md`) | tras F1, F2 |

## Bloqueos

Ninguno.

## Relación con otras olas

- ROOM_MAP: el desplazamiento lento de «Espacios» espera F1/F2 (F4).
