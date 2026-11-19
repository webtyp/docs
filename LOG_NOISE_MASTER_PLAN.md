# Logs de `webtyp dev` sin ruido repetido — master plan
> **Status:** PENDING · 2026-11-19 · esperando respuestas del dueño a P1–P2

**Para retomar:** skills `console-logs`, `webtyp-app`, `testing`. Código: `webtyp/app` (`section-build.go`, `client_file_event.go`, `client_change.go`, `sse_publisher.go`, `headless_tui.go`), `webtyp/server` (`strategies.go`). La paridad TUI/MCP ya está publicada (un anillo por manejador, `app` v0.8.38): lo que ve el humano lo ve el agente.

**Qué se ve hoy (dueño, 2026-11-19, mjosefa-cms):** «los logs son redundantes, puro ruido repetido… en cada reinicio se muestra lo mismo». Cada guardado de un `.go` imprime:
```
CLIENT    ✅ build ok [disk|8.1 MB]
CLIENT       [disk|8.1 MB]                       ← segunda línea, mismo build
SERVER    Restarting server... .
SERVER    Server running on port 8080
SECURITY  Warning authn_login_fail … device not trusted   ┐ los mismos 3 rechazos de
SERVER    auth: DEV_AUTOLOGIN rejected, login form shown… ┘ DEV_AUTOLOGIN, en cada reinicio
```

**Causas (verificadas):**
1. Doble línea del build: `app/section-build.go:458` (`"✅ build ok "+BuildSummary()`) y `app/client_file_event.go:36` (`w.LogSuccessState()` sin texto → solo el sufijo, `client_change.go:203-206`).
2. Cada reinicio es un proceso nuevo: el autologin vuelve a intentar sus credenciales (correcto: el estado de la base pudo cambiar) y cada rechazo vuelve a imprimirse igual que en el reinicio anterior.
3. La TUI solo junta repeticiones **consecutivas** («repetido N veces»); una línea idéntica a la del ciclo anterior, con otras en medio, se muestra entera otra vez.

## Preguntas abiertas
**P1 — ¿Qué hace la vista con una línea idéntica a una del reinicio anterior?**
A) Se junta con la anterior: la TUI y el MCP la muestran una vez con contador («×3 reinicios») y la hora del último; solo se ve entera si cambia. Aplica a todo manejador por igual (lo que pidió el dueño: consistencia).
B) Se muestra solo un resumen al final del reinicio: «sin cambios desde el reinicio anterior (6 líneas)».
C) Como hoy.
**Recomiendo A**: es la regla que ya existe para repeticiones seguidas, extendida a todo el ciclo; el agente y el humano siguen viendo lo mismo porque vive en el publicador (`sse_publisher.go`), no en la TUI.
```text
hoy (3 reinicios):  9 líneas SECURITY + 9 SERVER
A:                  SECURITY  Warning authn_login_fail … device not trusted   ×3
                    SERVER    auth: DEV_AUTOLOGIN rejected, login form shown on localhost   ×3
```
Respuesta:

**P2 — ¿El reinicio normal del servidor se muestra?**
A) Una sola línea viva por ciclo («Server running on port 8080 · reinicio 4»), que se reemplaza en cada reinicio; el texto «Restarting server…» solo aparece si el reinicio tarda o falla.
B) Como hoy: dos líneas por reinicio.
**Recomiendo A**: un reinicio que sale bien no es noticia; uno que falla sí, y queda visible.
Respuesta:

## Decisiones por implementar
| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 | Un build imprime **una** línea: se borra la segunda (`client_file_event.go:36` → la línea de `section-build.go:458` ya lleva el tamaño). Obvio, sin pregunta | `webtyp/app` |

## Fases
| Fase | Repo | Qué | Estado |
|---|---|---|---|
| F1 | webtyp/app (+ server si P2=A) | D1 y lo que resuelvan P1–P2, con tests que fallan primero (3 reinicios con las mismas líneas → una línea con contador; un build → una línea) | 🔴 espera P1–P2 |
| F2 | webtyp/devskills | `console-logs`: la regla «una línea idéntica a la del ciclo anterior no se repite» | 🔴 tras F1 |

## Bloqueos
- Ninguno.

## Relación con otras olas
- ENV_HOT_RELOAD: cada reinicio es también cuando el servidor relee `.env`.
