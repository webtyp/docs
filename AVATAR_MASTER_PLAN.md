# Avatar vectorial modular (`webtyp/avatar`) — master plan

> **Status:** IN PROGRESS · 2026-10-20 · Reabierto: el motor en vivo (v0.1.0) funciona pero mezcla actividad/emoción en un solo `State`, el parpadeo/habla no son fluidos, y falta cableado para acompañar la integración con `webtyp/agent`. Esperando respuestas P5–P9.

**Disparador:** el prototipo del avatar de Cote en `veltylabs/mjosefa-cote/docs/avatar/preview.html` demostró que el flujo de dibujo ideal es mantener un único archivo SVG puro editable en Inkscape (`docs/avatar/img/cote-base-v1.2.svg`, dividido en capas anatómicas `#torso`, `#head`, `#hair`, `#brows`, `#eyes`, `#nose`, `#mouth`, `#poses`, `#extras`), pero mantener la lógica de estados, secuencias de habla, parpadeo y bostezo aleatorio en CSS/JS suelto (~600 líneas en `preview.html` con 25 `@keyframes` y timers manuales) es frágil y no tipado.

**Objetivo:** crear la librería tipada `webtyp.com/avatar` (`webtyp/avatar`, con `gonew`) integrada a los contratos de `webtyp/svg`, `webtyp/dom`, `webtyp/widget` y `webtyp/css`, de modo que el archivo `.svg` de Inkscape siga siendo la única fuente de dibujo y el consumidor (`veltylabs/mjosefa-cote`, probado desde `web/client.go`) arme y controle el avatar desde Go con una API mínima.

**Relación con otras olas:** independiente de `SECURITY_BY_DEFAULT`, `DEPLOY_WIZARD` y `API_SURFACE`. Respeta las reglas de `UI_CONSISTENCY` (`widget/style` como única fuente de CSS en `css.go`) y de `BUILD_DESTINATIONS` (`css.go` y `svg.go` con `//go:build !wasm`; cliente en `web/client.go` hasta que corra la migración de `BUILD_DESTINATIONS`).

---

## Decisiones

### Fijadas por el usuario (2026-10-18)

| # | Decisión | Consecuencia |
|---|---|---|
| D1 | **El archivo `.svg` es 100 % puro para Inkscape.** No contiene bloques `<style>`, selectores `[data-state]` ni `var(--...)` que Inkscape no interpreta; usa capas nativas `inkscape:groupmode="layer"`, `style="display:none"` para ocultar variantes mientras se edita y `fill="#ffffff"` en los fondos de oclusión (`*-bg`). | Todo cambio de diseño o proporción de manos/rasgos se hace abriendo el `.svg` en Inkscape y guardando; el pipeline de Go se encarga de limpiar y transformar las capas al construir el sprite. |
| D2 | **Anatomía modular e invariante para escalar a múltiples avatares.** `#hair`, `#head` y `#torso` están separados y nunca se deforman entre estados; `#poses` contiene únicamente las manos superpuestas (con capa `*-bg` de oclusión + capa `*-ink` de trazo). | Cambiar de sexo, peinado o vestimenta entre avatares consiste en reemplazar el símbolo del slot `Hair` o `Torso` sin tocar ojos, cejas, bocas ni manos. |
| D3 | **Nueva librería `webtyp.com/avatar` (`webtyp/avatar`, creada con `gonew`).** Provee el componente `dom` del avatar, su catálogo anatómico, la máquina de estados semánticos (`Normal`, `Greeting`, `Speak`, `Think`, `Listen`, `Fatigue`, `Sad`, `Astonishment`, `Anger`) y sus estilos en `css.go`. | Desaparece la necesidad de escribir CSS o JS ad-hoc por proyecto para animar un avatar. |
| D4 | **`veltylabs/mjosefa-cote` es el primer consumidor y se prueba en `web/client.go`.** | `mjosefa-cote` declara el avatar de Cote a partir de `cote-base-v1.2.svg` y `web/client.go` monta una vista interactiva para probar todos los estados, la locución (`Speak`) y el intercambio de capas. |
| D5 | **Secuencias automáticas listas para un agente.** El consumidor o el agente solo pide un estado semántico (`SetState(avatar.Think)`, `SetState(avatar.Fatigue)`, `Speak("...")`) sin accionar bocas, ojos o manos por separado. | En `Speak` alternan automáticamente manos (`talk-a`/`talk-o`), ojos, cejas, globos y visemas; en `Think` se animan secuencialmente las burbujas y puntos de procesamiento; en `Fatigue` corre un bostezo lento (~5.5s) con pausa aleatoria de **10 a 30 segundos** entre uno y otro. |
| D6 | **Geometría SVG en `!wasm` (`svg.go`), solo referencias en WASM (skill `ui-icons`).** | `sitec` extrae `IconSvg() *sprite.Sprite` en tiempo de build e inyecta el `<svg style="display:none"><symbol .../></svg>` en `index.html` (`webtyp/sitec/emit_core.go:696-698`); el binario WASM solo contiene los IDs de cada símbolo (`svg.Icon`), nunca los 115 KB de trazados `<path>`. |

---

## Preguntas abiertas

Responde inline con `respuesta: X` (o texto libre) y luego «consolida el master».

### P1 — ¿Dónde vive el extractor que convierte las capas del `.svg` de Inkscape en `<symbol>` individuales de `*sprite.Sprite`?

En `webtyp/svg/sprite/file.go:12-23`, `(*Sprite).AddFile(id, svgFile)` hoy mete todo el archivo `.svg` dentro de un único `<symbol id="...">`. Como el Shadow DOM de `<use href="#..."/>` impide por especificación que CSS o `dom` oculten, muestren o transformen nodos internos (`#eyes-normal`, `#mouth-talk-a`, `#pose-greeting`), el avatar necesita que cada variante anatómica (`torso-normal`, `hair-normal`, `eyes-normal`, `pose-greeting`, `wave-inner`, `zzz-1`, etc.) se emita como su propio `<symbol id="<prefix>-<variant>" viewBox="0 0 512 512">`, limpiando el `style="display:none"` que deja Inkscape, preservando cualquier `transform="matrix(...)"` de Inkscape (`cote-base-v1.2.svg:434`) y convirtiendo el `fill="#ffffff"` de las capas de oclusión `*-bg` (`cote-base-v1.2.svg:437,458,478,499`) en `fill="var(--color-surface, #ffffff)"` para que las manos tapen el pelo/hombro tanto en tema claro como en tema oscuro.

- **A) (recomendada)** **En `webtyp/svg/sprite` (`sprite/layers.go`, `//go:build !wasm`)**: agregar `(*Sprite).AddLayers(prefix, svgFile string) error` (o `AddLayersBytes(prefix string, svgData []byte) error`). Parsea el XML del SVG de Inkscape, extrae cada variante con `id` dentro de las capas de primer nivel (`inkscape:groupmode="layer"` / `<g>`), limpia `style="display:none"` y atributos `inkscape:*`/`sodipodi:*`, conserva los `transform="..."` envolviendo el contenido del `<symbol>`, reemplaza `fill="#ffffff"` en paths `*-bg` por `fill="var(--color-surface, #ffffff)"` y registra cada `<symbol id="<prefix>-<id>">`.
  - *A favor:* una sola responsabilidad (**api-design** Gate 4: `webtyp/svg/sprite` ya es dueño de convertir archivos `.svg` en `<symbol>` con `AddFile` en `sprite/file.go:12`); reutilizable por cualquier otro componente con ilustraciones multicapa de Inkscape.
  - *En contra:* requiere publicar un tag menor en `webtyp/svg` como gate antes de `webtyp/avatar`.
- B) **Dentro de `webtyp/avatar` (`avatar/svg_loader.go`, `//go:build !wasm`)**, llamando a `(*Sprite).AddRaw(id, body, viewBox)` (`webtyp/svg/sprite/sprite.go:58-64`) sin tocar `webtyp/svg`.
  - *A favor:* un repo menos que despachar.
  - *En contra:* duplica el parseo de `viewBox` y XML que ya vive en `webtyp/svg/sprite/file.go:26-66` y `webtyp/svg/sanitize/sanitize.go:114-168`.

### P2 — ¿Cómo se reparten las animaciones entre Go (`webtyp/avatar` con señales `dom` + goroutines) y CSS (`webtyp/widget/style`)?

En `preview.html:95-228` hicieron falta 25 `@keyframes` con `steps(1, end)` (`speak-mouth-a`, `speak-mouth-o`, `speak-mouth-n`, `yawn-mouth-rest`, `yawn-mouth-mid`, `yawn-mouth-wide`, etc.) para conmutar la visibilidad de cada boca/mano/ojo por porcentajes. En `webtyp`, `webtyp/css` mantiene `keyframes(...)` y `rule(...)` privados al paquete (`webtyp/css/dsl.go:86,270`) y `webtyp/widget/style` solo expone recetas semánticas (`emit_flowdecls.go:97-216`).

- **A) (recomendada)** **Conmutación de variantes en Go (`dom.SignalString` + goroutine con `webtyp/time`) y solo micro-movimientos continuos en `webtyp/widget/style`.**
  - En Go (`webtyp/avatar`): el `<svg viewBox="0 0 512 512">` del componente tiene un `<use>` por cada capa anatómica (`Torso`, `Head`, `Hair`, `Brows`, `Eyes`, `Nose`, `Mouth`, `Pose`, `Extra1..3`) conectado con `BindAttr("href", sig)` (`webtyp/dom/element.go:301`). Una goroutine en `Init(ctx dom.Ctx)` (cancelada en `ctx.OnCleanup`) controla con `webtyp.com/time` qué variante se muestra en cada instante:
    - `Speak`: alterna sílabas (`mouth-talk-a` ↔ `mouth-talk-o` ↔ `mouth-normal`), manos (`pose-talk-a` ↔ `pose-talk-o`), cejas, mirada y globos de voz.
    - `Fatigue`: mantiene el reposo somnoliento y, tras un intervalo aleatorio de **10 a 30 segundos** (`10s + rand(0..20s)`), ejecuta las fases del bostezo de `5.5s` (`mouth-talk-o` → `mouth-astonishment` + `eyes-closed` + `nose-closed` + `brows-sad` → `mouth-talk-o` → `mouth-fatigue`).
    - `Think`, `Listen`, `Greeting`, `Sad`, `Astonishment`, `Anger`, `Normal`: activan sus combinaciones de rasgos y secuencias de extras.
  - En `webtyp/widget/style`: solo se agregan recetas genéricas de micro-movimiento continuo para partes (`Blink`, `Wave`, `Pulse`, `Float`, con soporte automático de `@media (prefers-reduced-motion: reduce)` en `emit_motion.go:12-90` y pivote en la base `transform-origin: 50% 100%` para las manos).
  - *A favor:* elimina de raíz los ~20 `@keyframes` de `steps(1, end)`; toda la lógica temporal (ritmo de habla, parpadeo, bostezo 10–30s random) queda escrita en Go tipado y se puede probar determinísticamente con `gotest` inyectando un reloj/semilla; `widget/style` solo recibe 3–4 recetas puras de movimiento.
  - *En contra:* requiere que WASM esté corriendo para animar los cambios de fotograma (en SSR estático renderiza el fotograma inicial del estado, que es exactamente lo esperado antes de hidratar).
- B) **Todo en `@keyframes` de CSS**, exportando un constructor general de keyframes en `webtyp/css` y `webtyp/widget/style` y usando Go únicamente para el timer aleatorio de 10–30s del bostezo.
  - *A favor:* las secuencias de `speak` corren sin ticks de Go.
  - *En contra:* traslada a Go las mismas 25 tablas de porcentajes CSS difíciles de mantener que motivaron este cambio, y obliga a abrir la superficie de `webtyp/css` y `webtyp/widget/style` con constructores de keyframes arbitrarios.

### P3 — ¿Cómo arma el consumidor su avatar y qué superficie pública expone `webtyp/avatar`?

El archivo `cote-base-v1.2.svg` ya sigue una convención fija de IDs por capa (`torso-normal`, `head-normal`, `hair-normal`, `brows-<variant>`, `eyes-<variant>`, `nose-<variant>`, `mouth-<variant>`, `pose-<variant>`, `extras-<variant>`).

- **A) (recomendada)** **Catálogo por convención `avatar.NewCatalog(prefix string)` con overrides fluidos por capa (`WithHair`, `WithTorso`, `WithHead`, `WithTiming`) + componente `avatar.New(cat)`**:
  ```go
  // 1. En el consumidor (mjosefa-cote):
  var CoteCatalog = avatar.NewCatalog("cote")
  // Para otro personaje o cambio de sexo basta con cambiar la capa deseada:
  // var AlexCatalog = CoteCatalog.WithHair(svg.Icon("alex-hair-short"))

  // 2. En la vista WASM (web/client.go):
  av := avatar.New(cote.CoteCatalog)
  av.SetState(avatar.Think)              // cambia de estado semántico
  av.Speak("Hola, soy Cote...")          // anima habla durante la locución y vuelve a Normal
  av.Stop()                              // detiene locución y vuelve a Normal
  ```
  Además, `av.SetLayerVisible(avatar.LayerHair, bool)` permite encender/apagar capas en vistas de depuración como `preview.html`.
  - *A favor:* pasa el *acid test* de **api-harness** (1 línea para declarar un avatar que sigue la plantilla de Inkscape; 1 método por intención; cero mapas en WASM; permite mezclar capas de distintos SVGs para crear múltiples avatares).
  - *En contra:* fija la convención de sufijos de IDs en las capas del SVG de Inkscape (documentada en el `README.md` y validada en `AddLayers` con error descriptivo si falta un ID requerido).
- B) **Struct literal explícito sin convención de sufijos**, donde el consumidor asigna manualmente cada uno de los ~40 campos `svg.Icon` (`Torso`, `Head`, `Hair`, `BrowsNormal`, `BrowsGreeting`, `EyesNormal`, etc.).
  - *A favor:* no exige nombres fijos dentro del SVG.
  - *En contra:* 40 líneas de boilerplate por cada avatar nuevo.

### P4 — ¿Cómo organizamos la integración en `veltylabs/mjosefa-cote` respecto a su `docs/PLAN.md` actual?

En `veltylabs/mjosefa-cote/docs/PLAN.md` (`lines 1-60`) hay un plan **no despachado** (sin `STATUS`, bloqueado esperando `webtyp.com/agenteval v0.3.0`) para actualizar `evals/`. Por otro lado, `cote-base-v1.2.svg` está hoy en `docs/avatar/img/cote-base-v1.2.svg`.

- **A) (recomendada)** Mover o referenciar `cote-base-v1.2.svg` desde `svg.go` (`//go:build !wasm`, `func IconSvg() *sprite.Sprite`) en el paquete raíz `cote`, exponer `cote.NewAvatar() *avatar.Avatar`, actualizar `web/client.go` con la vista completa de prueba del avatar (selector de los 9 estados, locución `Speak`/`Stop` y toggles de capas `#hair`/`#head`/`#torso`/`#poses`/`#extras`), eliminar `docs/avatar/preview.html` cuando `web/client.go` esté funcionando, y fusionar estas etapas en `veltylabs/mjosefa-cote/docs/PLAN.md` (regla **agents-workflow** → «A repo that already has a plan — merge, never queue»).
- B) Ejecutar el cambio de `mjosefa-cote` localmente con `LAST_PLAN_EXECUTED.md` una vez publicados `webtyp/svg`, `webtyp/widget` y `webtyp/avatar`, dejando `docs/PLAN.md` reservado para cuando salga `agenteval v0.3.0`.

---

## Evidencia en el código actual

1. **Extracción automática de `IconSvg()` y `RenderCSS()` con solo `web/client.go` (`webtyp/sitec`)**:
   - `sitec/reach.go:41-44` calcula la alcanzabilidad como la **unión** de `{"", ""}` (nativo) y `{"js", "wasm"}` (`web/client.go`).
   - `sitec/extract.go:244-306` ejecuta en Go nativo (`//go:build !wasm`) todos los `RenderCSS() *css.Stylesheet` e `IconSvg() *sprite.Sprite` alcanzables.
   - `sitec/emit_core.go:696-698` y `sitec/emit_html.go:48-59` inyectan el sprite `<svg aria-hidden="true" style="display:none"><symbol .../></svg>` directamente en `<body>` antes de `<div id="app">`.
2. **Limitación actual de `webtyp/svg/sprite` (`sprite/file.go:12-23`)**:
   - `(*Sprite).AddFile(id, svgFile)` extrae el interior de `<svg>...</svg>` como un único bloque crudo. No separa capas de Inkscape ni remueve `style="display:none"` de variantes ocultas.
3. **Limitación actual de `webtyp/widget/style` (`sheet_builder.go`, `emit_flowdecls.go:97-216`)**:
   - `style.Sheet` estiliza partes `.name__part` (`Part`, `When`, `WhenWithin`); no tiene recetas de micro-animación continua para partes de un avatar (`Blink`, `Wave`, `Pulse`, `Float`) ni pivote de transformación (`transform-box: fill-box; transform-origin: ...`).
4. **Reactividad de atributos en `webtyp/dom` (`element.go:301, 361`, `dom_frontend.go:737-751`)**:
   - `el.BindAttr("href", sigString)` actualiza en vivo el atributo `href` de un `<use>` en el DOM cuando `sigString.Set("#cote-mouth-talk-o")` cambia desde una goroutine en WASM, sin re-serializar el árbol del componente.

---

## Gate de diseño de API (`api-design`)

### 1. Prior art
1. **Rive (State Machines + Artboards)** — separa el archivo de diseño vectorial (con capas e inputs nombrados) de una máquina de estados declarativa donde la app solo dispara estados (`stateMachine.input("state").value = "think"`) sin manipular nodos internos. *Por qué diferimos:* Rive requiere un runtime propio en WASM/Canvas (~150–300 KB) y un formato binario propietario `.riv`; aquí el origen es un `.svg` estándar de Inkscape y el runtime es el propio `webtyp/dom` + CSS.
2. **Lottie (`lottie-web` / Bodymovin)** — exporta animaciones desde After Effects a JSON y las reproduce en SVG/Canvas. *Por qué diferimos:* el JSON de Lottie no es editable a mano ni en Inkscape, mezcla geometría con fotogramas en el bundle del cliente y no modela estados semánticos de un agente conversacional ni intervalos aleatorios (bostezo 10–30s).
3. **TalkingHead / Avataaars (React SVG Avatar libraries)** — componen un rostro a partir de props tipadas por rasgo (`topType="LongHair"`, `eyeType="Happy"`, `mouthType="Smile"`). *Por qué diferimos:* Avataaars codifica los `<path>` dentro de componentes JS (enviando toda la geometría al bundle del navegador y haciendo imposible editarlos en Inkscape); `webtyp/avatar` mantiene el `.svg` puro en Inkscape, extrae los `<symbol>` en el build (`!wasm`) y solo envía los IDs de referencia a WASM.

### 2. Novice-name test
- Paquete: `webtyp.com/avatar` (`avatar.Avatar`, `avatar.Catalog`, `avatar.State`).
- Estados (`avatar.State`): `avatar.Normal`, `avatar.Greeting`, `avatar.Speak`, `avatar.Think`, `avatar.Listen`, `avatar.Fatigue`, `avatar.Sad`, `avatar.Astonishment`, `avatar.Anger`.
- Constructores y métodos: `avatar.NewCatalog(prefix)`, `cat.WithHair(icon)`, `cat.WithTorso(icon)`, `cat.WithHead(icon)`, `avatar.New(cat)`, `av.SetState(state)`, `av.Speak(text)`, `av.Stop()`, `av.SetLayerVisible(layer, visible)`.
- Sin booleanos ambiguos ni abreviaturas; ningún método requiere leer documentación para adivinar qué hace.

### 3. Complexity ledger
```
Concepts the developer must learn   +2 (avatar.Catalog, avatar.Avatar) / −3 (25 CSS @keyframes, MutationObserver JS, manual DOM/SVG cloning in preview.html)
Files they must touch to do X       +2 (svg.go, web/client.go) / −1 (docs/avatar/preview.html)
Lines at the call site              +6 lines in Go / −260 lines of inline CSS+JS per consumer
Ways to do the same thing           +1 (webtyp.com/avatar) / −1 (ad-hoc HTML+CSS+JS preview runner) = 0 net
```

### 4. Where does it belong
- **`webtyp/svg/sprite`**: extracción de capas/variantes de un `.svg` de Inkscape a `<symbol>` individuales (`AddLayers`), porque `sprite` ya es el único responsable de convertir archivos `.svg` en definiciones de símbolos (`sprite/file.go`).
- **`webtyp/widget/style`**: recetas de micro-movimiento continuo y pivote de partes SVG (`Blink`, `Wave`, `Pulse`, `Float`), porque toda regla CSS de un widget debe venir de una receta de `widget/style` (skill **widget-styling**).
- **`webtyp/avatar`** (nuevo repo creado con `gonew`): el widget `Avatar`, su `Catalog` anatómico, su `css.go` y su motor de secuencias en Go (`Speak`, `Fatigue` con bostezo aleatorio 10–30s, `Think`, parpadeo).
- **`veltylabs/mjosefa-cote`**: el archivo `cote-base-v1.2.svg`, su `svg.go` (`IconSvg()`) y el montaje en `web/client.go`.

### 5. What does this change delete?
- Elimina el motor ad-hoc de CSS (`~185` líneas de `@keyframes` y selectores `[data-state]`) y JS (`~220` líneas de `mount`, `MutationObserver`, `triggerYawn`, `speak`) de `veltylabs/mjosefa-cote/docs/avatar/preview.html`, reemplazándolo por `veltylabs/mjosefa-cote/web/client.go` sobre `webtyp.com/avatar`.

---

## Tabla de librerías y orden de ejecución

Los `docs/PLAN.md` de cada librería se escriben una vez cerradas P1–P4.

| Orden | Librería | Rol / Gate | Cambio principal | Plan |
|---|---|---|---|---|
| 1 | `webtyp/svg` | **Gate 1** | `(*sprite.Sprite).AddLayers(prefix, svgFile)` en `sprite/layers.go`: extrae variantes con `id` de las capas de Inkscape a `<symbol id="<prefix>-<id>">`, limpia `style="display:none"` y metadatos `inkscape:*`/`sodipodi:*`, preserva `transform` y mapea `*-bg` a `fill="var(--color-surface, #ffffff)"`. | **Cerrado** (`v0.3.28`) |
| 2 | `webtyp/widget` | **Gate 1** | Recetas de animación para partes de widget/SVG en `widget/style` (`Wave`, `Pulse`, `Pivot` + `transform-box: fill-box` y reducción automática en `prefers-reduced-motion`). | **Cerrado** (`v0.6.42`) |
| 3 | `webtyp/avatar` | **Gate 2** (creado con `gonew`) | `avatar.Catalog`, `avatar.Avatar` (`dom.Element` embebido por valor), `State` (`Normal`..`Anger`), goroutines de habla/parpadeo/bostezo aleatorio 10–30s (`webtyp/time`), `css.go` (`RenderCSS()`) y tests de comportamiento con `gotest`. | **Cerrado** (`v0.1.0`) |
| 4 | `veltylabs/mjosefa-cote` | **Consumidor** | `svg.go` (`IconSvg()` con `cote-base-v1.2.svg`), constructor `cote.NewAvatar()`, tests unitarios y vista interactiva completa en `web/client.go`. | **Ejecutado localmente** (`LAST_PLAN_EXECUTED.md`) |
