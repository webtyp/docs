# Dinero y contabilidad — razonamiento y decisiones (histórico)

Estado vivo y fases: [MONEY_FLOWS_MASTER_PLAN.md](MONEY_FLOWS_MASTER_PLAN.md). Este documento guarda el porqué; no se vuelve a leer para saber qué falta.

## 1. Objetivo y alcance

`mjosefa-cms` mueve dinero por varios caminos a la vez: vende **servicios** (reservas), vende en la **farmacia** al público, **compra** medicamentos e insumos, **consume** insumos en las atenciones y debe **pagar** a profesionales y proveedores. Hoy cada módulo guarda su pedazo y nadie suma.

**Principio (usuario): el sistema es reutilizable por cualquier negocio; `mjosefa-cms` es solo el primer caso.** Nada de lo construido nombra a una clínica; lo propio de cada negocio (plan de cuentas, reglas, comisiones) es configuración que pone la aplicación.

Dentro de alcance: el modelo de dinero (libro, cuentas, asientos), cómo cada módulo informa, la caja, las cuentas por pagar y reportes simples. Fuera de alcance por ahora: documentos tributarios electrónicos y contabilidad legal (balance, libros oficiales) — ver D1.

## 2. Contexto del usuario (2026-10-19)

Contador externo (la contabilidad propia de `veltylabs` es un módulo futuro inyectable). Farmacia cobra con máquina de tarjeta. Cada profesional emite su propia boleta; el paciente paga antes en caja y vuelve con el comprobante. Cobra el profesional; la administradora necesita saber cuánto pagarle. No hay Fonasa/Isapre (los `Agreement` de `item_catalog` probablemente no se usan). Dos cajas y dos máquinas de tarjeta (consultorio, farmacia) — luego corregido: no son cajas fijas, es el turno de quien cobra (ver P9 en el master).

## 3. Qué hace la industria

Partida doble (Modern Treasury, Stripe-style ledgers): transacción que suma cero, no se edita, saldos calculados. ERPNext/Odoo: los documentos operativos generan solos los asientos; los módulos de operación no saben de contabilidad. Xero/QuickBooks/Wave: por fuera «entró/salió/debes», partida doble por debajo. Chile: prestaciones médicas ambulatorias pueden estar exentas de IVA (art. 12 letra E n.º 19 Ley IVA); medicamentos pagan IVA 19 %.

## 4. Recomendación aceptada (D1)

**Dos etapas, mismo contrato en el medio.** Ahora: hechos económicos (`webtyp/money`: `Money`, `Fact`, `Sink`, idempotencia) publicados por cada módulo de operación vía `events.Publisher` — no conocen la contabilidad. Después: módulo contable de `veltylabs` (otro `Sink`) traduce los hechos a partida doble; se inyecta sin tocar los módulos de operación. Permisos: cada hecho lleva su alcance.

Piezas: `webtyp/money`, módulos de operación publican hechos, `cash_manager` (cajas/turnos, primer `Sink`), `agreement_manager` (terceros/acuerdos, antes `provider_payouts`).

## 5. Acuerdos con terceros (6b) — los tres casos reales del usuario

| Caso | Quién | Cómo se calcula | Dirección |
|---|---|---|---|
| Arriendo de box | 2 kinesiólogos, 1 laboratorio | valor fijo mensual, algunos con insumos incluidos | tercero paga al negocio |
| Sueldo fijo | farmacéutica, matrón, médico general | monto fijo mensual | negocio paga al tercero |
| Porcentaje | doctores «a porcentaje» | % del servicio (ej. dental 50 %) | negocio paga su parte |

Idea común: un **acuerdo** con **cláusulas** (reparto de servicio, monto fijo a pagar, monto fijo a cobrar) + **inclusiones** de insumos editables con vigencia. `agreement_manager` reemplaza a `provider_payouts` (sin consumidores, solo modelos).

## 6. Preguntas A–E, G y sus respuestas (resumen de decisiones)

- **A (arriendo):** tercero persona o empresa con RUT; exclusivo o por turnos (`room_layout`); cobra directo a sus pacientes; forma de pago/documento/IVA quedan como **configuración del acuerdo**, no código.
- **B (insumos incluidos):** las tres reglas de inclusión (por paciente, cantidad mensual, a demanda); **con stock** (cada entrega es una salida imputada al acuerdo) — la única forma que permite avisar antes de que falte.
- **C (sueldo fijo):** el sistema calcula **lo que se le paga** (monto fijo + ajustes) en un estado de cuenta ordenado; **no** la liquidación legal (AFP, salud, impuesto) — eso lo hace el contador.
- **D (porcentaje):** la tarifa vive en el **servicio**; el trato individual es una **excepción registrada** (profesional × servicio o categoría, con motivo y vigencia), nunca un acuerdo invisible. Orden de resolución: excepción por servicio → excepción por categoría → tarifa estándar. Nace al **realizarse** la atención (cobrada); una devolución la revierte. Fórmulas especiales (mínimo garantizado, tope, tramos, bono) son **configurables y opcionales**: si no se activan, no aparecen en ninguna pantalla.
- **E (transversales):** una persona puede tener varios acuerdos vigentes; un cambio de condiciones rige solo hacia adelante; un rol administra los acuerdos y el afectado ve los suyos (necesita usuario — F2).
- **G (avisos, 6c):** nivel mínimo lo fija la persona a cargo; avisa al encargado del área (detalle) y a la administradora un resumen (solo conteos); correo primero, canales configurables; **G4: reposición en modo manual** (lista bajo el mínimo + cantidad sugerida + costo estimado) **+ caja de los próximos 30 días por método directo** (saldo hoy + lo que se sabe que entra − lo que se sabe que sale, con fecha conocida) — no automático (decisión de compra es de una persona) ni proyección larga (falta historial).

## 7. Tercera y cuarta ronda — cambios y hallazgos (6e–6g)

- **P6 cambia:** nada manual. El documento tributario de la **farmacia** se emite automáticamente por un puerto `Issuer` con proveedor certificado; las consultas no generan documento (cada profesional emite el suyo); el comprobante de pago de caja lo genera el sistema solo.
- **P9 cambia:** no hay «2 cajas» fijas — la unidad es el **turno de caja de una persona** (abre, cobra, cierra con arqueo de efectivo); el alcance sale de los permisos de quien cobra, no de una caja física. Tarjetas no se concilian a mano (eso lo ve el contador en el portal de la máquina).
- **P10 cambia:** el contador tiene su propia cuenta de solo lectura; no hay «Exportar» como flujo central.
- **P8:** `provider_payouts` → **`agreement_manager`** (hecho, v0.1.8+).
- **K1 (TUU):** la clínica emite con la máquina TUU hoy. Decisión: **dos puertos intercambiables** (terminal de cobro + emisor de documento tributario), como `notify`. **No se construye nada hasta saber si TUU tiene API** — sigue abierta (ver master, Bloqueos).
- **K3 (WhatsApp):** solo el teléfono, sin API de Business — WhatsApp queda **fuera de la primera etapa**.
- **K4 (correo):** dominio `monjitaschillan.cl` en Cloudflare, cuenta gratuita. Envío a direcciones **verificadas de la propia cuenta** es gratis en cualquier plan (sirve para avisos al personal); a terceros requiere Workers de pago.

## 8. Duplicaciones encontradas y corregidas (6h)

Regla nueva escrita en `api-design` («Step 0»): antes de crear algo, buscar en `webtyp/*`/`veltylabs/*`. Encontrado y corregido: `notify.Urgency` → se usa `msgtype.Type` (ya existe); una alerta viaja como `events.Event` (no un bus propio); `money.Fanout` eliminado (`events.Fanout` ya existe, un hecho es un `events` payload con `money.Publish`/`Subscribe`); `money.Sink` se mantiene junto a `events` porque expresa reintento/idempotencia que un `Handler` fire-and-forget no puede. `webtyp/bus` vs `webtyp/events`: duplicación **previa**, señalada, no resuelta aquí.

## 9. Compuerta api-design

Prior art: Modern Treasury, ERPNext/Odoo (asientos desde documentos), diseño previo de la clínica (`accounting_journal`/`accounting_line`). El cambio no borra API pública; `payment_manager` solo agrega eventos.
