# Dinero y contabilidad de la clínica — master plan
> **Status:** IN PROGRESS · 2026-10-22 · M1b (`money` v0.2.0), M4b-0, M4b-1, M4b-1c (`payment_manager` v0.3.0), M4c (v0.2.0), M4c-1 (v0.2.1), M4d-1 (`agreement_manager` v0.3.0), `business_calendar` (v0.5.0), `tuu` (v0.1.0) cerrados; pasos 2 y 3 (`transbank` v0.2.0, `payment_manager` v0.4.0) despachados; sin preguntas abiertas

Por qué (histórico): [MONEY_FLOWS_DESIGN.md](MONEY_FLOWS_DESIGN.md) · APIs de TUU: [tuu/docs/ARCHITECTURE.md](../../veltylabs/packages/tuu/docs/ARCHITECTURE.md) · **Diagrama del cobro (hoy y decidido):** [clinical_encounter/docs/diagrams/CHECKOUT_FLOW.md](../../veltylabs/modules/clinical_encounter/docs/diagrams/CHECKOUT_FLOW.md) · Forma de trabajo: skill **master-plan**.

## Preguntas abiertas

Ninguna.

## Decisiones por implementar

| # | Decisión | Dónde |
|---|---|---|
| P9 | La caja es el **turno de una persona** (abre, cobra, cierra con arqueo), no una caja fija | M2, M3 |
| P6 | La boleta de la consulta la emite la **clínica** (TUU), no el profesional; la farmacia emite con su TUU | M5, M9 |
| D2–D4 | Acuerdos (3 cláusulas, insumos con stock), tarifa en el servicio con excepciones, reposición manual + caja 30 días | M3, M4 (hecho) |
| D5 (K1) | TUU tiene API HTTP en la nube (Payment Intent, Pago Remoto); ni Bluetooth ni red local | M4b |
| D6 (P12, P14) | El profesional elige **«Enviar a caja»** (cierra la atención, no espera al paciente) o **QR en línea** (pago inmediato, no pasa por caja). Una solicitud de pago = la consulta con sus n servicios | M9 (`clinical_encounter`) |
| D7 (P12, P18) | La cajera con **turno abierto** recibe el aviso; lista de pendientes (paciente, profesional, hora, monto); selecciona, elige medio y aprieta Cobrar; con tarjeta el monto **viaja solo a la máquina**. Sin turno abierto, el profesional lo ve al enviar | M9 (`cash_manager`) |
| D8 (P12) | El profesional ve el estado de pago de cada consulta en su pantalla | M9 (`clinical_encounter`) |
| D9 (P15, P20) | Procesador elegido en BD como `notify`: adaptadores compilados, fila activa **por alcance** (clínica/farmacia, cuentas separadas) con nombre del secreto; cada pago guarda su procesador | M4b (`payment_manager`) |
| D10 (P16, K1) | Dos puertos genéricos: `Gateway` en línea (`Start` → URL, `Confirm(aviso)`, `Status`; Webpay mete su `Commit` en `Confirm`) y `Terminal` (enviar monto a una máquina por id, consultar estado) | M4b (`payment_manager`, `transbank`, `tuu`) |
| D11 (P19) | El profesional **no** cambia el monto; la boleta de honorarios al paciente no se registra (asunto del profesional; derivado de tu respuesta, que solo trató el pago a la clínica) | M9 |
| D13 (P23) | **Un solo documento por venta.** Con tarjeta (en caja o en línea) el comprobante del pago **es** la boleta (art. 54 Ley IVA, Res. SII 176/2020, vigente desde 2021, «medios de pago electrónicos» sin distinguir canal); solo el **efectivo** necesita boleta (D18). Requiere que la clínica tenga declarado en el SII el modelo «No emito boleta cuando recibo un pago electrónico» — el contador lo verifica | M4b, M9 |
| D14 (P24) | Servidor **local** + **Cloudflare Tunnel** que publica solo las rutas públicas (avisos de pago, reservas, página de pago) + respaldo de internet 4G; un solo servicio | SELFHOST_DEPLOY |
| D16 (P26) | **Cuenta por atención con saldo**: varios pagos hasta saldo cero, cada uno cobra lo que falta (`Due`; el monto sigue sin venir del cliente) | M4b-1 (`payment_manager`; hecho); falta `ChargeReader` de la atención (M9/M10) |
| D18 (P21) | **Efectivo:** la cajera recibe el dinero, digita esa venta en efectivo en la TUU (que imprime la boleta) y confirma en el sistema; es la única digitación que queda. No se construye emisor por API en esta ola (sin segundo proveedor ni puerto `Issuer`) | M9 (`cash_manager`) |
| D19 (P28) | **Tratamiento de IVA por máquina, sin cobros mixtos**: el consultorio cobra solo prestaciones exentas y la farmacia solo productos afectos; consultorio **exenta**, farmacia **afecta** (IVA 19 %). Es configuración de cada máquina en el adaptador (`tuu`: tipo de documento y monto exento por número de serie), no del puerto `Terminal`, que sigue sin saber de impuestos | M4b-2 (`tuu`), M5 (la venta de farmacia separa neto e IVA para el contador) |
| D20 | **`Gateway` se generaliza sí o sí y se construyen los dos adaptadores en línea** (`transbank` y `tuu`); el adaptador verifica lo que manda su procesador (Webpay: commit; TUU: firma HMAC) y entrega a `payment_manager` un resultado ya verificado (`OnlineResult`). Cuál está activo es configuración (D9) | M4b-1c, M4b-2, M4b-3 |
| D21 | Con tarjeta el dinero llega **después y menos**: plazo de abono y comisión según el plan de cada cuenta de procesador | D22, D23 |
| D24 (P22) | **TUU es el procesador en línea activo** (QR de la consulta y abono al reservar); el adaptador `transbank` se adapta igual (D20) y queda disponible. Pendiente con TUU, no bloquea: comisión del Payment Intent y cómo se hace una devolución parcial por API; si TUU no devuelve por API, se abre aquí una pregunta sobre cómo devolver el abono | M4b-2 (`tuu` primero), D9 |
| D23 (P30) | **Se registra todo.** Efectivo y transferencia → `Charged` con `Means`. Tarjeta → `Accrued` de entrada (por cobrar al procesador, `DueAt` = día hábil del depósito, `Means: CARD`) + `Accrued` de salida por comisión e IVA (`Party` = procesador). La «Caja 30 días» pone el neto el día del depósito; el cierre de turno cuenta solo `Charged`/`Refunded` con `Means: CASH` y `Actor` = la cajera del turno | M1b, M4b-5, M3b |
| D22 (P29) | **Abono y comisión = adaptador configurable detrás de un puerto común** de `payment_manager`, como `Gateway`/`Terminal`: cada procesador lo implementa con su propio modelo (TUU: abono al instante/1/2/5/10 días hábiles, comisión porcentual 1,49 % o mixta 0,79 % + $65, más IVA; Transbank: otro, p. ej. % por crédito/débito con mínimo en UF). El adaptador dice **neto, comisión, IVA de la comisión y días hábiles**; `payment_manager` convierte días hábiles en fecha con los feriados de `business_calendar` (el adaptador no sabe de calendario). El plan de cada cuenta es configuración (fila de D9) | M4b-5 |
| D17 (P27) | El abono obligatorio (porcentaje o monto) y la regla de devolución por anticipación viven en el servicio | M10 (`service_catalog`) |
| K3 | WhatsApp fuera de la primera etapa | hecho (`notify` v0.2.1) |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| M0 | `notify`, `email`, `whatsapp` | canales configurables en BD | cerrada (`notify` v0.2.1) |
| M0b | `alert_manager` | quién recibe qué alerta | cerrada v0.1.0 |
| M1 | `webtyp/money` | hechos, eventos, idempotencia | cerrada v0.1.3 |
| M1b | `webtyp/money` | `Fact.Means` (CASH/CARD/TRANSFER, obligatorio en Charged/Refunded) y `Fact.Actor` | cerrada v0.2.0 |
| M2 | `payment_manager` | comprobante, hechos | cerrada v0.1.21 |
| M3 | `cash_manager` | turnos, arqueo, `Sink` | cerrada v0.1.0 |
| M4 | `agreement_manager` | acuerdos, excepciones, estado de cuenta | cerrada v0.1.13 |
| M4b-1 | `payment_manager` | puerto `Terminal` (D10, mitad máquina), saldo por referencia `Due` (D16), procesador guardado en cada pago, `ReconcilePayments` | cerrada v0.2.0 |
| M4b-1c | `payment_manager` | D20: `Gateway` sin `Commit`, `OnlineResult`, `Payer`, `MethodCard`, sin `Deps.ReturnURL` | cerrada v0.3.0 |
| M4b-0 | `webtyp/dom` | `Navigate(url)` por GET (TUU no usa POST) | cerrada v0.13.27 |
| M4b-1b | `webtyp/driver` (nuevo; búsqueda: `notify/driver.go` tiene la única copia de `Config`/`Setting`/`Field`/`SecretResolver`/`Driver`) + `notify` + `payment_manager` | D9 (procesador elegido en BD por alcance) sin duplicar el patrón de `notify`: extraerlo a una librería que ambos usen | crear el repo con `gonew` (lo corre el usuario) |
| M4b-2 | `veltylabs/tuu` | `Terminal` por Pago Remoto v2 (IVA por máquina D19) | cerrada v0.1.0 |
| M4b-2b | `veltylabs/tuu` (Payment Intent) | `Gateway` en línea de TUU (D20, D24) | **sin plan**: la respuesta de creación del Payment Intent no está documentada (ver Bloqueos) |
| M4b-3 | `transbank` `docs/PLAN.md` (v0.2.0) | adaptar a D20: `Provider()`, commit dentro de su ruta de retorno, `OnlineResult`, `Config.ReturnURL` | [Antigravity · 2026-10-22] despachado |
| M4b-5 | `business_calendar` (v0.5.0) → `payment_manager` `docs/PLAN.md` (v0.4.0) → `tuu` `docs/PLAN_PAYOUT.md` (v0.2.0) | `BankDayAfter`; puerto `PayoutTerms` + `BankDays`, hechos con `Means`/`Actor`, tarjeta = `Accrued` con `DueAt` y comisión; plan de abono de TUU configurable | `business_calendar` cerrada v0.5.0; `payment_manager` despachado (paso 3); `tuu` espera paso 4 |
| M3b | `cash_manager` `docs/PLAN.md` (v0.2.0) | cierre de turno solo efectivo de la cajera (`Means`/`Actor`); guarda medio y autor; id de hecho acepta `_` | [Antigravity · 2026-10-22] despachado |
| M4c | `agreement_manager` | D15 + **defecto encontrado**: los hechos de dinero salían por el tema `"money"` (no `money.fact`) e incompletos → `cash_manager` nunca los recibía; porcentaje con `float` | cerrada v0.2.0 |
| M4c-1 | `agreement_manager` | traducción al español de documentación y comentarios de código | cerrada v0.2.1 |
| M4d-1 | `agreement_manager` | diez operaciones (terceros, acuerdos, devengos, saldar, saldar con boleta, estado de cuenta) y las vistas apuntando a ellas | cerrada v0.3.0 |
| M4d-2 | `agreement_manager` | pantallas (diseño con el usuario: textos, flujo de saldar con boleta) | sin plan: necesita preguntas de pantalla |
| M9 | `clinical_encounter`, `cash_manager`, `payment_manager` (`ChargeSource` de la atención), QR con `webtyp/qr` | cobro desde la atención, sin papel (D6–D8, D11) | espera M4b-1c y M4b-2/3 |
| M10 | `service_catalog`, `appointment_booking` | abono al reservar (D17) y su devolución | espera M4b-1c |
| M5 | venta de farmacia | venta, cobro por `Terminal`, boleta, stock | espera M4b, CATALOG_SPLIT (F) |
| M6 | `purchase_manager` (nuevo) | compras con costo de lote | después de M3/M4 |
| M7 | contable de `veltylabs` | partida doble | después de M1-M6 |
| M8 | `mjosefa-cms` | menú y permisos por alcance | después de todo |

## Bloqueos

- 2026-10-22 · **Despacho**: `payment_manager` v0.3.0, `agreement_manager` v0.3.0, `business_calendar` v0.5.0, `tuu` v0.1.0 cerrados. **Paso 2**: `transbank` v0.2.0 despachado (sesión 15715883845839842515); `cash_manager` v0.2.0 en curso. **Paso 3**: `payment_manager` payout v0.4.0 despachado (sesión 12361960052788271007). **Paso 4** (con `tuu` v0.1.0 y `payment_manager` v0.4.0): `tuu` PLAN_PAYOUT (espera paso 3).
- 2026-10-22 · Payment Intent de TUU: la documentación no dice qué devuelve la creación (¿URL de pago?, ¿campo?). Hace falta una llamada al ambiente de integración (credenciales públicas de prueba en developers.tuu.cl) o preguntarlo a TUU; sin eso no se escribe M4b-2b.
- 2026-10-22 · `webtyp/driver` (M4b-1b): el repo se crea con `gonew driver "<descripción>"`; lo corre el usuario.
- 2026-10-22 · Preguntar a TUU comisión del Payment Intent y devolución parcial por API (D24, no bloquea). El contador verifica el modelo de emisión declarado (D13).
- `alert_manager`: UX rechazada; depende de MODULE_GUIDANCE (P15/P16), fuera de esta ola.

- 2026-10-22 · **Defecto en `cash_manager` (M3)**: el cierre de turno suma como efectivo esperado **todos** los cobros del tenant desde la apertura, también los con tarjeta y los de otros cajeros (`cash_manager/ops.go:146`), porque `money.Fact` no dice con qué medio se pagó. Se corrige en M1b → M4b-5 → M3b.

## Relación con otras olas

- Misma ola que CATALOG_SPLIT (F espera la caja); M10 toca `service_catalog`.
- D14 fija la forma del servidor para SELFHOST_DEPLOY (túnel, servidor local, 4G).
