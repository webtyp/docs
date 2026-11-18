# Mapa del framework — master plan
> **Status:** IN PROGRESS · 2026-11-18 · M1 publicada (`devskills` v0.0.57); grupos por dominio y marcas 🟢/🔴 publicados (`devskills` v0.0.61); sigue M2 (skill **session**); sin automatización · bloquea D75 de [ARCHITECTURE](ARCHITECTURE_MASTER_PLAN.md) (inicio de sesión), que espera la skill de su componente

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
| D4 (dueño, 2026-11-18) | Los repos se agrupan por dominio en `framework-map`, cada uno con 🟢 (un trabajo claro, en su dominio) o 🔴 (defecto con su evidencia en la skill). Se verificó con los imports reales: 13 🔴. Las relaciones entre repos se dibujan en la skill de cada dominio, no en el índice | `devskills` v0.0.61 (hecho) |

## Fases

Cada fase termina con la revisión (🔵) del dueño antes de la siguiente.

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| M2 | `devskills` | Una skill por dominio sin skill (offline sync, events, session, documents, AI and voice), una por vez, empezando por **session** (desbloquea D75): diagrama de quién importa a quién y la responsabilidad de cada repo, leído del código | 🔴 |
| M3 | repos 🔴 | Corregir cada 🔴 de `framework-map` (`appkit`, `auth`, `layout`, `server`, `lang`, `dbsync`, `pwa`, `image`, `imagemin`, `pdf`, `broker`, `binary`, `retrieval`); cada uno es una pregunta en ARCHITECTURE antes de tocarlo | 🔴 |

## Bloqueos

- Ninguno.

## Relación con otras olas

- ARCHITECTURE: D75 (el inicio de sesión como una pieza) espera la skill **session** (M2).
