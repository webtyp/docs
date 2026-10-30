# Despliegue continuo en servidor propio — master plan
> **Status:** IN PROGRESS · 2026-10-30 · esperan tus P11–P13 (claves en el contenedor, quién migra, imagen e instalación del supervisor); S4a–S4b cerradas, S4c (API HTTP) en Jules; S10: pruebas cerradas (appkit v0.0.26, sqlite v0.3.26, erd v0.0.16, changelog v0.2.14); queda ddl

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

**P11 — ¿Cómo recibe la app sus claves dentro del contenedor?** Docker entrega cada secreto como un archivo (`/run/secrets/<nombre>`), pero la app lee `DATABASE_URL` del entorno (`env.Get`, `mjosefa-cms/config/server.go`) y `webtyp/env` no sabe leer archivos de secretos. A) `webtyp/env` aprende la convención estándar `<CLAVE>_FILE`: si `DATABASE_URL` no está y `DATABASE_URL_FILE` sí, lee el archivo (lo mismo que hace la imagen oficial de Postgres con `POSTGRES_PASSWORD_FILE`). B) Las claves van como variables normales en el `.env`. **Recomiendo A**: con B cualquiera que pueda correr `docker inspect` o `docker compose config` ve las claves en texto plano; con A la app no cambia ni una línea.
```yaml
# A — compose.yaml generado: la app lee el archivo, nadie ve la clave
app:
  environment: { DATABASE_URL_FILE: /run/secrets/database_url }
  secrets: [database_url]
# B — .env: la clave queda en texto plano en el equipo y en docker inspect
DATABASE_URL=postgres://app:Kd8...@db/shop
```
Respuesta:

**P12 — ¿Quién migra la base al cambiar de versión?** D19 decidió dos usuarios de Postgres: uno que migra (dueño) y otro que solo lee y escribe filas; pero D14/D21 dicen que la app migra al arrancar, y la app corre con el usuario limitado, que no puede crear columnas. A) El supervisor migra antes de cambiar: corre el programa de migración de la imagen nueva con el usuario dueño (`docker compose run --rm app migrate`) y recién después cambia la app; si la migración falla, no se cambia nada. La imagen trae los dos programas (`mjosefa-cms` ya tiene `cmd/migrate`). B) Un solo usuario dueño para todo, y la app migra al arrancar. **Recomiendo A**: mantiene D19 (si alguien toma el control de la app, no puede borrar tablas), y como `Sync` solo agrega, la versión anterior sigue funcionando con la base ya migrada. El flujo de cambio queda: copia → bajar imagen → **migrar** → cambiar → salud. Respuesta:

**P13 — ¿Cómo se publica la imagen del supervisor y cómo se instala?** A) Una Action del repo público `webtyp/selfupdate` construye `ghcr.io/webtyp/selfupdate:<versión>` en cada tag (repo público: minutos gratis), y la instalación en el servidor es un solo `docker run` de esa imagen: el servidor solo necesita Docker. B) `gorelease` la construye en tu equipo como a las apps (D42) y en el servidor se instala un binario de Go. **Recomiendo A**: D42 dejó fuera las Actions por los minutos del repo **privado** de la app; el supervisor es una herramienta pública de webtyp, igual que las que hoy publica `webtyp/gorelease@v1`.
```bash
# A — lo único que se escribe en el servidor recién formateado (con Docker instalado)
docker run --rm -it -v /opt:/opt -v /var/run/docker.sock:/var/run/docker.sock \
  ghcr.io/webtyp/selfupdate:v0.1.0 install acme/shop
```
Respuesta:

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D2 (cambiada por P32 del roadmap) | **El servidor pregunta, el dev construye.** Nada entra a la LAN; `gorelease` publica desde el equipo del dev **el mismo árbol que en desarrollo** (`web/server/server` + `web/public/`, D18 de BUILD_DESTINATIONS) como imagen o Release imagen (D44 del roadmap), con checksums y firma; se niega con cambios sin commit | S2 |
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
| D13 (P4) | **Firma con `webtyp/crypto/asym`** (ECDSA P-256, ya existe): la privada vive en el `webtyp/keyring` de quien publica; la pública la guarda el servidor en `update.pub` al instalar (D19); lo firmado es el manifiesto con el digest de la imagen, nunca un tag (como cosign). El repo es privado: el supervisor usa un token de solo lectura | S2, S3, S4, S9 |
| D14 (P5) | **Nadie escribe migraciones**: cada módulo migra con `ddl.Sync` (crea la tabla si falta, agrega columnas, borra solo columnas vacías); al arrancar una versión nueva el supervisor hace `pg_dump` y la app corre sus pasos; si algo falla, vuelve la imagen anterior con la base intacta. Renombrar una columna es el único caso a mano. Al final `ddl` deja de exportar `CreateTable`, para que nadie vuelva a crear tablas que nunca cambian | S4, S10 |
| D15 (P7) | **Sin rama `release`**: lo que está en producción es lo que tiene un Release en GitHub; los tags de `main` son solo versiones. Se borran S1, S6 y el antiguo D1 | — |
| D16 (P8) | **S8 mínima**: ante un reinicio del servidor la PWA muestra «actualizando…» y reintenta la petición; la cola sin conexión queda en OFFLINE_FIRST | S8 |
| D17 (P9) | **Primera instalación con un comando**: `gorelease` genera el `compose.yaml` desde `config/deploy.go` y lo sube en el Release; en el servidor `selfupdate install <owner/repo>` lo baja, genera las claves de Postgres en archivos que solo lee root, pide una vez `TUNNEL_TOKEN` y el token de lectura de GitHub (sin mostrarlos) y levanta el compose. Cada versión trae su compose | S2, S4 |
| D18 (P10) | **La app avisa al supervisor por HTTP** en la red interna del compose (sin puerto en el equipo), con un token generado al instalar: `POST /apply`; la pantalla muestra la respuesta o el error | S4 |
| D19 | **Quien verifica es el supervisor, no la app** (consecuencia de D44 del roadmap: la app ya no descarga nada): la clave pública vive en `/opt/<app>/update.pub`, escrita al instalar (no es secreta); se borra la idea de `config/update_key.go` | S4 |
| D20 | **El supervisor cambia de imagen con `docker compose`**: escribe `APP_IMAGE=<imagen>@<digest>` en `.env` (el compose usa `image: ${APP_IMAGE}`) y recrea solo el servicio `app`; volver atrás es escribir la anterior. Contrato del servidor (`/opt/<app>/`: compose, `.env`, `update.pub`, `secrets/`, `state.json`, `backups/`; servicios `db`, `app`, `cloudflared`, `selfupdate`) en [selfupdate/docs/ARCHITECTURE.md](https://github.com/webtyp/selfupdate/blob/main/docs/ARCHITECTURE.md) y sus nombres en `selfupdate/layout.go`, que importan `deploy` y `gorelease` | S2, S3, S4 |
| D21 | **Antes de cambiar, copia de la base** en `backups/` (las últimas 7); la app migra sus tablas al arrancar con `Sync` (D14), así volver a la imagen anterior deja la base usable | S4 |

## Fases

Ecosistema (Step 0): no existe `selfupdate`; `gorelease` existe (Release desde el equipo, solo `cmd/`) → **se extiende**;
**nuevo `webtyp/selfupdate`** encima de `webtyp/update` y `webtyp/pwa`; **nuevo `Target` `selfhost`** en `deploy` (sin
importar `goflare`).

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| S0 | docs | Contestar P5, P7–P10; consolidar | cerrada 2026-10-30 |
| S2 | `gorelease` | Construir la app de `webtyp` (`web/server` con Go, `web/public` con TinyGo), `gotest`, imagen `linux/amd64` en GHCR, manifiesto firmado (S9) y el `compose.yaml` (D17); negarse con cambios sin commit | espera S9 publicado y S3 |
| S3 | `deploy` (`Target` `selfhost`) | Par de claves (D13, la pública impresa para `install`) y `Compose(plan)`: el `compose.yaml` que sube `gorelease` (D17), con los nombres de `selfupdate/layout.go` | espera S4a publicado |
| S4 | `selfupdate` (repo nuevo, `gonew`) | S4a: nombres del servidor, estado y revisión verificada ([CHECK_FLOW](https://github.com/webtyp/selfupdate/blob/main/docs/diagrams/CHECK_FLOW.md)); S4b: cambiar imagen, copia, salud y vuelta atrás ([APPLY_FLOW](https://github.com/webtyp/selfupdate/blob/main/docs/diagrams/APPLY_FLOW.md)); S4c: API HTTP con token e `install`; S4d: módulo de la app (pantalla, RBAC, `ActiveClients`) | S4a cerrada v0.0.2; S4b cerrada v0.0.3; siguen S4c–S4d |
| S5 | — | Absorbida por S2 (`gorelease` empaqueta) | cerrada sin código |
| S7 | `mjosefa-cms` | `config/deploy.go`, `selfupdate` en `modules/`; al aceptar borrar `DESPLIEGUE_WINDOWS.md` §5 y B2 de `MASTER.md` | espera S2–S4 |
| S8 | `pwa` / `devskills` | Cliente ante reinicio del servidor (D16: «actualizando…» y reintento); skill de despliegue en servidor propio y excepción D9 en **secrets** | con S4 |
| S9 | `update` | Manifiesto firmado | cerrada v0.0.5 |
| S10 | `ddl`, luego docs | D14: los 15 módulos y todas las pruebas ya usan `Sync`; `ddl` deja de exportar `CreateTable` (su `conformance` pasa a `Sync`); al publicarse, barrido de los ~31 AGENTS.md/README que aún muestran el ejemplo viejo `d.CreateTable(...)` | [Jules · 2026-10-30 19:30] `ddl` despachado; barrido de docs después |

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
