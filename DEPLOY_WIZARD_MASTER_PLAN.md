# Despliegue guiado — master plan

> **Status:** PENDING · 2026-10-15 · D1–D6 fijadas por el usuario; P1–P14 abiertas, todas con recomendación.

**Disparador:** desplegar `veltylabs/misitio` exige hoy pasos manuales repartidos en tres documentos
(`misitio/docs/DEPLOY.md`, `CI_TOKEN.md`, `PLAN_INFRA_BLOQUEADA.md`): crear D1 y R2 a mano, crear un
token de Cloudflare a mano, `gh secret set` cuatro veces, cargar secretos del Worker en el dashboard.
El último intento falló porque el token hecho a mano no tenía `Workers Scripts:Edit`
(`PUT workers/scripts/misitio → 403`), y nada lo detectó antes del deploy.

**Objetivo:** cada vez que se quiere desplegar, `webtyp/app` abre un wizard que **sabe qué falta**,
lo pide (los secretos en `webtyp/keyring`), lo crea donde hay API, lo entrega a cada destino
(CI, Worker) y despliega. El desarrollador nunca edita `.env`, nunca corre `gh secret set`, nunca
crea un token a mano.

**Alcance:** `webtyp/deploy`, `webtyp/app`, `webtyp/wizard`, `webtyp/goflare`, el destino GitHub,
`webtyp/keyring` (consumo, sin cambios), y `veltylabs/misitio` como primer caso real.

**Relación con otras olas:** independiente de UI_CONSISTENCY, API_SURFACE, NO_REFLECTION y
SOURCE_SELECTION. Consume KEYRING_DEPENDENCY_FREE (keyring inyectable, ya publicado). Es el
mecanismo con el que SECURITY_BY_DEFAULT §9 (misitio) entregará `IAM_CLIENT_PRIVATE_KEY` al Worker;
no la bloquea.

---

## Decisiones

Fijadas por el usuario el 2026-10-15; no se reabren.

| # | Decisión | Consecuencia |
|---|---|---|
| D1 | **Los secretos viven solo en `webtyp/keyring`** y se ingresan una vez en el wizard de `webtyp/app`. Nunca en `.env`, repo, plan, shell ni chat. | Skill **secrets** creada (devskills v0.0.14). Se borra todo doc que diga `gh secret set X` o "pon X en `.env`". |
| D2 | **`webtyp/app` es el orquestador**: no sabe *cómo* se despliega nada; recibe inyectadas las formas de desplegar y las configura. | `app` es la raíz de composición: es el único que importa `goflare` y el destino GitHub. |
| D3 | **`webtyp/deploy` no depende de ninguna forma de desplegar.** Solo define el contrato. | Se borran de `deploy`: `providers/cloudflare/`, `pusher_cloudflare.go`, el registro global `RegisterPusher`/`init()`, las claves `CF_*` y `goflare/<proyecto>` de `store.go`, el mapeo heredado `"cloudflare"→"cloudflarePages"` de `wizard.go:48`. `deploy/go.mod` deja de requerir `goflare`. |
| D4 | **Cada proyecto declara cómo se despliega y qué necesita.** webtyp no lo infiere ni lo trae en código. | Formato: P1. |
| D5 | **Una sola interfaz común** para todas las herramientas involucradas (skill **api-design**). | Forma: P7. |
| D6 | **El wizard dice qué falta** antes de desplegar, con el flujo ya establecido para cada caso. | Diagnóstico (estilo `doctor`) antes de cada deploy: P11. |

---

## Preguntas abiertas

Responde inline con `respuesta: X` (o texto libre) bajo cada una; luego «consolida el master».

### P1 — ¿Dónde declara el proyecto su despliegue?

- **A) (recomendada)** Archivo Go `config/deploy.go` con un valor literal tipado, **leído por AST**
  (sin ejecutar el proyecto), igual que `config/routes.go` (routescan) y `RenderImages()` (skill
  **ui-images**). Ejemplo misitio:
  ```go
  func Deploy() deploy.Plan {
      return deploy.Plan{
          Target: goflare.Worker{Name: "misitio", Domain: "misitio.velty.cl", D1: "DB", R2: "ASSETS"},
          CI:     github.Actions{Repo: "veltylabs/misitio"},
          Secrets: []deploy.Secret{
              {Key: "IAM_CLIENT_PRIVATE_KEY", What: "clave con la que misitio firma sus llamadas a iam", Source: deploy.Generated, To: deploy.Runtime},
          },
      }
  }
  ```
- B) `deploy.yaml` (hoy existe en `deploy/config.go` para el Puller). Sin tipos: un typo se
  descubre al desplegar.
- C) Inferirlo del `.github/workflows/deploy.yml` (inputs de la action `goflare`). Dos fuentes:
  el workflow es una salida, no la declaración.

Evidencia: el ecosistema ya eligió "declaración Go leída por AST" dos veces; SST (`sst.config.ts`)
y Encore (infra desde el código) son el mismo patrón en la industria. **Consecuencia de A:** el
`.github/workflows/deploy.yml` pasa a ser **generado** desde la declaración (`ghaction` ya existe
para eso, `ghaction/README.md`), y se borra `deploy.writeGHAWorkflow` (`deploy/wizard.go:113`).

### P2 — ¿Quién ejecuta el deploy?

- **A) (recomendada)** **Solo CI** (push a `main`). El wizard aprovisiona y entrega secretos; su
  botón "Desplegar" hace push o `workflow run` y muestra el run en vivo en la TUI.
- B) CI y también local desde webtyp. Dos caminos para lo mismo (ledger > 0).
- C) Solo local. Pierde trazabilidad y repetibilidad; obliga a tener TinyGo y credenciales en cada
  máquina.

Evidencia: misitio e iam ya despliegan por `tinywasm/goflare@v1` en Actions; Vercel, Netlify y
Cloudflare Pages hacen del push el deploy. Fly/Railway ofrecen ambos y su documentación empuja a CI
para producción.

### P3 — ¿Cómo llega un secreto del keyring a su destino?

- **A) (recomendada)** El wizard lo **copia desde el keyring** a cada destino declarado mediante la
  API del destino: GitHub Actions secrets (cifrado sellado que exige la API de GitHub) y secretos
  del Worker (API de Cloudflare). Vuelve a copiarlo solo cuando cambia en el keyring.
- B) CI lo lee en cada run de un gestor externo (1Password, Doppler, Infisical), como los adaptadores
  de Kamal 2 (`.kamal/secrets`). Agrega un servicio pago y otra credencial que proteger.
- C) Manual (hoy). Es el origen del 403 de misitio.

Nota: los secretos de GitHub son de solo escritura. El wizard puede comprobar que **existen** por nombre,
no su valor; para saber si están al día guarda en el keyring la huella (hash) de lo último que
entregó.

### P4 — Credenciales de Cloudflare

- **A) (recomendada)** Un **token de arranque** por cuenta, creado una sola vez con la plantilla
  oficial "Create additional tokens", guardado en el keyring (global, no por proyecto). Con él el
  wizard **crea un token por proyecto con exactamente los permisos que la declaración implica**
  (Worker → `Workers Scripts:Edit`; D1 → `D1:Edit`; R2 → `R2:Edit`; dominio → `Workers Routes` +
  zona; rate limit → `Zone WAF:Edit`) y lo entrega a GitHub. Cloudflare recomienda limitar el de
  arranque con TTL o filtro de IP.
- B) El desarrollador crea cada token a mano con una checklist de permisos. Es el mecanismo actual;
  ya falló.
- C) Un token global con todos los permisos en cada repo. Un repo comprometido compromete la cuenta.

Evidencia: el esqueleto de A ya existía (`CF_BOOTSTRAP_TOKEN`, `deploy/wizard.go:21`) pero nunca se
implementó. `wrangler login` usa OAuth, que Cloudflare no ofrece a herramientas de terceros: el
token de arranque es la alternativa que Cloudflare documenta.

### P5 — Credenciales de GitHub

- **A) (recomendada)** Reusar la sesión de `gh` (`gh auth token`); `devflow`/`codejob` ya dependen de
  ella. Si falta, el wizard lanza `gh auth login` (flujo de dispositivo).
- B) Un PAT propio en el keyring. Otra credencial que caduca y que hay que escopear a mano.

### P6 — ¿Para qué GCP?

**Para misitio, para nada.** misitio ya no autentica con Google: delega en iam
(commit `ff120d0` "consume veltylabs/iam remotely"; `grep -rn GOOGLE --include=*.go misitio` →
vacío). La tabla `GOOGLE_*` de `misitio/docs/DEPLOY.md:74-76` está desactualizada y se borra.
**Solo iam** necesita un cliente OAuth de Google, y **Google no tiene API pública para crear clientes
OAuth web**: la IAP OAuth Admin API quedó obsoleta el 2025-01-22 y la consola es el único camino.

- **A) (recomendada)** Paso guiado: el wizard muestra la URL exacta de la consola y el valor de
  "Authorized redirect URI" (calculado desde el dominio declarado), pide el client id/secret
  (`Sensitive`), los **valida** contra el endpoint de token de Google y los guarda en el keyring.
- B) Integrar `gcloud` y automatizar lo automatizable (proyecto, APIs habilitadas). Agrega una
  dependencia pesada para ahorrar un paso que igual es manual.

### P7 — Forma de la interfaz común (D5)

- **A) (recomendada)** `webtyp/deploy` contiene **solo tipos y contratos**, sin implementaciones:
  - `deploy.Plan` — lo que el proyecto declara (P1).
  - `deploy.Secret{Key, What, Source, To}` — los tres textos que exige la skill **documentation**
    viajan en el tipo, así el wizard y los docs los muestran sin redactarlos de nuevo.
  - `deploy.Target` — una forma de desplegar: `Check(plan) []deploy.Missing`, `Provision(plan, secrets)`,
    `Release(plan)`. Lo implementan `goflare` (Worker), y mañana SSH, Pages, etc. **D3:** `deploy`
    no importa ninguno.
  - `deploy.Destination` — recibe secretos: `Has(key)`, `Put(key, value)`. Lo implementan
    GitHub Actions y los secretos del Worker.
  - `deploy.Missing` — un ítem pendiente con su texto para el usuario y el paso de wizard que lo
    resuelve.
- B) Un único `Provider` que lo hace todo (lo actual: `deploy/provider.go`, que mezcla build,
  deploy, wizard y eventos de `devwatch`). Viola una preocupación por tipo y obliga a cada
  implementación a todo.

Nombres (prueba del novato): `Plan`, `Target`, `Destination`, `Missing` se leen como frases ("el
plan", "el destino del secreto", "lo que falta"). Se descartan `Sink`, `Pusher` y `Puller`
(jerga).

### P8 — Dos almacenes de secretos hoy

Hoy conviven el **`deploy.SecureStore`** (servicio `"webtyp-deploy"`, decide qué es secreto con una
lista fija de nombres, `deploy/store.go:30`) y la **pestaña SECRETS de `app`** (keyring por módulo,
`app/missing_secrets.go`). `CF_BOOTSTRAP_TOKEN` ni siquiera está en esa lista, así que acabaría en
`.env`.

- **A) (recomendada)** Uno solo: el keyring por proyecto de `keyring/auto.OpenForModule`, más una
  entrada global por cuenta para el token de arranque de Cloudflare (P4). Se borran `SecureStore`,
  `sensitiveKeys` y `KeyringServiceName`.
- B) Mantener ambos. Dos formas de hacer lo mismo.

### P9 — `KEY=keyring://` en `.env`

- **A) (recomendada)** La declaración (P1) es la **única** lista de secretos. `.env` queda solo con
  preferencias no secretas (`dev_mode`, `browser_*`…, como el `.env` actual de misitio). `webtyp/env`
  resuelve una clave declarada como secreta directamente desde el keyring. Se borra el marcador
  `keyring.Scheme` y `app.MissingSecrets` pasa a leer la declaración.
- B) Mantener el marcador como declaración de secretos de runtime local, y la declaración Go solo
  para despliegue. Dos listas de secretos por proyecto.

### P10 — Puller de Windows, SSH y webhook dentro de `deploy`

Son formas de desplegar; con D3 no pueden seguir en `deploy`. El Puller además no está implementado
en Linux (`deploy/manager_default.go`: "not implemented on linux").

- **A) (recomendada)** Moverlos a su propio repo (`webtyp/selfhost`), que implementa
  `deploy.Target`; se mantienen solo si existe un consumidor real (verificar en
  `veltylabs/*` antes de despachar).
- B) Borrarlos si no hay consumidor. Recuperables desde el historial de git.

### P11 — ¿Cuándo y dónde aparece el wizard?

- **A) (recomendada)** La pestaña **DEPLOY** de la TUI corre el diagnóstico al abrir el proyecto y
  muestra una checklist (✅ / ❌ con el texto de `deploy.Missing`). Cada ❌ con solución automática
  se resuelve con un paso del wizard; "Desplegar" se habilita cuando no queda ninguno. Una
  herramienta MCP `deploy_status` expone **la misma checklist sin valores**, para que el agente
  diga qué falta sin pedir secretos (skill **secrets**).
- B) Solo un comando `webtyp deploy` en la terminal.

Precedente: `flutter doctor`, `brew doctor`, `fly launch` (detecta, pregunta solo lo que falta,
crea lo que falta).

### P12 — Recursos de infraestructura (D1, R2, dominio)

- **A) (recomendada)** El `Target` los **crea si faltan** (idempotente) con los nombres de la
  declaración, como `fly launch` y SST. `goflare` ya resuelve dominio y rate limit
  (`goflare/cloudflare.go:306,360,426`); faltan D1 y R2.
- B) Manual con instrucciones. Es lo que hoy obliga a copiar IDs a secretos de GitHub
  (`D1_DATABASE_ID`, `R2_BUCKET_ID`), que **ni siquiera son secretos**: con A desaparecen como
  secretos y se resuelven por nombre.

### P13 — ¿Dónde vive el destino GitHub?

- **A) (recomendada)** Un repo nuevo, `webtyp/github` (secretos de Actions, `workflow run`,
  seguimiento del run), que implementa `deploy.Destination`. `devflow` es un conjunto de CLIs, y
  `ghaction` declara expresamente que no tiene conocimiento de dominio (solo genera YAML).
- B) Dentro de `devflow`, que ya usa `gh`. `app` arrastraría todo `devflow` para un solo concepto.

### P14 — `DEV_AUTOLOGIN` en `.env`

La skill **dev-autologin** indica poner credenciales JSON reales en `.env`
(`dev-autologin/SKILL.md:24-26`). Contradice D1.

- **A) (recomendada)** Pasa al keyring como cualquier secreto declarado (P9); el wizard lo pide.
  Se actualiza la skill.
- B) Excepción documentada por ser solo de desarrollo. Una excepción es la puerta por la que se
  filtra el siguiente secreto.

---

## Evidencia — estado actual

| Hecho | Dónde |
|---|---|
| `deploy` importa `goflare` y tiene un proveedor Cloudflare dentro | `deploy/go.mod:8`, `deploy/providers/cloudflare/provider.go` |
| Registro global de pushers en `init()` | `deploy/pusher.go:21`, `deploy/pusher_cloudflare.go:5` |
| Wizard con menú impreso a mano y 5 pasos "proxy" fijos porque `wizard` no admite pasos dinámicos | `deploy/wizard.go:28-104` |
| Lista fija de qué es secreto; el token de arranque no está | `deploy/store.go:30-42` |
| `goflare` solo lee el token desde la variable de entorno (CI) | `goflare/auth.go:12-24` |
| Los IDs de D1/R2 se copian a mano como "secretos" de GitHub | `misitio/docs/DEPLOY.md:54-57,107-108` |
| Token de Cloudflare mal escopeado → 403 en cada deploy desde v0.0.7 | run 38746286153; `misitio/docs/PLAN_INFRA_BLOQUEADA.md` |
| `GOOGLE_*` documentados para misitio, sin uso en el código | `misitio/docs/DEPLOY.md:74-76` |

## Industria — cómo despliegan otros

| Herramienta | Declaración | Secretos | Aprovisiona | Diagnóstico |
|---|---|---|---|---|
| Vercel / Netlify | detecta el framework; `vercel.json` opcional | UI o `vercel env add`, guardados en la plataforma | sí (proyecto) | log del build |
| Fly.io | `fly.toml` generado por `fly launch` | `fly secrets set`, cifrados en la plataforma, nunca en archivo | sí (app, Postgres, IPs) | `fly doctor` |
| Kamal 2 (Rails) | `config/deploy.yml` | `.kamal/secrets` obtiene valores de 1Password/Bitwarden con adaptadores | no (servidores propios) | `kamal config` |
| SST / Encore | **código** (`sst.config.ts`; infra inferida del código Go en Encore) | `sst secret set` → gestor del proveedor de nube | sí, todo | errores tipados al compilar |
| Cloudflare Wrangler | `wrangler.toml` | `wrangler secret put` | parcial (`d1 create` a mano) | — |

Patrón común: (1) **login una vez** con el proveedor, (2) un comando que **detecta, pregunta solo lo
que falta y crea**, (3) secretos guardados en el almacén del destino, **nunca en archivos**, (4)
diagnóstico antes de desplegar. Donde webtyp difiere: Cloudflare no ofrece OAuth a terceros (de ahí
el token de arranque de P4), y webtyp declara en Go leído por AST (como SST/Encore, pero sin
ejecutar el proyecto).

## Sugerencias de experiencia de desarrollo (no bloquean esta ola)

1. **Docs de despliegue generados desde la declaración.** `DEPLOY.md` de misitio quedó
   desactualizado (`GOOGLE_*`); si la tabla de secretos sale de `deploy.Secret.What/Source`, no
   puede divergir.
2. **Deploys de vista previa por PR** (versiones de Worker de Cloudflare con URL propia), como
   Vercel/Netlify. Ver un PR de Jules funcionando sin mergearlo.
3. **Rollback de un paso** desde la TUI (Cloudflare guarda las versiones del Worker).
4. **Run de CI en vivo en la TUI** en vez de ir a GitHub a leer logs (lo que hubo que hacer hoy
   para ver el 403).

## Puerta api-design (borrador, se completa al cerrar P1/P7)

- **Precedentes:** SST, Encore, Fly, Kamal, Vercel (tabla anterior).
- **Nombres:** `deploy.Plan`, `deploy.Target`, `deploy.Destination`, `deploy.Secret`,
  `deploy.Missing` (P7).
- **Balance de complejidad** (misitio, con todas las recomendaciones):
  ```
  Conceptos a aprender            +2 (Plan, Secret)  / −4 (Pusher, Puller, SecureStore, keyring://)
  Archivos a tocar para desplegar +1 (config/deploy.go) / −2 (deploy.yml a mano, DEPLOY.md a mano)
  Pasos manuales                  0 / −9 (crear D1, R2, token, 4× gh secret set, 2× secretos de Worker)
  Formas de hacer lo mismo        −2 (dos almacenes de secretos, dos listas de secretos)
  ```
- **Dónde va:** contrato en `deploy`; Cloudflare en `goflare`; GitHub en `webtyp/github` (P13);
  composición en `app` (D2).
- **Qué borra:** ver D3, P8, P9, P10, y en misitio las secciones manuales de `DEPLOY.md`,
  `CI_TOKEN.md` y `PLAN_INFRA_BLOQUEADA.md` que el wizard reemplaza.

## Librerías (se detalla al cerrar las preguntas)

| Repo | Cambio | Tipo |
|---|---|---|
| `webtyp/wizard` | pasos agregados en tiempo de ejecución (elimina los 5 proxy de `deploy/wizard.go`) | **gate** de `app` |
| `webtyp/deploy` | queda solo el contrato (P7); borra implementaciones (D3, P8, P10) | **gate** de goflare, github, app |
| `webtyp/goflare` | implementa `deploy.Target` (Worker); token por proyecto desde el token de arranque (P4); D1/R2 si faltan (P12); `deploy.Destination` para secretos del Worker | paralelo con github |
| `webtyp/github` (nuevo, P13) | `deploy.Destination` para secretos de Actions; `workflow run` y seguimiento | paralelo con goflare |
| `webtyp/env` | resuelve claves declaradas como secretas desde el keyring (P9) | paralelo |
| `webtyp/app` | pestaña DEPLOY: diagnóstico + wizard + run en vivo; MCP `deploy_status`; inyecta goflare y github | tras goflare y github |
| `veltylabs/misitio` | `config/deploy.go`; borra los pasos manuales de sus docs | último, caso de aceptación |

**Aceptación de la ola:** en una máquina con solo `gh auth login` hecho y el token de arranque de
Cloudflare en el keyring, abrir misitio en webtyp, pasar el wizard sin salir de la TUI, y ver el
Worker respondiendo en `misitio.velty.cl`, sin ningún secreto en `.env`, en el repo ni en el chat.
