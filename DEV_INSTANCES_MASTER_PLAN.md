# Varios proyectos a la vez en `webtyp dev` — master plan
> **Status:** IN PROGRESS · 2026-11-05 · tui, devflow, gorun y devtui cerrados; falta `app` (fase 4, Jules)

**Qué duele hoy.** Con `mjosefa-cms` corriendo, abrir `webtyp dev` en `room_layout` para buscar un error de vista
**mata** el proyecto anterior: hay que detenerlo y volver a arrancar el otro. El agente vive lo mismo y peor: cuando
el demonio muere (por ejemplo, tras instalar un `webtyp` nuevo) sus herramientas MCP responden `ECONNREFUSED` y no
tiene cómo levantarlo. **Regla de la ola: lo que el dev hace en la TUI, el agente lo hace por MCP; si no, es un fallo.**

**Lo que encontré en el código (lectura superficial):**
- Un solo proyecto por demonio: `projectCancel`, `activeHandler` y `lastPath` son únicos (`app/daemon.go:869-931`);
  `launchProject` cierra el anterior antes de abrir el nuevo.
- `webtyp dev` en otra carpeta **reinicia el demonio entero**: `ShouldRestartDaemon` compara la carpeta activa
  (`app/net.go:34-39`, usado en `app/bootstrap.go:84`).
- Puerto fijo: `Config.ServerPort()` lee `PORT` del proceso o `8080` (`app/config.go:204-209`); el demonio espera a que
  `8080` se libere antes de arrancar otro (`daemon.go:900-918`).
- Un solo navegador (`d.browser`) y una sola sesión de logs (`SSEPublisher.StartSession`, `app/sse_publisher.go:160`).
- MCP del demonio: `start_development`, `restart_development`, `app_get_logs`, `app_info` (`daemon.go:646-700`).
- Ya existe `webtyp stop` / `webtyp status` (`app/status.go`) y `relaunch` con `syscall.Exec` (`app/relaunch_unix.go`).

## Preguntas abiertas

Ninguna.

## Decisiones por implementar
| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (P1) | Un demonio (MCP `6060`) con varios proyectos, cada uno en su puerto `8080`…`8084`; `start_development` suma, no reemplaza; `project` opcional en las demás herramientas (sin él: el último usado); nuevas `list_development` y `stop_development` (sin `project`: todos) | `app` (`daemon.go`, `config.go`) |
| D2 (P2) | Máximo **5** proyectos contando el primero; el sexto se rechaza nombrando los 5 que corren: «detén uno» | `app` |
| D3 (P3) | Pestaña MCP: `HandlerSelection` con un botón por proyecto + «Todos». 1.er Enter entra a elegir (flechas), 2.º Enter detiene el elegido; Esc cancela — ya es la interacción de `devtui` (`tui/interfaces.go:60-63`). `Value()` = el proyecto actual (el resaltado arranca ahí), así nunca queda vacío | `app` |
| D4 (P4) | Cada terminal muestra BUILD/DEPLOY y logs de **su** carpeta. **Un solo navegador**, que salta al puerto del último proyecto usado; `browser_*` actúan sobre ese | `app` |
| D5 (P5) | El demonio vigila su ejecutable; si cambia, guarda la lista de proyectos, se relanza (`relaunch`) y los vuelve a arrancar. Lista en `os.UserConfigDir()/webtyp/projects.json` (stdlib, los 3 SO: Linux `~/.config/webtyp`, macOS `~/Library/Application Support/webtyp`, Windows `%AppData%\webtyp`); el `relaunch` de Windows no tiene `syscall.Exec`: arranca el nuevo y sale. Respaldo: `webtyp mcp &` en la skill **webtyp-app** | `app` (`relaunch_*.go`) |
| D6 (P6) | Otra carpeta = sumar un proyecto; el demonio solo se reinicia si cambió la **versión** (se borra la comparación de carpeta de `ShouldRestartDaemon`, `app/net.go:34-39`) | `app/net.go`, `bootstrap.go` |
| D7 (P7) | La pestaña MCP de **cada** terminal muestra **todos** los proyectos del demonio, el de su carpeta primero (incluye los que arrancó el agente sin terminal) | `app/section-mcp.go` |
| D8 | Hoy `gorun` mata **todo** proceso llamado `server` al arrancar o detener (`pgrep -x server`, `gorun/cleanup.go:52`, `RunProgram.go:18`): el proyecto B mataría el servidor de A. Se mata por **ruta absoluta del ejecutable**, no por nombre | `gorun` → `server` → `app` |
| D9 (P8) | Cerrar una terminal (`Ctrl+C`) detiene **solo su proyecto**; el demonio sigue vivo sin proyectos (navegador cerrado) y se apaga solo con `webtyp stop` | `app/bootstrap.go`, `devtui` |
| D10 | Causa de la caída del 2026-11-05: `devflow` `Install` llama `gorun.StopApp("webtyp")` antes de `go install` (`devflow/go_handler.go:704`) y mata al demonio. Se borra; en Windows el `.exe` en uso se renombra a `.old` | `devflow` |

## Fases
Orden de publicación: `tui` → `devflow` → `gorun` → `devtui` → `app` (cada uno usa al anterior).

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| 0 | `devflow` | `Install` deja de matar al binario en uso (D10) | cerrada v0.4.166 |
| 0b | `devflow` | `gotest` sin `docs/TESTS.md` no avisa nada; si existe, se revisa como antes | cerrada v0.4.167 (ejecutada en local) |
| 1 | `tui` | constante `tui.ProjectKey = "project"` | cerrada v0.1.4 |
| 2 | `gorun` | matar por ruta absoluta del ejecutable, no por nombre (D8); `server` ya lo usa | cerrada v0.0.29 |
| 3 | `devtui` | el cliente envía su proyecto en `/logs`, `webtyp/state`, `webtyp/action`; `Ctrl+C` ya no envía `stop` (D4, D9) | cerrada v0.4.70 |
| 4 | `app` | hasta 5 proyectos, herramientas, pestaña MCP, cliente, relanzarse al cambiar el binario (D1–D7, D9) — `docs/PLAN.md` + 5 etapas | [Claude · 2026-11-05 18:54] despachado; al cerrar, subir `app` a `gorun` v0.0.29 y `devflow` v0.4.167 (la cascada lo saltó por la sesión activa) |
| 5 | skill **webtyp-app** | varios proyectos, `list/stop_development`, `webtyp mcp &` como respaldo | con fase 4 |
| 6 | `mjosefa-cms` + `room_layout` | correr los dos a la vez, detener uno desde la TUI y otro por MCP, reinstalar `webtyp` sin perderlos, capturas | última |

## Bloqueos
- Ninguno.

## Relación con otras olas
- Sale de DATA_IMPORT (M6: el demonio cayó tras instalar `app` v0.7.59). No la bloquea: DATA_IMPORT sigue con un proyecto.
- TUI_STAGES ordena las pestañas de **un** proyecto; esta ola trata **cuántos** proyectos. Comparten la pestaña MCP.
