# Diagrams — Diagramas de Arquitectura

Esta carpeta contiene los cuatro diagramas de arquitectura de PropConnect. Todos los diagramas están escritos en Mermaid y se renderizan directamente en GitHub.

## Tabla de Diagramas

| Diagrama | Tipo | Descripción |
|---|---|---|
| [01-system-context.md](01-system-context.md) | C4 Nivel 1 | PropConnect como caja única rodeada de actores humanos y sistemas externos |
| [02-bounded-context-map.md](02-bounded-context-map.md) | DDD Context Map | Los 8 bounded contexts agrupados por tipo de dominio con patrones de integración |
| [03-data-flow.md](03-data-flow.md) | Diagramas de Secuencia | Flujos críticos: pago de boost (2 partes), firma de contrato y consulta IA |
| [04-deployment.md](04-deployment.md) | Deployment / Infrastructure | Topología de contenedores, zonas de red, servicios de datos y observabilidad |

## Consistencia con la Documentación Escrita

Todos los diagramas son consistentes con:
- La arquitectura de módulos descrita en `proposals/03-service-module-decomposition.md`.
- Los bounded contexts de `proposals/02-bounded-contexts.md`.
- Las decisiones de infraestructura de los ADRs (ADR-001 al ADR-007).
- Los componentes de infraestructura en `/infrastructure`.
