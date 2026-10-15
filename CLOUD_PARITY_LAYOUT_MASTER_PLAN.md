# Layout para proyectos cloud — master plan

> **Status:** PENDING · 2026-10-15 · D1–D3 fijadas por el usuario; P1–P4 abiertas, todas con recomendación.

**Disparador:** misitio es un proyecto **100 % cloud** (Cloudflare Worker), pero hay que probarlo
en local con `web/server.go`. Para que eso compile, `config/` terminó con la composición del
servidor, el cliente de iam, adaptadores y UI metidos a la fuerza, y aparecieron `api/` y
`routes/` como parches. El navegador importa `config/` y arrastra el servidor: `client.wasm` llegó a
682 KB gzip contra un presupuesto de 512 KB (2026-10-15). Detalle de las fallas: P27 de
[DEPLOY_WIZARD_MASTER_PLAN.md](DEPLOY_WIZARD_MASTER_PLAN.md) (M1–M8).

**Objetivo:** una ubicación de archivos que **por sí sola** sea la configuración correcta: cada
binario (navegador, Worker, servidor local) compila solo lo suyo, lo de producción se prueba en
local con el mismo código, y no hay etiquetas nuevas que aprender.

**Relación con otras olas:** reemplaza la regla de etiquetas de las skills **app-modules**
(`server.go` → `//go:build !wasm`) y **project-layout** ("`config/` es la raíz de composición").
Cambia D34 de DEPLOY_WIZARD (dónde vive la declaración de despliegue, P3). misitio es el caso de
aceptación; su plan siguiente espera esta ola.

---

## Decisiones

| # | Decisión | Consecuencia |
|---|---|---|
| D1 | **No se agregan etiquetas de build nuevas** (usuario, 2026-10-15). | P27 de DEPLOY_WIZARD (etiqueta `edge`) queda rechazada. |
| D2 | **`config/` es la carpeta compartida y agnóstica del despliegue**: la importan el navegador, el Worker y el servidor local. | Lo que no es de los tres sale de `config/`. |
| D3 | **Los `main` (`edge/main.go`, `web/server.go`, `web/client.go`) asumen cada uno su responsabilidad**: elegir el entorno y componer. | La composición deja de vivir en `config/`. |

---

## Diagnóstico — lo que no estábamos viendo

1. **`//go:build !wasm` significa "servidor" solo por casualidad.** En un proyecto nativo el
   servidor es el único binario no-wasm, y la skill **app-modules** se apoya en eso
   (`server.go` → `!wasm`). En un proyecto cloud el Worker **también** es wasm: la etiqueta deja
   de separar y todo cae en el mismo saco.
2. **En Go la frontera que siempre funciona es el paquete**: un paquete que nadie importa no se
   compila. Medido en misitio: importar `config` (con la composición adentro) cuesta 549.952 B gzip
   en el navegador; importar un paquete hoja (`api`) cuesta 118.032 B. TinyGo no descarta lo que
   una importación arrastra (inicialización y variables de paquete), aunque no se use.
3. **El ecosistema ya tiene la forma correcta**, pero solo para módulos reutilizables: la skill
   **app-modules** pide `<m>` (dominio, servidor) y `<m>/ui` (vista, navegador). Los módulos
   locales de cada app usan otra forma (archivos `server.go`/`browser.go` con etiquetas). Dos
   formas para lo mismo.
4. **Lo único que cambia entre producción y local son los adaptadores** (D1 o SQLite, R2 o un
   directorio, bindings o `.env`). El resto debería ser literalmente el mismo código.

## Preguntas abiertas

Responde inline con `respuesta: X` (o texto libre) y luego «consolida el master».

### P1 — ¿Cómo se separa navegador de servidor sin etiquetas?

- **A) (recomendada) Frontera por paquete, igual que los módulos reutilizables.**

  ```text
  config/                 compartido, SIN etiquetas: nombre, rutas (paths), IDs, DTOs de cable,
                          política, lista de módulos, tema. No importa nada del repo.
  modules/<m>/            mitad servidor del módulo: handlers, operaciones, modelos.
  modules/<m>/ui/         mitad navegador: Browser(...), ID, Label, css.go/svg.go.
  modules/server.go       registro del servidor: Register(r router.Router, b Bindings)
                          — la tabla de rutas con su acceso, una sola vez.
  web/client.go           main del navegador: compone con config + modules/*/ui.
  web/server.go           main local: adaptadores locales + modules.Register.
  edge/main.go            main del Worker: D1/R2 + modules.Register.
  ```

  Regla única, para módulos locales y reutilizables: **el navegador importa solo `config` y paquetes
  `ui`**. Se verifica con un test (`go list -deps ./web/` no contiene `webtyp.com/orm` ni ningún
  `modules/<m>` sin `/ui`) y con `router/layoutscan`.
  Por qué: cero conceptos nuevos (es la forma que ya tienen los módulos reutilizables), el
  compilador hace cumplir la frontera, y `config/` vuelve a ser lo que dice su nombre.
  Precedentes: Next.js separa con `server-only`/`client-only` y directorios; Remix/React Router
  con `.server`/`.client`; Rails con `app/` vs `app/javascript`; Go, por paquete.
  Balance: conceptos −1 (la tabla de etiquetas de **app-modules**), −2 (`api/`, `routes/`);
  formas de hacer un módulo −1 (local y reutilizable igual).
- B) Mantener las etiquetas actuales y aceptar que en proyectos cloud `config/` mezcle. Es el
  estado actual que se quiere dejar.
- C) Etiqueta `edge` en el Worker. Rechazada por D1.

### P2 — ¿Cómo se prueba en local lo que corre en la nube?

- **A) (recomendada) Mismo código, adaptadores locales con el mismo contrato.** Un paquete nuevo
  `webtyp.com/cloudflare/local` da los bindings de Cloudflare en local: D1 sobre un archivo
  SQLite (`.build/local.db`; D1 **es** SQLite, mismo dialecto), R2 sobre un directorio
  (`.build/r2/`), variables desde `.env` y el keyring. `web/server.go` de un proyecto cloud =
  `local` + `modules.Register` + `httpd`; `edge/main.go` = `d1`/`r2` + `modules.Register` +
  `edge.Serve`. Lo único distinto entre los dos `main` son los adaptadores.
  Precedentes: Miniflare/`wrangler dev` (D1 local sobre SQLite, R2 sobre disco), los emuladores de
  Firebase, LocalStack; principio X de 12-factor (paridad desarrollo/producción) y IV (servicios
  de respaldo como recursos intercambiables).
  Hoy misitio usa `mem` en local (`misitio/web/server.go:24`), un almacenamiento distinto del de
  producción: los errores de SQL solo aparecen en la nube.
- B) Ejecutar el Worker real en local (`workerd`, el runtime abierto de Cloudflare) y borrar
  `web/server.go` en proyectos cloud. Paridad máxima, pero cada cambio exige recompilar con TinyGo
  (~20 s, medido en `misitio/tests/edge_size_test.go`), depende de un binario externo y pierde el
  depurador nativo. Puede sumarse después como "vista previa local" antes de desplegar.

### P3 — ¿Dónde vive la declaración de despliegue sin etiqueta?

Hoy (D34 de DEPLOY_WIZARD) es `config/deploy.go` con `//go:build deploy`, justamente para que
`goflare` no entre a ningún binario. Con D1 y D2 no puede quedar en `config/`.

- **A) (recomendada)** **`config/deploy/deploy.go`**, un subpaquete que **nadie importa**: no entra
  a ningún binario, `go vet ./...` lo compila (errores de tipo al momento), y `deploy.Scan` lo lee
  por AST. Mismo patrón que `config/layouts/` (subpaquete permitido por **project-layout**).
  Requiere cambiar `deploy.DefaultFile` y quitar `ErrMissingBuildTag` en el plan de `deploy`, que
  está corriendo: se le manda el cambio por `codejob reply` al cerrar esta pregunta.
- B) Dejar `//go:build deploy`. Es una etiqueta más, contra D1.

### P4 — ¿Aplica a todos los proyectos o solo a los cloud?

- **A) (recomendada)** **A todos.** Un solo layout que un junior (o un agente) aprende una vez. En
  un proyecto nativo `web/server.go` usa adaptadores nativos (`sqlite`, `postgres`) y no hay
  `edge/main.go`; el resto es igual. El generador de `web/server.go` (skill **app-server-main**)
  pasa a llamar `modules.Register(r, b)` con los adaptadores que corresponden: locales de Cloudflare
  si existe `edge/main.go`, nativos si no.
- B) Solo a proyectos cloud. Dos layouts para aprender y dos generadores.

---

## Librerías y repos afectados (se detalla al cerrar las preguntas)

| Repo | Cambio con las recomendaciones |
|---|---|
| devskills | **project-layout**, **app-modules**, **app-server-main**, **app-rest-edge**: el nuevo layout; se borra la tabla de etiquetas `server.go`/`browser.go` |
| `webtyp/cloudflare` | paquete `local` (D1 sobre SQLite, R2 sobre disco, variables) — P2 |
| `webtyp/server` | generador de `web/server.go`: `modules.Register(r, b)` con adaptadores según el tipo de proyecto — P4 |
| `webtyp/router` | `layoutscan`: verifica la regla de P1 |
| `webtyp/deploy` | `DefaultFile = "config/deploy/deploy.go"`, sin `ErrMissingBuildTag` — P3 |
| `webtyp/app` | lo que lea `config/routes.go` o los registros de módulos |
| `veltylabs/misitio` | re-layout completo (M1–M8 de DEPLOY_WIZARD P27); primer caso |
| `veltylabs/iam`, `veltylabs/mjosefa-cms` | migración al layout, en una segunda fase |
