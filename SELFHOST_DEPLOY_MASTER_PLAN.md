# Despliegue continuo en servidor propio — master plan
> **Status:** IN PROGRESS · 2026-10-31 · sin preguntas; S2 cerrada (`gorelease` v0.1.10); S4c-2b (`selfupdate`) en Jules; después S4d (`selfupdate` módulo de la app) y S8 (`pwa`)

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

Ninguna.

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
| D22 (P11) | **Claves como archivos**: el compose entrega cada secreto en `/run/secrets/<nombre>` y la app lo lee con la convención `<CLAVE>_FILE` de `webtyp/env` (`DATABASE_URL_FILE`); ninguna clave en `.env` ni en `docker inspect` | `env`, S3 |
| D23 (P12) | **El supervisor migra antes de cambiar**: tras bajar la imagen corre su programa de migración con el usuario dueño (`docker compose run --rm app migrate`); si falla, no cambia nada; la app corre con el usuario que solo lee y escribe filas (D19). La imagen trae los dos programas. Flujo: copia → bajar → **migrar** → cambiar → salud | S4, S2, S3 |
| D24 (P13) | **Imagen pública del supervisor por Action** en `webtyp/selfupdate` en cada tag (`ghcr.io/webtyp/selfupdate:<versión>`); la instalación es un solo `docker run … ghcr.io/webtyp/selfupdate:<versión> install <owner/repo>`: el servidor solo necesita Docker | S4 |
| D25 | **Contrato del compose** en [selfupdate/docs/ARCHITECTURE.md](https://github.com/webtyp/selfupdate/blob/main/docs/ARCHITECTURE.md#the-compose-file--contract): servicios `db`, `migrate` (perfil, solo lo corre el supervisor), `app`, `cloudflared`, `selfupdate`; claves como archivos `_FILE`; volumen `certs` (el certificado de Let's Encrypt sobrevive a cada actualización: sin él cada cambio pediría uno nuevo y Let's Encrypt limita los repetidos); la puerta de la clínica en `${LAN_ADDRESS}:443`, la de internet sin publicar | S3, S4, S2 |
| D26 | **El manifiesto firma también la huella del `compose.yaml`** (`ComposeDigest`): un compose adulterado en el Release (por ejemplo, uno que monte el disco del servidor) se rechaza igual que una imagen adulterada | `update`, S4, S2 |
| D27 | **Imagen de la app**: `/app/server` (arranque), `/app/migrate`, `/app/web/public`, carpeta de trabajo `/app`, usuario sin privilegios, `linux/amd64` | S2 |
| D28 | **El usuario de la app** (solo filas) lo crea Postgres en su primer arranque con `secrets/initdb.sql`, que genera `install` junto con las claves | S4 |
| D29 (P14) | **El `compose.yaml` lo genera `selfupdate`**, junto al contrato (D25): `selfupdate.ComposeFile(selfupdate.App{Name, Domain, Repo})` (`Compose` ya es el motor de docker); `gorelease` lee `Name` y `Domain` de `config/deploy.go` y lo llama. `deploy` no cambia: la app declara `Target: selfupdate.Server` y `CI: gorelease.Local`, dos constantes de identidad (corrige D39 del roadmap) | S4, S2 |
| D30 (P15) | **Copia diaria de la base** por el supervisor en `backups/` (la misma copia de D21, las últimas 7); en el servidor esa carpeta es el disco de respaldos (D14); la copia fuera del equipo se decide en M10 | S4 |

## Fases

Ecosistema (Step 0): no existe `selfupdate`; `gorelease` existe (Release desde el equipo, solo `cmd/`) → **se extiende**;
**nuevo `webtyp/selfupdate`** encima de `webtyp/update` y `webtyp/pwa`; **nuevo `Target` `selfhost`** en `deploy` (sin
importar `goflare`).

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| S0 | docs | Contestar P5, P7–P10; consolidar | cerrada 2026-10-30 |
| S2 | `gorelease` | S2a `gorelease image`: imagen de la app (D27) en ghcr.io por su huella, se niega con árbol sucio, tag que no es HEAD o pruebas rojas; S2b manifiesto firmado (imagen + compose de `selfupdate.ComposeFile`) y Release en el mismo repo privado | cerrada v0.1.10 |
| S3 | — | Absorbida por S4 (D29): `deploy` no cambia | cerrada sin código |
| S4 | `selfupdate` | Cerradas S4a, S4b, S4b-2, S4c-1, S4c-2a (v0.0.7), S4e+S4f (v0.0.8); siguen S4c-2b (`serve` con la copia cada 24 h, `install` con dueño y permisos de cada clave, `Dockerfile`, Action) y S4d (módulo de la app) | S4c-2b [Antigravity · 2026-10-31 01:43] despachado; sigue S4d |
| S5 | — | Absorbida por S2 (`gorelease` empaqueta) | cerrada sin código |
| S7 | `mjosefa-cms` | `config/deploy.go`, `selfupdate` en `modules/`; al aceptar borrar `DESPLIEGUE_WINDOWS.md` §5 y B2 de `MASTER.md` | espera S2–S4 |
| S8 | `pwa` / `devskills` | Cliente ante reinicio del servidor (D16: «actualizando…» y reintento); skill de despliegue en servidor propio y excepción D9 en **secrets** | con S4 |
| S9 | `update` | Manifiesto firmado | cerrada v0.0.5 |
| S11 | `env` | D22: `<CLAVE>_FILE` | cerrada |
| S12 | `update` | D26: `ComposeDigest`, `Digest`, `CheckCompose` | cerrada v0.0.6 |
| S13 | `server` | `FromEnv` acepta `CLOUDFLARE_DNS_API_TOKEN_FILE` | cerrada v0.2.105 |
| S10 | `ddl`, docs | D14: `CreateTable` ya no es pública (`ddl` v0.0.37); barrido de docs completado | cerrada |

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
