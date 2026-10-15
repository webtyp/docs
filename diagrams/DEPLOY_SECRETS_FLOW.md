# Flujo de un secreto de despliegue

> STATUS (quitar esta nota al cerrar P21 de DEPLOY_WIZARD_MASTER_PLAN.md): este flujo supone un solo desarrollador. La propuesta para equipos elimina la copia local de los secretos de producción; si se acepta, este diagrama se reescribe.

Un secreto tiene **tres lugares**, cada uno con un papel distinto:

| Lugar | Qué es | Quién lo lee |
|---|---|---|
| **Origen** | Donde nace el valor: Cloudflare lo mintea, el wizard lo genera, o lo entrega un proveedor sin API (consola de Google). | nadie después de entregarlo |
| **Keyring local** (`webtyp/keyring`) | La copia del desarrollador en el almacén de credenciales del sistema operativo. Es la **fuente** desde la que el wizard alimenta los destinos. | el wizard, al aprovisionar |
| **Destino** (nube) | Donde vive en producción: secretos de GitHub Actions (token de deploy) o secretos del Worker (secretos de runtime). | CI al desplegar; el Worker al ejecutarse |

Nunca está en `.env`, en el repo, en un plan ni en el chat.

**Los destinos son de solo escritura**: ni GitHub ni Cloudflare devuelven el valor de un secreto.
Por eso el keyring **no puede traer** un secreto desde la nube. Si una máquina no lo tiene (máquina
nueva, keyring borrado), el wizard lo **vuelve a crear** (mintea o genera uno nuevo y reemplaza el
del destino) o, si viene de un proveedor sin API, **lo pide** al desarrollador.

```mermaid
flowchart TD
    A[El proyecto declara el secreto<br/>config/deploy.go] --> B{¿Está en el<br/>keyring local?}
    B -- sí --> H{¿El destino tiene<br/>la versión actual?<br/>huella en el keyring}
    B -- no --> C{¿De dónde nace?}
    C -- Cloudflare --> D[El wizard mintea un token<br/>con el token de arranque]
    C -- generado --> E[El wizard genera<br/>una clave nueva]
    C -- proveedor sin API --> F[El agente guía paso a paso por MCP<br/>el dev escribe el valor en el wizard]
    D --> G[Se guarda en el keyring local]
    E --> G
    F --> G
    G --> H
    H -- no --> I[El wizard lo copia al destino<br/>por la API del destino]
    I --> J{¿Quién lo usa?}
    H -- sí --> K[Listo para desplegar]
    J -- CI --> L[Secreto de GitHub Actions<br/>token de deploy]
    J -- runtime --> M[Secreto del Worker<br/>p. ej. IAM_CLIENT_PRIVATE_KEY]
    L --> K
    M --> K
```

En producción el keyring no participa: CI lee su secreto de GitHub Actions y el Worker lee los
suyos de sus bindings. El keyring solo interviene cuando alguien aprovisiona o cambia un secreto
desde su máquina.
