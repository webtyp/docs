# Usabilidad de las apps — master plan
> **Status:** PENDING · 2026-11-04 · anotado desde `mjosefa-cms` con 8 módulos; no se trabaja hasta que el dueño lo abra

Problemas de diseño y uso vistos al cargar datos reales. Solo anotados: cada uno se convierte en pregunta con
opciones y capturas cuando se abra la ola.

## Preguntas abiertas

Ninguna todavía (se escriben al abrir la ola).

## Decisiones por implementar

| # | Decisión | Dónde se implementa |
|---|---|---|
| U1 | Principio del dueño: **un módulo aporta un solo enlace** al menú (responsabilidad única); varios enlaces se justifican por escrito (p. ej. seguridad: personas, roles, asignaciones). La pantalla del módulo muestra de forma simple lo más que se puede hacer ahí, para no saltar entre módulos | `layout`, `appkit`, todos los módulos |
| U2 | **La app agrupa** los enlaces de sus módulos en el menú según su preferencia; los demos de cada módulo prueban con sus propios menús | `appkit.App`, `layout` |
| U3 | En listas largas el **filtro va primero** (p. ej. servicios por especialidad) y la búsqueda después; para filtrar ya existe `components/selectsearch` | `crudview`, módulos con listas largas |

## Fases

| Fase | Repo | Qué | Estado |
|---|---|---|---|
| A1 | `layout` | (Bloquea uso diario) El menú lateral se corta en escritorio (y en móvil al abrir «Usuarios y roles»): no tiene desplazamiento | por diseñar |
| A2 | `appointment_booking`, `staff_manager` | Dos «Personal» en el menú (fichas vs horario y servicios por profesional) | por diseñar, con U1 |
| A3 | todos los módulos | Revisar cuántos enlaces aporta cada uno y justificar los que sean más de uno | por diseñar |
| A4 | `appkit`, `layout` | Agrupación del menú declarada en la app (U2): qué cambia en `appkit.App` | por diseñar |
| A5 | `service_catalog` y otros | Filtro por especialidad antes que búsqueda (U3) | por diseñar |
| F | `mjosefa-cms` | Capturas de cada pantalla con el menú nuevo | al final |

## Bloqueos

- Ninguno.

## Relación con otras olas

- DATA_IMPORT (`mjosefa-cms`): de ahí salen estos hallazgos; A1 es el único que molesta ya.
- UI_CONSISTENCY: reglas visuales de formularios y colores; esta ola trata navegación y flujo.
