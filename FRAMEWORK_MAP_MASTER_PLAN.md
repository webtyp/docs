# Mapa del framework — master plan
> **Status:** IN PROGRESS · 2026-11-18 · M1 publicada (`devskills` v0.0.57); sigue M2 (skill **session**); sin automatización · bloquea D75 de [ARCHITECTURE](ARCHITECTURE_MASTER_PLAN.md) (inicio de sesión), que espera la skill de su componente

Problema que se ve: un agente llega sin saber qué piezas forman una app ni qué skill explica cada una. Entonces supone, pone código en la pieza equivocada y el dueño tiene que revisarlo todo. Caso real (2026-11-18): el modo de entrada `trusted_ip` quedó dentro de `appkit` (D68), y ninguna skill dice qué repos forman el componente «sesión».
Reglas: **devskills** (una skill = un dominio, ≤50 líneas es lo ideal, más de 150 se rechaza).

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (dueño, 2026-11-18) | `appkit` es el único lugar donde una app configura, pero no implementa ninguna pieza: arma lo que la app eligió. Lo que hoy implementa por su cuenta (el modo `trusted_ip`) sale a la pieza dueña | ARCHITECTURE D75 |
| D2 (dueño, 2026-11-18) | Una sola skill índice, `framework-map`, que enlaza solo las skills de los componentes globales de una app (transporte, sesión, codec, base de datos…). Skills chicas. **Nada se automatiza** por ahora: sin generador ni gates (reemplaza P1–P5, rechazadas) | `devskills` |
| D3 (P6) | La skill índice lleva un diagrama mermaid de componentes y una tabla componente → repos → skill; cada skill de componente repite el patrón a su escala; el último eslabón es el código del repo (`go doc`), nunca una copia de firmas en las skills | `devskills` (hecho en M1) |

## Fases

Cada fase termina con la revisión (🔵) del dueño antes de la siguiente.

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| M2 | `devskills` | Las skills de componente que faltan, una por vez, empezando por **session** (desbloquea D75). Cada una leída del código, no de memoria | 🔴 |

## Bloqueos

- Ninguno.

## Relación con otras olas

- ARCHITECTURE: D75 (el inicio de sesión como una pieza) espera la skill **session** (M2).
