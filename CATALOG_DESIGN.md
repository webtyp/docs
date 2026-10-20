# Catálogo: servicios, farmacia e insumos — diseño

Registro permanente de por qué el antiguo `item_catalog` se separó en `service_catalog`, `pharmacy_manager` y `supply_manager` sobre la librería `webtyp/stock` (2026-10-19/20). El estado de trabajo vive en otro lado; aquí solo queda el razonamiento.

## Objetivo y alcance

Hoy `item_catalog` mezcla en una sola tabla (`catalog_item`) lo que la clínica **presta** (servicios) y lo que **tiene**
(productos), con un radio «Type» en el formulario. En pantalla se presta a confusión. Se pide:

- que **servicios**, **productos** y **especialidades** sean cada uno su propia entrada del menú «Catálogo»;
- decidir si la base de datos separa las tablas o mantiene una sola de ítems;
- tener en cuenta que `mjosefa-cms` es una **clínica que no vende productos, pero tiene una farmacia**, y que existen **insumos** que se usan en la atención.

Dentro de alcance: `item_catalog`, un módulo nuevo para productos e insumos si P1 lo decide, y el registro de uso de insumos en `clinical_encounter`. Fuera: precios, facturación y contabilidad (`payment_manager`), compras a proveedores.

## Lo que dice el código (hechos, no opiniones)

| Hecho | Dónde |
|---|---|
| Una sola tabla `catalog_item` con `type` = `S` o `P`; el formulario pide el tipo con un radio | `item_catalog/model.go:11-12, 52-64` |
| Los **únicos** lectores de otros módulos son `ServiceExists` y `ServiceCard`, y ambos **filtran `type == S`**: nadie lee un producto | `item_catalog/mcp.go:435-461`; consumidor: `appointment_booking/booking.go:25,72,79` |
| **Cero** ítems y cero especialidades en producción; la importación histórica crea **solo servicios** (precio 0) | `mjosefa-cms/docs/MASTER.md` B3; plan de importación de datos de `mjosefa-cms` |
| El `sku` de un servicio lleva el prefijo de su especialidad (`md0001`) | `item_catalog/migration.go`, `model.go` |
| `catalog_agreement` (aseguradora, código, precio) cuelga de un ítem: son **convenios**, solo tienen sentido en servicios | `item_catalog/model.go` (`AgreementModel`) |
| La especialidad la usa también `room_layout` como «área» de un box | `mjosefa-cms/docs/MASTER.md` RL-3 |
| Una reserva **congela** nombre, precio y moneda del servicio al crearse (`PriceSnapshot`…) | `appointment_booking/AGENTS.md` («Instantáneas») |
| `provider_payouts` ya nombra el catálogo de servicios como `service_catalog.ServiceItem` | `provider_payouts/model.go:38` |
| La **base de datos anterior** de la clínica ya había resuelto esto: `item` con `item_type`, `for_sale`, `unit` y `id_product_type` (`PHARMACY`, `CLINICAL_SUPPLY`, opcional); inventario con ubicación, lote y vencimiento, movimientos con `INTERNAL_USE`; las revisiones concluyen que el insumo se descuenta **al usarse en la atención, no al facturar**, y avisan del riesgo de doble descuento y de «fuga de ingresos» | `client-monjitaschillan-platform/docs/diagrams/database/db-er-v9.mmd`, `db-er-v8-bugs.md`, `db-er-v9-bugs.md`, `db-er-v6-bugs3.md` |

## Qué hace la industria

| Referente | Cómo lo resuelve | Lectura para nosotros |
|---|---|---|
| **Odoo 19** | Un solo modelo de producto: tipo *Goods / Service / Combo*, y «Track Inventory» (por cantidad, lote o serie) como interruptor aparte ([Cybrosys](https://www.cybrosys.com/odoo/odoo-books/v19/inventory/products-menu/), [foro Odoo](https://www.odoo.com/es/forum/ayuda-1/product-types-261635)) | ERP de venta y compra: un maestro con banderas |
| **ERPNext** | Un solo *Item*; los servicios llevan «Maintain Stock» apagado: sin bodega ni libro de stock ([docs](https://docs.frappe.io/erpnext/erpnext-for-services-organization)) | Igual: un maestro con bandera |
| **HL7 FHIR** (estándar clínico) | Separa la **definición del servicio** (`ActivityDefinition`), su **precio** (`ChargeItemDefinition`) y el catálogo como mera agrupación («Catalog is just a grouping/publication mechanism», [FHIR chat](https://chat-archive.fhir.org/stream/179166-implementers/topic/Update.20product.20prices.20(catalog.3F).html), [Order Catalog IG](https://hl7.org/fhir/uv/order-catalog/StructureDefinition-LabChargeItemDefinition-definitions.html)); los productos (medicamento, dispositivo, insumo) son recursos distintos del servicio | Los sistemas clínicos separan servicio de producto |
| **Chile (farmacia)** | Estupefacientes y psicotrópicos exigen libro de control visado, con ingresos y egresos por producto (DS 404 y DS 405 art. 18); el reglamento de farmacias y botiquines es el DS 466 ([DS 405](https://iura.cl/dto-405-minsal/18), [DS 404](https://iura.cl/dto-404-minsal/22)) | La farmacia es un dominio **regulado**; no se mezcla con la agenda |

La investigación **no** confirmó las reglas de lote y vencimiento ni lo que distingue un botiquín de una farmacia en una clínica: hay que verificarlo con el químico farmacéutico responsable y con el texto vigente del DS 466 antes de implementar (**P3**, **P10**). Los nombres de los recursos de producto de FHIR (`Medication`, `Device`, `SupplyDelivery`) vienen de conocimiento general, no de esta búsqueda.

**Lectura de conjunto:** los ERP (vender y comprar cosas) usan un maestro único con banderas; los sistemas clínicos (agendar y registrar actos) separan el servicio del producto. Nosotros somos lo segundo con un poco de lo primero (la farmacia).

## Opciones de modelo y recomendación

| | A. Una tabla con `type` (hoy) | B. Dos tablas en el mismo módulo | C. Núcleo común + extensión por tipo | D. Dos módulos: servicios+especialidades / productos+insumos+stock |
|---|---|---|---|---|
| Tipado (cero valores deny) | campos de servicio quedan vacíos en productos y al revés | cada tabla solo trae lo suyo | núcleo tipado, extensiones nulas | cada una solo lo suyo |
| Invariantes distintas (especialidad obligatoria; lote y vencimiento) | un `if type` en cada regla | separadas | separadas, con un JOIN | separadas |
| Permisos (RBAC es por **recurso y acción** de cada operación, `Requires(resource, action)`; el módulo no es la unidad) | un solo recurso o el recurso depende del `type`: hay que partir las operaciones | se parte por recurso | se parte por recurso | se parte por recurso (y el módulo no lo impide ni lo exige) |
| Referencia desde otros módulos | una sola FK | dos (tipo + id) | una FK al núcleo | id de texto por módulo + **snapshot** (la regla del ecosistema; la reserva ya congela precio y nombre) |
| Costo de migrar hoy | — | bajo | medio | bajo: **0 productos en producción** |
| Evolución (lotes, proveedores, stock) | ensancha la misma tabla | ensancha `product` | ensancha la extensión | crece en su propio módulo |

**Recomendación: D**, con una sola entidad para productos **e insumos** (P4). Razones, en orden de peso:

1. **El único vínculo que hoy une servicio y producto — «precio de lista y moneda para cobrar» — ya se resuelve con una instantánea** en la línea de cobro, como hace la reserva. No necesita una FK única; es lo que hace innecesario el maestro común.
2. **Permisos — corrección (2026-10-19):** dije que el ecosistema autoriza por módulo y es impreciso: cada operación declara su propio `Requires(resource, action)` y los roles reciben recursos y acciones, no módulos (`router/README.md`, `docs/INTROSPECTION.md`). Separar módulos **no** es lo que da la separación de permisos; la da **partir los recursos**. Este argumento queda fuera; la decisión D sigue en pie por los demás (3, 4 y 5) y por la evolución distinta de cada dominio.
3. **Invariantes y regulación distintos**: servicio = especialidad, convenios, duración (vive en la configuración del profesional); producto = unidad, lote, vencimiento, ubicación, costo, proveedor y, en medicamentos, control sanitario.
4. **Cero datos que migrar**: hoy hay 0 productos; separar ahora cuesta nada y después cuesta una migración.
5. La propia clínica ya llegó a esto: su base anterior ni siquiera usaba tablas `service_item`/`product_item` vacías (v9-bugs §B).

**Qué no recomiendo:** A (es lo que genera la confusión), y C (la complejidad de un núcleo con extensiones se justifica cuando el mismo registro se **vende** en ambos mundos; aquí la clínica no vende productos).

**Respuesta directa a «¿tabla separada o una de ítem?»:** separar servicio de producto. Lo que **sí** se mantiene en una sola tabla es *producto + insumo*: son el mismo objeto físico con `kind` (`PHARMACY`, `SUPPLY`) y la bandera `for_sale`; así lo hacen Odoo y ERPNext, y así lo tenía la base anterior.

## Decisiones cerradas (2026-10-19)

Respuestas del usuario: la farmacia es **autorizada** y **vende al público**; hoy **no tiene sistema**, solo un Excel; «acepto tus recomendaciones».

| # | Decisión | Consecuencia |
|---|---|---|
| D1 (P1) | Los productos viven en un **módulo nuevo**; `item_catalog` queda con servicios y especialidades | repo `veltylabs/stock_manager` (el nombre sigue la convención `<sustantivo>_manager`; en el borrador se llamaba `stock_manager`) |
| D2 (P2) | **Renombrar** `item_catalog` → `service_catalog` (módulo, paquete y tablas `service`, `service_agreement`, `specialty`) | rompe la ruta de importación en `appointment_booking`, `clinical_encounter`, `staff_manager` (semillas y pruebas) y `mjosefa-cms`; el repo se renombra en GitHub por el usuario antes de la fase A |
| D3 (P3) | Stock **con lote y vencimiento** (por ser farmacia autorizada; ajuste de mi recomendación inicial) | todo el stock se lleva por lote; vencimiento obligatorio en medicamentos; salida por primero en vencer |
| D4 (P4) | Productos e insumos = **una entidad** `product` con `kind` `PHARMACY`/`SUPPLY` y `for_sale` | **reemplazada por D13**: dos módulos, cada uno con su entidad |
| D5 (P5) | El uso de insumos se registra en la **atención** (`clinical_encounter`) con instantánea; descuenta una sola vez | fase C |
| D6 (P6) | Bandera `billable` **por línea de uso**, por omisión no cobrable | fase C |
| D7 (P7) | Las especialidades solo clasifican servicios | |
| D8 (P8) | Servicios mantienen `sku`; productos: `sku` + código de barras y N.º de registro sanitario opcionales | |
| D9 (P9) | Convenios solo para servicios | |
| D10 (P10) | `controlled` es solo una **bandera** sin lógica; el libro regulado de controlados sigue fuera del software | **riesgo de cumplimiento: confirmarlo con el químico farmacéutico responsable**; si la farmacia vende controlados, el libro es obligatorio (DS 404/405) |
| D11 (P11) | Menú «Catálogo» con Servicios y Especialidades hoy; la farmacia entra con su propio grupo «Farmacia e insumos» (Productos, Ubicaciones) | el grupo lo define `stock_manager`; `mjosefa-cms` lo agrega al menú; **reemplazada por D13**: «Farmacia» (`pharmacy_manager`) e «Insumos» (`supply_manager`) son grupos de módulos distintos |
| D12 (P12) | Se borra `type`, `ItemTypeService/Product`, el filtro `type` y el radio; sin datos que migrar | fase A |

**D13 (cierra P14, 2026-10-19) — farmacia e insumos son dos módulos con un núcleo común.** Decisión del usuario: «B con un motor común (código)»; «la farmacia ya existe, dejar insumos y farmacia juntos es un error». Consecuencias:

- **Dos módulos con tablas, recursos de RBAC, reglas y menú propios:** `pharmacy_manager` (medicamentos: vencimiento obligatorio, precio y venta, `controlled`, código de barras, registro sanitario; el grupo de menú «Farmacia») y `supply_manager` (insumos: sin precio ni venta, vencimiento opcional, uso en la atención; el grupo «Insumos»). Cada uno tiene sus ubicaciones: no se transfiere de uno a otro.
- **Lo común es una librería, `webtyp.com/stock`**, porque la regla del ecosistema impide que un módulo importe a otro y que la misma lógica viva en dos repos (si un módulo necesita algo que no existe, se arregla aguas arriba). Contiene la **lógica pura** (sin acceso a datos): tipos de movimiento y su dirección, salida por primer vencimiento sobre una lista de lotes, predicados de vencido y por vencer, aplicación de un movimiento sin dejar el stock negativo; y una **suite de conformidad** (`stock/stocktest`) que cada módulo ejecuta contra su propia implementación, para que ambos cumplan la misma conducta.
- **Límite honesto de «motor común»:** `ormc` genera las tablas desde literales estáticos de `model.Definition` dentro de cada módulo, y dos módulos no pueden compartir los mismos nombres de tabla. Por eso las tablas, el repositorio, las transacciones y las operaciones se escriben en cada módulo (con la misma forma); lo que no se duplica son las reglas ni su verificación. Esto es consecuencia de cómo funciona `ormc`, no una elección.
- **El repo `veltylabs/stock_manager`, creado antes, queda sin uso con ese nombre.** Recomiendo **renombrarlo en GitHub a `pharmacy_manager`** (casi vacío; conserva su historia y su `AGENTS.md`) y crear `supply_manager` y `webtyp/stock` nuevos. Lo decide el usuario.
- Orden: (1) `webtyp/stock`, se publica con su tag; (2) los planes de `pharmacy_manager` y `supply_manager` se escriben con ese tag exacto y se despachan en paralelo; (3) siguen el plan de uso de insumos en la atención (`clinical_encounter` ← `supply_manager`) y la venta (P13) sobre `pharmacy_manager`.

**Detalles de diseño fijados al escribir los planes (2026-10-19; sin pregunta abierta, corregibles si no te cuadran):**

- **Un módulo, una entidad con su nombre:** `pharmacy_product` y `supply_item`; tablas, recursos de RBAC y operaciones con el prefijo de su módulo (`pharmacy_*`, `supply_*`), porque `ormc` no permite dos módulos con las mismas tablas. Cada recurso es la tabla (`pharmacy_product`, `pharmacy_location`, `pharmacy_batch`, `pharmacy_stock_level`, `pharmacy_movement`): un rol puede recibir solo uno.
- **Reglas de stock solo en `webtyp/stock`:** primer vencimiento (`Allocate`), tipos de movimiento, vencido; los módulos no escriben ninguna. Cada módulo ejecuta `stocktest.Run` sobre su propia implementación.
- **Farmacia:** vencimiento **obligatorio** en el lote; `tax_treatment` obligatorio (`TAXED` / `EXEMPT`; la tasa no se guarda: es de la ley y va en la contabilidad); precio en **pesos enteros con IVA incluido**; no se vende un producto sin la bandera `for_sale` (`ErrNotForSale`); `controlled` es solo bandera. **Insumos:** sin precio, sin venta; lote opcional (`NO-BATCH`); la única salida de consumo es `INTERNAL_USE`.
- **Costo:** el lote lleva `unit_cost` (pesos enteros por unidad, 0 = desconocido), fijado al recibir; un costo distinto en un lote existente es `ErrBatchCostMismatch`.
- **Hechos de dinero:** con `Deps.MoneyScope` no vacío y `Deps.Publisher`, cada **salida** (`SALE`, `INTERNAL_USE`, `ADJUST_OUT`) con costo publica un `money.Consumed` (costo × cantidad, redondeo hacia afuera del cero). **Las compras no son un hecho de estos módulos** (`purchase_manager` las posee, M6 de dinero): evita contarlas dos veces. Una falla al publicar se **devuelve** después del commit, nunca se traga.
- **Alertas:** cada producto tiene `min_quantity` (0 = sin aviso); `list_low_stock` y `list_expiring` son lecturas que usará `alert_manager`; los módulos no levantan alertas.
- **Sin pantalla de stock todavía:** en esta versión solo hay «Productos/Artículos» y «Ubicaciones»; las pantallas de entradas, salidas y vencimientos son la fase E.
- **Servicios:** `service_catalog` conserva `price` (decimal) y `currency` porque la reserva congela ese precio como decimal; pasar los precios a pesos enteros es otra ola. Agrega `tax_treatment` opcional (para negocios que sí emiten documento por el servicio).

