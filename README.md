# PropConnect — Marketplace de Bienes Raíces

PropConnect es una plataforma de compra y arrendamiento de inmuebles que conecta a consultores (compradores/arrendatarios), vendedores, asesores inmobiliarios y tramitadores legales en un ecosistema unificado. El sistema resuelve la fricción del proceso inmobiliario tradicional: fragmentado, opaco y dependiente de contactos informales.

---

## Enfoques Arquitectónicos Comparados

| Enfoque | Descripción |
|---|---|
| **Monolito Modular** | Sistema completo en un único proceso desplegable, organizado en módulos con contratos internos explícitos. Sin overhead de red entre contextos. Ver [propuesta completa](Section 1 - Architecture Proposal/proposals/01-high-level-architecture.md). |
| **Microservicios** | Cada bounded context como servicio independiente con su propia base de datos y pipeline de CI/CD.

## Enfoque Recomendado

Se adopta el **Monolito Modular** por la interdependencia transaccional entre dominios, el tamaño reducido del equipo y la necesidad de velocidad de iteración en esta fase del producto.

---

## Tabla de Navegación

### `/proposals` — Propuesta de Arquitectura

| Documento | Descripción |
|---|---|
| [01-high-level-architecture.md](Section%201%20-%20Architecture%20Proposal/proposals/01-high-level-architecture.md) | Comparación Monolito Modular vs. Microservicios con tabla de trade-offs y recomendación justificada |
| [02-bounded-contexts.md](Section%201%20-%20Architecture%20Proposal/proposals/02-bounded-contexts.md) | 8 bounded contexts con entidades, eventos de dominio y mapa de contextos DDD |
| [03-service-module-decomposition.md](Section%201%20-%20Architecture%20Proposal/proposals/03-service-module-decomposition.md) | Estructura completa del repositorio, descripción de módulos y enforcement de límites |
| [04-data-flow-and-interactions.md](Section%201%20-%20Architecture%20Proposal/proposals/04-data-flow-and-interactions.md) | 4 flujos end-to-end con diagramas de secuencia, camino feliz y caminos de fallo |

### `/adrs` — Architecture Decision Records

| Documento | Descripción |
|---|---|
| [ADR-001-deployment-model.md](Section%202%20-%20Architecture%20Decision%20Records/ADRs/ADR-001-deployment-model.md) | Monolito Modular vs. Microservicios — justificación de la elección de despliegue |
| [ADR-002-communication-style.md](Section%202%20-%20Architecture%20Decision%20Records/ADRs/ADR-002-communication-style.md) | Comunicación híbrida: EventBus asíncrono + llamadas síncronas directas |
| [ADR-003-database-strategy.md](Section%202%20-%20Architecture%20Decision%20Records/ADRs/ADR-003-database-strategy.md) | PostgreSQL compartido con prefijos por módulo + Elasticsearch para búsqueda |
| [ADR-004-event-bus.md](Section%202%20-%20Architecture%20Decision%20Records/ADRs/ADR-004-event-bus.md) | EventEmitter2 in-process con interfaz abstracta para migración futura a RabbitMQ |
| [ADR-005-authentication.md](Section%202%20-%20Architecture%20Decision%20Records/ADRs/ADR-005-authentication.md) | Auth propio con JWT + RBAC y ABAC contextual para permisos por recurso |
| [ADR-006-observability.md](Section%202%20-%20Architecture%20Decision%20Records/ADRs/ADR-006-observability.md) | Stack open source: OpenTelemetry + Prometheus + Grafana + Loki |
| [ADR-007-api-design.md](Section%202%20-%20Architecture%20Decision%20Records/ADRs/ADR-007-api-design.md) | REST con JSON y OpenAPI 3.0 para la API pública |

### `/diagrams` — Diagramas de Arquitectura

| Documento | Descripción |
|---|---|
| [01-system-context.md](Section%203%20-%20Architecture%20Diagrams/Diagrams/01-system-context.md) | C4 Nivel 1 — PropConnect como caja única con actores y sistemas externos |
| [02-bounded-context-map.md](Section%203%20-%20Architecture%20Diagrams/Diagrams/02-bounded-context-map.md) | Mapa DDD con agrupación por tipo de dominio y patrones de integración |
| [03-data-flow.md](Section%203%20-%20Architecture%20Diagrams/Diagrams/03-data-flow.md) | Diagramas de secuencia: pago de boost (2 partes), firma de contrato, consulta IA |
| [04-deployment.md](Section%203%20-%20Architecture%20Diagrams/Diagrams/04-deployment.md) | Topología de despliegue: contenedores, zonas de red, servicios externos |

### `/infrastructure` — Componentes de Infraestructura

| Documento | Componente |
|---|---|
| [01-api-gateway.md](Section%204%20-%20Infrastructure%20Components/Infrastructure/01-api-gateway.md) | Nginx — Reverse proxy, terminación SSL, rate limiting |
| [02-database-primary.md](Section%204%20-%20Infrastructure%20Components/Infrastructure/02-database-primary.md) | PostgreSQL 16 — Base de datos primaria con PostGIS |
| [03-cache.md](Section%204%20-%20Infrastructure%20Components/Infrastructure/03-cache.md) | Redis 7 — Caché de búsquedas y sesiones |
| [04-event-bus.md](Section%204%20-%20Infrastructure%20Components/Infrastructure/04-event-bus.md) | EventEmitter2 — Event Bus para comunicación asíncrona |
| [05-authentication.md](Section%204%20-%20Infrastructure%20Components/Infrastructure/05-authentication.md) | Auth JWT — Autenticación y autorización |
| [06-notifications.md](Section%204%20-%20Infrastructure%20Components/Infrastructure/06-notifications.md) | SendGrid + Firebase FCM — Notificaciones por email y push |
| [07-storage.md](Section%204%20-%20Infrastructure%20Components/Infrastructure/07-storage.md) | AWS S3 + CloudFront — Almacenamiento de objetos y CDN |
| [08-observability.md](Section%204%20-%20Infrastructure%20Components/Infrastructure/08-observability.md) | Prometheus + Grafana + Loki — Stack de observabilidad |

---

## Actores del Sistema

| Actor | Rol |
|---|---|
| **Consultor** | Busca inmuebles para comprar o arrendar, agenda citas, usa el asistente IA |
| **Vendedor** | Publica inmuebles, contrata asesores/tramitadores, paga boosts de visibilidad |
| **Asesor** | Ofrece servicios inmobiliarios bajo contrato, accede a datos completos del inmueble |
| **Tramitador** | Gestiona procesos legales entre comprador y vendedor bajo contrato |
| **Administrador** | Supervisa la plataforma, gestiona usuarios y resuelve quejas |

## Stack Tecnológico Principal

| Capa | Tecnología |
|---|---|
| Backend | NestJS (TypeScript) |
| Base de datos | PostgreSQL 16 + TypeORM |
| Búsqueda | Elasticsearch 8 |
| Caché | Redis 7 |
| Pagos | Stripe |
| Notificaciones | SendGrid (email) + Firebase FCM (push) |
| IA | OpenAI API (GPT-4o) |
| Almacenamiento | AWS S3 + CloudFront |
| Observabilidad | Prometheus + Grafana + Loki + OpenTelemetry |
| Reverse Proxy | Nginx |
