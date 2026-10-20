# Dinero y contabilidad de la clínica — master plan
> **Status:** IN PROGRESS · 2026-10-20 · D1, P2–P11, P6/P9/P10 (con cambios), P8 (rename) y H1–H3/K2–K4 cerradas; **M0–M4 despachadas o cerradas**; **K1 (TUU) sigue abierta** y bloquea M4b/M5

Por qué y cómo (histórico): [MONEY_FLOWS_DESIGN.md](MONEY_FLOWS_DESIGN.md). Forma de trabajo: skill **master-plan**.

## Preguntas abiertas

**K1 — ¿TUU (la máquina de la farmacia) tiene API?** Sin esto no se construyen el puerto de terminal de cobro ni el emisor de boleta (M4b/M5): no hay respaldo manual (ver D-P6 en el histórico). *Pregúntalo a TUU y al contador; mientras no haya respuesta, M4b/M5 quedan fuera de la ola.*

## Decisiones por implementar

| # | Decisión | Dónde |
|---|---|---|
| D1 | Cableado de hechos ahora; contabilidad es módulo inyectable después | `webtyp/money` (hecho) |
| P9 | La caja es el **turno de una persona** (abre, cobra, cierra con arqueo de efectivo), no una caja fija | M2, M3 |
| P6 | La farmacia **emite sola** su boleta (puerto `Issuer`); consultas no generan documento; comprobante de pago lo genera el sistema | M4b, M5 |
| D2 (6b) | `agreement_manager` modela los tres casos (arriendo, sueldo, porcentaje) con cláusulas + inclusiones de insumos con stock | M4 |
| D3 (6b, D) | La tarifa vive en el servicio; el trato individual es una excepción registrada y visible (profesional × servicio/categoría, con motivo y vigencia) | M4 |
| D4 (6c, G4) | Reposición en modo **manual** + caja de los próximos 30 días por **método directo** | M3 |
| K3 | WhatsApp fuera de la primera etapa (correo + dentro de la app) | hecho (`notify` v0.2.1) |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| M0 | `webtyp/notify`, `veltylabs/email`, `veltylabs/whatsapp` | puerto `Channel`, canales configurables en BD, secretos por nombre | cerrada (`notify` v0.2.1, `email` v0.1.1, `whatsapp` v0.0.2 solo con `PRICING.md`) |
| M0b | `alert_manager` | quién recibe qué alerta, registro de envíos | **local, sin publicar, UX rechazada por el usuario** — redibujo espera MODULE_GUIDANCE P15/P16 (no es parte de esta ola) |
| M1 | `webtyp/money` | `Money`, `Fact`, `Sink`, eventos (`Publish`/`Subscribe`, sin `Fanout`), idempotencia | cerrada v0.1.3 |
| M2 | `payment_manager` | número de comprobante generado por el sistema, publicar hechos (`money.Publish`) | **cerrada** v0.1.21 |
| M3 | `veltylabs/cash_manager` (nuevo) | turnos por persona, apertura/cierre con arqueo, primer `Sink`, pantallas «Hoy»/«Cierre»/«Reposición»/«Caja 30 días», rol de contador solo lectura | `[Antigravity · 2026-10-20 18:23]` en curso (sesión 767920940306581773, implementando etapas 4-6) |
| M4 | `agreement_manager` | terceros, acuerdos (3 cláusulas), inclusiones con stock, excepciones de tarifa, devengo, «por pagar»/«por cobrar», estado de cuenta | **cerrada** v0.1.13 |
| M4b | puertos de **terminal de cobro** + **emisor de documento tributario** | contrato tipo `notify`, implementación TUU si aplica | espera K1 |
| M5 | módulo de venta de farmacia | venta, cobro por el puerto de terminal, boleta automática, salida de stock | espera M4b, CATALOG_SPLIT (F) |
| M6 | `purchase_manager` (nuevo) | compras con costo de lote | después de M3/M4 |
| M7 | módulo contable de `veltylabs` | partida doble, después de producción | después de M1-M6 |
| M8 | `mjosefa-cms` | menú y permisos por alcance | después de todo |

## Bloqueos

- K1 (TUU) sin responder: M4b y M5 no se construyen.
- `alert_manager`: UX rechazada por el usuario (no explica nada a quien no conoce el sistema); redibujo depende de MODULE_GUIDANCE_MASTER_PLAN.md (P15/P16), fuera de esta ola.

## Relación con otras olas

Misma ola que CATALOG_SPLIT (P4/P5 ya cerradas, la desbloquean). Independiente de UI_CONSISTENCY, SECURITY_BY_DEFAULT y API_SURFACE. Toca `DATA_IMPORT_MASTER_PLAN` de `mjosefa-cms` solo si se importan pagos históricos (hoy no).
