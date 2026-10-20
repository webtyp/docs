# Ayuda integrada en los módulos, avisos por módulo y rediseño de `alert_manager` — master plan
> **Status:** IN PROGRESS · 2026-10-20 · P1–P16 aceptadas (todas); H1 y H4 cerradas, H7 hecha; H2 dividida en H2a–d (H2a en curso); H3/H5/H6/H8 esperan sus gates; `alert_manager` local, sin publicar

Regla que origina la ola (ya en la skill **ui-consistency**): un módulo se explica solo y guía paso a paso; nadie debe tener que explicarlo.

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D1 (P1) | Descripción del módulo como capacidad opcional `Described` (como `Badged`) | H2 |
| D2 (P2) | Botón **«?» pegado a la derecha del título** del módulo, en escritorio y teléfono; su toque no abre el menú | H2 |
| D3 (P3) | La bienvenida vista se recuerda en el navegador de cada persona | H2 |
| D4 (P4) | `view.WithDescription` y `view.WithEmpty`; `crudview` los muestra cuando no hay nada seleccionado | H1, H3 |
| D5 (P5) | Regla: todo campo de significado no obvio lleva `Field.Help` (revisión del plan, sin test obligatorio) | H7 |
| D6 (P6) | Componente genérico de **pasos de configuración** con ✓, dentro de la bienvenida | H2 |
| D7 (P7) | Los tipos de aviso son un **catálogo** que declara cada módulo (código, nombre, ayuda, área) | H4 |
| D8 (P8) | Las **áreas** son un catálogo que declara la aplicación; se eligen de una lista | H4, H6 |
| D9 (P9) | «Quién recibe» se elige entre los **usuarios** del sistema | H6 |
| D10 (P10) | Cada controlador de envío declara los datos que necesita; «Medios de envío» es una sola pantalla | H4, H5, H6 |
| D11 (P11) | **Aviso manual** («Nuevo aviso»: área, asunto, detalle, urgencia), queda registrado quién lo creó | H6 |
| D12 (P12) | El aviso lleva su **origen** (módulo y registro) y ofrece «Ver» | H4, H6 |
| D13 (P13) | Los módulos de stock avisan al cruzar el mínimo y en una revisión diaria de vencimientos | H8 |
| D14 (P14) | La administradora ve la burbuja y un resumen por área, sin detalle | H6 |
| D15 | **`alert_manager` es una sola entrada en el menú, sin submódulos**: su vista es la Bandeja; la configuración (medios, qué se avisa, quién recibe) se abre desde un botón dentro del módulo, como pasos de un acordeón (`components/stepaccordion`), solo para quien tiene el permiso | H6 |
| D16 | Cada aviso tiene estado **por destinatario**: enviado → **visto** (al abrirlo); quien lo creó ve ✓ / ✓✓ como en WhatsApp | H4, H6 |
| D17 | Rediseño de nombres: Bandeja, Medios de envío, Qué se avisa, Quién recibe (por área), Historial de envíos; `lang.json` completo; demo con avisos de al menos dos módulos, uno resuelto y un envío fallido | H6 |
| D18 (cierra P15) | **`alert_manager` guarda todos los avisos** con su módulo de origen y estado «visto» por persona; cada módulo solo llama `notify.Raise`; la aplicación conecta a cada entrada del menú la cuenta de lo no visto por esa persona en ese módulo | H4, H6 |
| D19 (cierra P16) | El **shell** (`platformd`) dibuja una franja de avisos arriba del contenido de cualquier módulo con avisos no vistos, cada uno con «Ver» y «Marcar visto» | H2 |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| H1 | `webtyp/view` | `WithDescription`, `WithEmpty` | **cerrada** v0.6.40 |
| H2a | `webtyp/layout` (`platformd/capabilities.go`) | solo tipos `Described`/`Step`/`AlertedModule`/`ModuleAlert`, sin tocar `platformd.go` | **cerrada** v0.3.57 |
| H2b | `webtyp/layout` (`platformd.go`) | botón «?» (señal + `BindStateFunc`, igual que `menuOpen`) | tras cierre de H2a |
| H2c | `webtyp/layout` (`platformd.go`) | tarjeta de bienvenida con pasos, persistencia primera visita | tras cierre de H2b |
| H2d | `webtyp/layout` (`platformd.go`) | franja de avisos (D19) | tras cierre de H2a (independiente de H2b/H2c) |
| H3 | `webtyp/layout` (`crudview`) | descripción y estado vacío | `[Antigravity · 2026-10-20 15:52]` despachada |
| H4 | `webtyp/notify` | catálogo de tipos, origen del aviso, datos de cada controlador, estado visto | **cerrada** v0.2.2 |
| H5 | `veltylabs/email` | declarar sus datos (remitente, secreto) | espera cierre de H4 |
| H6 | `alert_manager` | rediseño completo (D8–D17) sobre el código local actual; luego v0.1.0 | espera cierre de H2–H4 |
| H7 | `devskills` | regla de `Field.Help` y de descripción de módulo | **cerrada** |
| H8 | `pharmacy_manager`, `supply_manager` | declarar y publicar sus avisos; descripción del módulo | espera cierre de H4 |

## Bloqueos

Ninguno.

## Relación con otras olas

- MONEY_FLOWS: reemplaza la fase M0 en lo que toca a `alert_manager`.
- CATALOG_SPLIT: su fase E (pantallas de stock) y el aviso «falta X» con visto (su D15) dependen de esta ola.
- UI_CONSISTENCY: H1–H3 extienden `view`, `crudview` y `platformd`.
