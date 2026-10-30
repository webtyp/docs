# Despliegue continuo en servidor propio — master plan
> **Status:** IN PROGRESS · 2026-10-30 · imagen por versión + supervisor (D44 del roadmap); S9 (manifiesto firmado) en Jules; esperan tus P5, P7, P8, P9 y P10

**Disparador:** `mjosefa-cms` corre en un servidor propio dentro de la red del consultorio y depende de ~11 módulos
`veltylabs/*` y ~40 librerías `webtyp/*`. Cuando el dev publica una versión de la app, tiene que llegar sola al servidor
sin cortar una atención. [DEPLOY_WIZARD_MASTER_PLAN.md](DEPLOY_WIZARD_MASTER_PLAN.md) no lo cubre: su único `Target` es
`goflare.Worker` (nube) y D16 borró el puller/SSH/webhook por falta de consumidor, dejando previsto un `deploy.Target` de
servidor propio.

**Enfoque (fijado por el usuario; cambiado el 2026-10-30):** el dev **construye, prueba, firma y publica desde su
equipo** con `gorelease`, cuando una pieza está lista (P32 de MVP_DEPLOYMENT_ROADMAP); el **servidor pregunta** (sale él a
GitHub, sin puertos de entrada). Producción corre en Docker Compose, con el `compose.yaml` generado por `Provision` (D37
y D39 del roadmap). Cuándo aplicar una versión lo decide un módulo de la app, con pantalla y sin TUI. Los clientes son
PWA: con service worker la app abre aunque el servidor esté caído (OF-10 de OFFLINE_FIRST) y `pwa/ui.UpdatePrompt`
nunca recarga a mitad de una atención (D-PWA-8).

**Lo que se reutiliza:** `deploy.Plan` + `Check/Provision/Release` (el "dice qué falta"), `gorelease` (compila, saca
checksums y sube un Release desde el equipo del dev), `webtyp.com/update` (descarga, checksum, `Swap`, `Rollback`; hoy lo
usa `core/update.go:77`), `ddl.Sync` (agrega columnas y solo borra las vacías), `webtyp/pwa` y el RBAC de la app.

**Lo que falta (verificado):** (1) un `Target` de servidor propio — `Job` es solo `uses: accion@v1`
(`deploy/interfaces.go:15`); (2) `gorelease` solo compila `cmd/` (`gorelease/docs/GORELEASE.md`): no conoce
`web/server` + `web/public` (TinyGo), ni imagen, ni firma; (3) el módulo de actualización y quien relanza (P35 del
roadmap); (4) los módulos solo crean tablas, así que un campo nuevo no llega a producción (P5); (5) `config/deploy.go`
en `mjosefa-cms`.

## Preguntas abiertas

**P5 — Migraciones sin escribir migraciones (reescrita: pediste algo simple que cualquier dev entienda).** Hoy cada
módulo solo crea las tablas que faltan (`CREATE TABLE IF NOT EXISTS`, `service_catalog/migrate`): si mañana agregas un
campo a `Service`, la tabla de producción **no recibe la columna** y la pantalla falla. `webtyp/ddl` ya tiene `Sync`:
compara el modelo con la tabla, agrega lo que falta y borra una columna solo si está vacía (`ddl/sync.go:213`, prueba
`ddl/ddl_test.go:284`). Una sola regla: **al arrancar una versión nueva, la app copia la base (`pg_dump`) y llama
`Sync`; nadie escribe migraciones.** A) Eso; si algo falla, vuelve la versión anterior con la base intacta, porque `Sync`
nunca borra datos. B) Seguir solo creando tablas y escribir a mano cada cambio de columna. **Recomiendo A**: un cambio de
modelo llega solo a producción y no hay archivos de migración que mantener. Único caso aparte: renombrar una columna
(`Sync` ve una nueva vacía y una vieja con datos, que no borra); se hace a mano y se avisa en la versión.
```go
// hoy (service_catalog/migrate): la tabla se crea una vez y nunca cambia
d := ddl.New(conn, compiler)
d.CreateTable(&servicecatalog.Service{})
// A: agrega las columnas nuevas; borra solo las que quedaron vacías
d.Sync(&servicecatalog.Specialty{}, &servicecatalog.Service{}, &servicecatalog.ServiceSupply{}, &servicecatalog.ServiceAgreement{})
```
Respuesta:

**P7 — ¿Sigue haciendo falta la rama `release`?** D1 la creó para que la nube supiera qué publicar; ahora publicas tú con
`gorelease` sobre un tag (D42 del roadmap), y `gopush` ya etiqueta cada push (`mjosefa-cms` va en `v0.0.77`). A) Sin rama
`release`: lo que está en producción es lo que tiene un Release en GitHub; los tags de `main` son solo versiones. B) Se
mantiene: se fusiona `main` → `release` y se publica desde ahí. **Recomiendo A**: una rama menos que sincronizar, el
Release ya marca qué se publicó, y se borran S1 y S6. Respuesta:

**P8 — ¿Qué parte de S8 entra antes de salir en línea?** P31 del roadmap pide todo SELFHOST antes de salir, y S8 (la PWA
muestra «actualizando» y reintenta sin perder lo escrito) usa `outbox` de OFFLINE_FIRST, que está fuera del MVP. A) S8
mínima: la PWA muestra «actualizando…» y reintenta la petición; la cola sin conexión queda en OFFLINE_FIRST. B) S8
completa con `outbox`, lo que mete OFFLINE_FIRST al MVP. **Recomiendo A**: D5 ya evita actualizar con clientes
conectados, así que «alguien escribiendo mientras se reinicia» casi no ocurre. Respuesta:

**P9 — ¿Cómo se instala la primera vez en el servidor?** D39 del roadmap dice que el `compose.yaml` no se escribe a mano, pero alguien tiene que ponerlo en el equipo la primera vez, y el servidor no tiene el repo (`config/deploy.go`). A) `gorelease` genera el `compose.yaml` desde `config/deploy.go` y lo sube **en el Release**, junto a la imagen; en el servidor un solo comando del supervisor (`selfupdate install <repo>`) lo baja, genera las claves de Postgres en archivos que solo lee root (D19), pide una vez `TUNNEL_TOKEN` y el token de lectura de GitHub (sin mostrarlos) y levanta el compose. Cada versión nueva trae su compose. B) `Provision` corre en el servidor leyendo `config/deploy.go`, lo que obliga a tener el repo privado allí. **Recomiendo A**: el compose viaja con la versión que lo necesita, las claves nacen en el servidor y nadie las ve, y el servidor solo necesita Docker y un comando. Respuesta:

**P10 — ¿Cómo le dice la app al supervisor «actualiza ahora»?** La app decide cuándo (D5, D6) y el supervisor es el único con acceso a Docker (D44 del roadmap). A) Por HTTP en la red interna del compose: el supervisor escucha solo ahí (sin puerto en el equipo) y exige un token que genera la instalación (P9); la app llama `POST /apply` y la pantalla muestra la respuesta o el error. B) Un archivo en un volumen compartido que el supervisor vigila. **Recomiendo A**: un error vuelve a la pantalla del admin; con B, si el supervisor está caído, nadie se entera. Respuesta:

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (P1 anterior) | **`release` es una rama como `main`.** En revisión por P7: con publicación local puede sobrar | S1, S6 |
| D2 (cambiada por P32 del roadmap) | **El servidor pregunta, el dev construye.** Nada entra a la LAN; `gorelease` publica desde el equipo del dev **el mismo árbol que en desarrollo** (`web/server/server` + `web/public/`, D18 de BUILD_DESTINATIONS) como imagen o Release según P35 del roadmap, con checksums y firma; se niega con cambios sin commit | S2 |
| D3 (cambiada por P32 del roadmap) | **Módulo → app, a mano:** el dev sube las dependencias (`gopush` ya hace los commits `deps: update …`), corre `gotest` y publica con `gorelease` cuando una pieza está lista. Sin GitHub Actions, sin horario y sin bot, por eso P6 ya no aplica | S2 |
| D4 | **La actualización es un módulo integrado** (`webtyp/selfupdate`, sin TUI): pantalla "Actualizaciones" para el administrador, recurso RBAC propio y contador en el menú cuando hay una pendiente | S4, S7 |
| D5 | **Política de reinicio:** con una actualización pendiente y clientes conectados, **solo el administrador** la aplica (botón). Con **cero clientes** se aplica **sola**: caso típico, el servidor se reinició por un corte de luz y arrancó con una versión pendiente; mientras se instala, los clientes PWA esperan y reintentan | S4 |
| D6 (P1) | **Revisión asíncrona y nunca bloqueante**: al arrancar y **cuando se desconecta el último cliente**, más el botón «Buscar ahora» del administrador. Un fallo de red es una línea de log, nunca un error para el usuario (como `core/update.go:23`) | S4 |
| D7 (P35 del roadmap) | **Imagen por versión + supervisor:** `gorelease` publica la imagen y un manifiesto firmado (S9); el supervisor `selfupdate` (contenedor propio, único con acceso a Docker) verifica el manifiesto, baja `imagen@digest`, recrea el servicio `app` y vuelve a la imagen anterior si `/health` falla (D8). `webtyp.com/update` aporta formato, firma y verificación; `Swap`/`Rollback` de archivos no se usan aquí | S4 |
| D8 | **Vigilancia tras reiniciar:** si `/health` no responde en N segundos, se vuelve a la versión anterior y se marca la nueva como mala (no se reintenta sola) | S4 |
| D9 | **Secretos de producción solo en el servidor** (`DATABASE_URL`, token del túnel, token de lectura de GitHub): sin API ni conexión entrante, D1/D29 de DEPLOY_WIZARD se cumplen tal cual. La app informa lo que falta al arrancar (mismo texto de `Missing`) | S4, S7 |
| D10 (cambiada por P3) | **Una plataforma:** el servidor es Debian 13 amd64 en Docker; se publica solo `linux/amd64` | S2 |
| D11 (P2) | **Cliente conectado** = sesión con latido reciente (30 s, en línea < 90 s, como CR-6 de `chat_room`) o stream SSE abierto, contado por una función que la app inyecta (`ActiveClients() int`); una PWA abierta sin servidor no cuenta | S4 |
| D12 (P3) | **Servidor: Debian 13** en el HPE MicroServer Gen10 Plus, con Docker Compose; antes, el MVP en el PC del dev (D14 y D37 del roadmap) | S7 |
| D13 (P4) | **Firma con `webtyp/crypto/asym`** (ECDSA P-256, ya existe; reemplaza el ed25519 propuesto, que habría sido una segunda forma de firmar): la clave privada vive en el `webtyp/keyring` de quien publica, la pública va compilada en la app (`config/update_key.go`); lo firmado es el manifiesto con el digest de la imagen, nunca un tag (como cosign). El repo es privado: el servidor usa un token de solo lectura | S2, S3, S4, S9 |

## Fases

Ecosistema (Step 0): no existe `selfupdate`; `gorelease` existe (Release desde el equipo, solo `cmd/`) → **se extiende**;
**nuevo `webtyp/selfupdate`** encima de `webtyp/update` y `webtyp/pwa`; **nuevo `Target` `selfhost`** en `deploy` (sin
importar `goflare`).

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| S0 | docs | Contestar P5, P7, P8, P9 y P10; consolidar | pendiente (tú) |
| S1 | `deploy` | `Plan.Branch`; D8 de DEPLOY_WIZARD pasa a "push a `Plan.Branch`" | solo si P7 = B |
| S2 | `gorelease` | Construir la app de `webtyp` (`web/server` con Go, `web/public` con TinyGo), `gotest`, imagen `linux/amd64` en GHCR, manifiesto firmado (S9) y, según P9, el `compose.yaml`; negarse con cambios sin commit | espera S9, P9 |
| S3 | `deploy` (`Target` `selfhost`) | Par de claves (D13) y `config/update_key.go`; lo demás según P9 | espera P9 |
| S4 | `selfupdate` | Supervisor (contenedor): instalación (P9), revisión, verificación del manifiesto, cambio de imagen, vigilancia (D8), migración (P5); módulo de la app: pantalla, RBAC, `ActiveClients()` (D11), aviso al supervisor (P10) | espera S9, P5, P9, P10 |
| S5 | — | Absorbida por S2 (`gorelease` empaqueta) | cerrada sin código |
| S6 | devflow (`gopush`) | Etiquetar desde la rama de `Plan.Branch` | solo si P7 = B |
| S7 | `mjosefa-cms` | `config/deploy.go`, `selfupdate` en `modules/`; al aceptar borrar `DESPLIEGUE_WINDOWS.md` §5 y B2 de `MASTER.md` | espera S2–S4 |
| S8 | `pwa` / `devskills` | Cliente ante reinicio del servidor (alcance: P8); skill de despliegue en servidor propio y excepción D9 en **secrets** | espera P8, con S4 |
| S9 | `update` | Manifiesto firmado: `Manifest{Version, Image, Digest}`, `SignManifest`, `VerifyManifest`, `Reference()` | [Jules · 2026-10-30] despachado |

**Aceptación:** (1) el dev publica con `gorelease` → el servidor lo detecta; con clientes conectados queda pendiente y el
administrador lo aplica; (2) apagar y encender el servidor con una versión pendiente → se instala sola antes de recibir
clientes y estos esperan sin perder datos; (3) migración fallida o `/health` rojo → queda la versión anterior; (4) una
versión sin firma válida se rechaza.

## Bloqueos

- Ninguno fuera de las preguntas abiertas.

## Relación con otras olas

- Extiende DEPLOY_WIZARD (cerrado): añade un `Target`; no cambia el resto de su contrato.
- MVP_DEPLOYMENT_ROADMAP (`mjosefa-cms`): su consumidor; su D11 exige SELFHOST completo antes de salir en línea, y su P35 decide imagen o binario.
- BUILD_DESTINATIONS define el árbol que empaqueta S2 (`web/server/server` + `web/public/browser.wasm`, D18) y que las rutas relativas se resuelven contra el directorio de trabajo (D20).
- OFFLINE_FIRST (mjosefa-cms): la cola sin conexión (`outbox`) es suya; S8 depende de P8.
- SECURITY_BY_DEFAULT: el módulo exige permiso explícito, no expone datos en `/health` y verifica firma antes de ejecutar.
- MONEY_FLOWS D14 (2026-10-22): el servidor de `mjosefa-cms` recibe avisos de pago desde internet por **Cloudflare Tunnel** (solo rutas públicas) + respaldo 4G; un solo servicio local.
