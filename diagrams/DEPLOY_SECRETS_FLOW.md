# Flujo de un secreto de despliegue

Pensado para un **equipo**: cualquier persona autorizada aprovisiona, rota y despliega sin recibir
nada de otra.

| Lugar | Qué guarda | Quién lo lee |
|---|---|---|
| **Destino** (nube) | Los secretos de **producción**: secretos de GitHub Actions (token de deploy, ID de cuenta) y secretos del Worker (secretos de runtime). Es el **único** lugar donde existen. | CI al desplegar; el Worker al ejecutarse |
| **Keyring de cada persona** (`webtyp/keyring`) | Solo **lo personal**: su token de arranque de Cloudflare y los secretos de **su** entorno de desarrollo. | el wizard de esa persona; el servidor de desarrollo local |
| **Origen** | Donde nace un valor de producción: Cloudflare lo mintea, el wizard lo genera, o la persona lo escribe en el wizard (proveedor sin API). Se entrega al destino y no se guarda en ninguna otra parte. | nadie después de entregarlo |

Nunca en `.env`, en el repo, en un plan ni en el chat.

**Los destinos son de solo escritura**: ni GitHub ni Cloudflare devuelven el valor. Por eso nadie
"recupera" un secreto de producción: **se rota** (se crea uno nuevo y reemplaza al anterior). Saber
si un secreto existe y cuándo cambió se lee de los metadatos del destino (nombres y `updated_at`),
nunca del valor.

```mermaid
flowchart TD
    A[El proyecto declara el secreto<br/>config/deploy.go] --> B{¿Para dónde es?}
    B -- desarrollo local --> C{¿Está en MI keyring?}
    C -- sí --> Z[Listo]
    C -- no --> D[El wizard me lo pide<br/>y lo guarda en mi keyring]
    D --> Z
    B -- producción --> E{¿El destino ya lo tiene?<br/>se lee por nombre}
    E -- sí --> F{¿Se pidió rotar?}
    F -- no --> Z
    F -- sí --> G
    E -- no --> G{¿De dónde nace?}
    G -- Cloudflare --> H[Se mintea un token de la cuenta<br/>con mi token de arranque]
    G -- aleatorio --> I[El wizard genera<br/>un valor nuevo]
    G -- proveedor sin API --> J[El agente guía paso a paso por MCP<br/>la persona lo escribe en el wizard]
    H --> K[Se entrega directo al destino<br/>sin copia local]
    I --> K
    J --> K
    K --> L{¿Quién lo usa?}
    L -- CI --> M[Secreto de GitHub Actions]
    L -- runtime --> N[Secreto del Worker]
    M --> Z
    N --> Z
```

Dar de baja a una persona no exige rotar nada: se la quita de la cuenta de Cloudflare y de la
organización de GitHub. Los tokens por proyecto pertenecen a la cuenta, no a quien los creó.
