# Proposals — Propuesta de Arquitectura

Esta carpeta contiene los cuatro documentos que definen la arquitectura de PropConnect desde el nivel más alto hasta los flujos concretos de datos.

| Documento | Contenido |
|---|---|
| [01-high-level-architecture.md](01-high-level-architecture.md) | Comparación Monolito Modular vs. Microservicios con tabla de trade-offs y recomendación justificada para PropConnect |
| [02-bounded-contexts.md](02-bounded-contexts.md) | 8 bounded contexts con entidades tipadas, eventos de dominio y mapa de relaciones DDD |
| [03-service-module-decomposition.md](03-service-module-decomposition.md) | Árbol de directorios del codebase, descripción de cada módulo y mecanismos de enforcement de límites |
| [04-data-flow-and-interactions.md](04-data-flow-and-interactions.md) | 4 flujos end-to-end con diagramas de secuencia Mermaid, camino feliz y caminos de fallo/compensación |

## Orden de Lectura Recomendado

1. `01-high-level-architecture.md` — define el paradigma arquitectónico adoptado.
2. `02-bounded-contexts.md` — define los dominios y sus contratos.
3. `03-service-module-decomposition.md` — mapea los dominios a código concreto.
4. `04-data-flow-and-interactions.md` — verifica que los dominios y módulos funcionen como sistema.
