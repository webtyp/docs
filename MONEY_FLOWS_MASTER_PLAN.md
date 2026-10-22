# Dinero y contabilidad de la clínica — master plan
> **Status:** IN PROGRESS · 2026-10-22 · P21, P23–P28 consolidadas (D13–D19); **P22 espera al usuario**; M4b-1 y M4c despachados

Por qué (histórico): [MONEY_FLOWS_DESIGN.md](MONEY_FLOWS_DESIGN.md) · APIs de TUU: [tuu/docs/ARCHITECTURE.md](../../veltylabs/packages/tuu/docs/ARCHITECTURE.md) · **Diagrama del cobro (hoy y decidido):** [clinical_encounter/docs/diagrams/CHECKOUT_FLOW.md](../../veltylabs/modules/clinical_encounter/docs/diagrams/CHECKOUT_FLOW.md) · Forma de trabajo: skill **master-plan**.

## Preguntas abiertas

**P22 — Pago en línea (QR y abono al reservar): ¿con qué procesador?** (sin respuesta). Comisiones 2026 publicadas (sin IVA): Webpay Plus débito 1,49–1,75 %, crédito 2,29–2,95 %; Mercado Pago 2,89–3,99 %; Flow 2,89–3,19 %; Khipu (solo transferencia) 0,69 %; TUU en línea no publica tarifa. A) **Webpay Plus**: más barato con tarjeta y el adaptador `transbank` **ya existe**. B) TUU Payment Intent: un solo proveedor, tarifa desconocida (pregúntala a TUU). C) A + Khipu después para transferencias. **Recomiendo A**, salvo que TUU ofrezca menos que el débito de Webpay. Ojo: en Webpay la devolución parcial es solo para crédito; en débito se anula completa. No bloquea a `payment_manager` (el puerto es el mismo, D10); sí bloquea el adaptador y el QR. Respuesta:

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
| D15 (P25) | El estado de cuenta registra la boleta de honorarios del profesional: número, bruto, retención (tasa por año, configurable; 15,25 % en 2026) y líquido a transferir | M4c (`agreement_manager`) |
| D16 (P26) | **Cuenta por atención con saldo**: varios pagos hasta saldo cero, cada uno cobra lo que falta (`Due`; el monto sigue sin venir del cliente) | M4b-1 (`payment_manager`); que el abono de la reserva se descuente de la atención lo hace el `ChargeReader` de la atención (M9/M10) |
| D18 (P21) | **Efectivo:** la cajera recibe el dinero, digita esa venta en efectivo en la TUU (que imprime la boleta) y confirma en el sistema; es la única digitación que queda. No se construye emisor por API en esta ola (sin segundo proveedor ni puerto `Issuer`) | M9 (`cash_manager`) |
| D19 (P28) | **Tratamiento de IVA por máquina, sin cobros mixtos**: el consultorio cobra solo prestaciones exentas y la farmacia solo productos afectos; consultorio **exenta**, farmacia **afecta** (IVA 19 %). Es configuración de cada máquina en el adaptador (`tuu`: tipo de documento y monto exento por número de serie), no del puerto `Terminal`, que sigue sin saber de impuestos | M4b-2 (`tuu`), M5 (la venta de farmacia separa neto e IVA para el contador) |
| D17 (P27) | El abono obligatorio (porcentaje o monto) y la regla de devolución por anticipación viven en el servicio | M10 (`service_catalog`) |
| K3 | WhatsApp fuera de la primera etapa | hecho (`notify` v0.2.1) |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| M0 | `notify`, `email`, `whatsapp` | canales configurables en BD | cerrada (`notify` v0.2.1) |
| M0b | `alert_manager` | quién recibe qué alerta | local, espera MODULE_GUIDANCE P15/P16 |
| M1 | `webtyp/money` | hechos, eventos, idempotencia | cerrada v0.1.3 |
| M2 | `payment_manager` | comprobante, hechos | cerrada v0.1.21 |
| M3 | `cash_manager` | turnos, arqueo, `Sink` | cerrada v0.1.0 |
| M4 | `agreement_manager` | acuerdos, excepciones, estado de cuenta | cerrada v0.1.13 |
| M4b-1 | `payment_manager` | puerto `Terminal` (D10, mitad máquina), saldo por referencia `Due` (D16), procesador guardado en cada pago, `ReconcilePayments` | [Claude · 2026-10-22 10:19] despachado (v0.2.0) |
| M4b-1b | `webtyp/driver` (nuevo; búsqueda: `notify/driver.go` tiene la única copia de `Config`/`Setting`/`Field`/`SecretResolver`/`Driver`) + `notify` + `payment_manager` | D9 (procesador elegido en BD por alcance) sin duplicar el patrón de `notify`: extraerlo a una librería que ambos usen | espera M4b-1; crear el repo con `gonew` (lo corre el usuario) |
| M4b-2 | `veltylabs/tuu` (búsqueda: nada de TUU en el ecosistema, `transbank` es el modelo) | `Terminal` por Pago Remoto (no depende de P22); `Gateway` por Payment Intent si P22 = B | espera M4b-1; repo creado sin `gonew`: falta `go.mod`, `.gitignore`, tag |
| M4b-3 | `transbank` | `Provider()` en `Gateway` (ruptura de M4b-1); la mitad en línea de D10 (`Confirm`) espera P22 | espera M4b-1 |
| M4c | `agreement_manager` | D15 + **defecto encontrado**: los hechos de dinero salían por el tema `"money"` (no `money.fact`) e incompletos → `cash_manager` nunca los recibía; porcentaje con `float` | [Claude · 2026-10-22 10:19] despachado (v0.2.0) |
| M4d | `agreement_manager` | operaciones y pantalla (hoy `MountOperations` está vacío: nadie puede saldar desde una pantalla) | espera M4c |
| M9 | `clinical_encounter`, `cash_manager`, `payment_manager` (`ChargeSource` de la atención), QR con `webtyp/qr` | cobro desde la atención, sin papel (D6–D8, D11) | espera M4b-1/2; QR espera P22 |
| M10 | `service_catalog`, `appointment_booking` | abono al reservar (D17) y su devolución | espera M4b-1, P22 |
| M5 | venta de farmacia | venta, cobro por `Terminal`, boleta, stock | espera M4b, CATALOG_SPLIT (F) |
| M6 | `purchase_manager` (nuevo) | compras con costo de lote | después de M3/M4 |
| M7 | contable de `veltylabs` | partida doble | después de M1-M6 |
| M8 | `mjosefa-cms` | menú y permisos por alcance | después de todo |

## Bloqueos

- 2026-10-22 · P22 espera al usuario (preguntar a TUU su tarifa en línea). El contador verifica el modelo de emisión declarado (D13).
- `alert_manager`: UX rechazada; depende de MODULE_GUIDANCE (P15/P16), fuera de esta ola.

## Relación con otras olas

- Misma ola que CATALOG_SPLIT (F espera la caja); M10 toca `service_catalog`.
- D14 fija la forma del servidor para SELFHOST_DEPLOY (túnel, servidor local, 4G).
