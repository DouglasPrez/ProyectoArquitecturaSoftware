# ADRs — Architecture Decision Records

Esta carpeta contiene los registros de decisiones arquitectónicas de PropConnect. Cada ADR documenta el contexto que motivó la decisión, las opciones evaluadas con pros y contras honestos, la decisión tomada y sus consecuencias.

## Tabla de ADRs

| ADR | Título | Estado | Decisión Tomada |
|---|---|---|---|
| [ADR-001](ADR-001-deployment-model.md) | Modelo de Despliegue | Aceptado | Monolito Modular sobre Microservicios |
| [ADR-002](ADR-002-communication-style.md) | Estilo de Comunicación | Aceptado | Híbrido: EventBus asíncrono + llamadas síncronas directas |
| [ADR-003](ADR-003-database-strategy.md) | Estrategia de Base de Datos | Aceptado | PostgreSQL compartido + Elasticsearch para búsqueda |
| [ADR-004](ADR-004-event-bus.md) | Event Bus | Aceptado | EventEmitter2 in-process con interfaz abstracta para migración futura |
| [ADR-005](ADR-005-authentication.md) | Autenticación y Autorización | Aceptado | Auth propio JWT + RBAC/ABAC contextual |
| [ADR-006](ADR-006-observability.md) | Estrategia de Observabilidad | Aceptado | Prometheus + Grafana + Loki + OpenTelemetry (open source) |
| [ADR-007](ADR-007-api-design.md) | Diseño de API | Aceptado | REST con JSON y OpenAPI 3.0 |

## Consistencia entre ADRs

Los ADRs forman un sistema coherente:
- ADR-001 (Monolito) → motiva ADR-002 (comunicación en proceso), ADR-003 (DB compartida), ADR-004 (EventEmitter2).
- ADR-005 (Auth JWT) → es consumido por ADR-007 (todos los endpoints REST requieren JWT).
