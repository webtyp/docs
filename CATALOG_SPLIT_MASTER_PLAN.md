# Catálogo: servicios, farmacia e insumos — master plan
> **Status:** IN PROGRESS · 2026-10-20 · A, B1, B2, C0, C1, C2 cerradas; **C3 espera P22**; E espera MODULE_GUIDANCE; F espera la caja de MONEY_FLOWS

Razonamiento de las decisiones ya implementadas: [CATALOG_DESIGN.md](CATALOG_DESIGN.md).

## Preguntas abiertas

**P22 — Si al marcar la atención como realizada la rebaja falla** (el profesional no tiene turno en un box a esa hora, el box no tiene bodega de insumos, falta stock o el lote venció). A) la atención **se cierra igual**; cada línea que no se pudo rebajar queda «pendiente de rebaja» y genera un aviso al área Insumos, que la resuelve desde su pantalla; B) la atención **no se puede cerrar** hasta resolverlo. **Recomiendo A**: la ficha clínica no puede quedar bloqueada por un problema de bodega, y el aviso hace visible el problema (regla «nunca en silencio»). Respuesta:

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| D5 | El uso de insumos se registra en la atención, con instantánea, y se rebaja **una sola vez** | C |
| D6 | Cada línea de uso tiene la bandera `billable`, por omisión no cobrable | C |
| D14 (P15) | Cada servicio tiene una **receta de insumos**; al usarse en una atención el stock se rebaja **solo**, con una llamada directa a `supply_manager` (un error de stock se ve al guardar) | C |
| D15 (P15) | El profesional ve en la atención los insumos de esa consulta y, con un par de clics, avisa «falta X» al área Insumos; sabe si el aviso **fue visto** (como el doble check de WhatsApp) | aviso manual y «visto» de MODULE_GUIDANCE (H6) |
| D16 (P18) | `clinical_encounter` incorpora `tenant_id` en todas sus tablas y operaciones | C |
| D18 (P16) | La bodega sale de `room_layout`: el box donde el profesional tiene turno a la hora de la atención; cada ubicación de insumos puede indicar el box que abastece. **Ningún módulo importa a otro**: `clinical_encounter` declara interfaces estrechas y la aplicación le pasa las instancias de `room_layout` y `supply_manager` | C0, C2, C3 |
| D19 (P17) | La receta se rebaja **al marcar la atención como realizada**; una atención cancelada no toca el stock | C3 |
| D20 (P20) | La receta vive en `service_catalog` y se arma en la pantalla del servicio; los insumos se eligen mediante una interfaz que la aplicación conecta a `supply_manager` | C1 |
| D21 (P21) | Antes de cerrar, el profesional ve la lista cargada desde la receta y puede cambiar cantidades o agregar insumos | C3 |
| D22 (P13) | La venta al público es un **módulo aparte** (líneas, precio congelado, cobro con `payment_manager`, salida `SALE`) | F |
| D17 (P19) | El profesional registra y ajusta el uso (permiso `encounter_supply_use`); la encargada de insumos ve el resultado en los movimientos de `supply_manager` | C |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| L | `webtyp/stock` | reglas puras de stock y suite `stocktest` | cerrada v0.1.1 |
| A | `service_catalog` | solo servicios y especialidades (renombre de `item_catalog`) | cerrada v0.4.0 |
| B1 | `pharmacy_manager` | medicamentos, lotes con vencimiento, ubicaciones, movimientos | cerrada v0.1.0 |
| B2 | `supply_manager` | insumos, lote opcional, ubicaciones, movimientos | cerrada v0.1.1 |
| C0 | `room_layout` | lector `RoomOfOccupant`: en qué box tiene turno un profesional en un instante | cerrada v0.3.37 |
| C1 | `service_catalog` | receta de insumos (`service_supply`, lector `ServiceRecipe`); `mcp.go` separado en `module.go`, `errors.go`, `ops.go` | cerrada v0.5.0; **su pantalla** (elegir insumos) va con E |
| C2 | `supply_manager` | `room_id` en la ubicación (una activa por box) y `UseInRoom` | cerrada v0.2.0; **elegir el box** en el formulario va con E |
| C3 | `clinical_encounter` | `tenant_id`, líneas de uso desde la receta, ajuste, rebaja al realizar | espera P22 y C0–C2 |
| E | `pharmacy_manager`, `supply_manager`, `service_catalog` | pantallas de stock, entradas, salidas y próximos a vencer; elegir el box de una ubicación; armar la receta del servicio | espera MODULE_GUIDANCE |
| F | módulo de venta (nuevo) | venta al público | después de la caja (MONEY_FLOWS M2–M4) y de K1 |
| D | `mjosefa-cms` | grupos «Farmacia» e «Insumos» en el menú; permisos por módulo | pendiente, después de E |
| — | `veltylabs/modules/*/AGENTS.md` | la línea «implementación de referencia: `item_catalog`» pasa a `service_catalog` al publicar cada módulo | pendiente menor |

## Bloqueos

Ninguno: solo faltan respuestas.

## Relación con otras olas

- MONEY_FLOWS: costo por lote y `tax_treatment` ya implementados; la venta (F) depende de su caja (M2–M4) y de K1.
- MODULE_GUIDANCE: la fase E y el aviso «falta X» con visto (D15) se construyen sobre su ayuda integrada y sus avisos.
