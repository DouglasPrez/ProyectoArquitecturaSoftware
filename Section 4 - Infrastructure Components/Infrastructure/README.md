# Infrastructure — Componentes de Infraestructura

Esta carpeta documenta los 8 componentes de infraestructura de PropConnect. Cada archivo explica las responsabilidades del componente, por qué el sistema lo necesita, la tecnología elegida versus alternativas, sus trade-offs y cómo se integra con el resto del sistema.

## Tabla de Componentes

| Archivo | Componente | Tecnología Elegida |
|---|---|---|
| [01-api-gateway.md](01-api-gateway.md) | API Gateway | Nginx |
| [02-database-primary.md](02-database-primary.md) | Base de Datos Primaria | PostgreSQL 16 |
| [03-cache.md](03-cache.md) | Caché | Redis 7 |
| [04-event-bus.md](04-event-bus.md) | Event Bus / Cola de Mensajes | EventEmitter2 (NestJS) |
| [05-authentication.md](05-authentication.md) | Autenticación y Autorización | Auth Propio (JWT + bcrypt) |
| [06-notifications.md](06-notifications.md) | Sistema de Notificaciones | SendGrid + Firebase FCM |
| [07-storage.md](07-storage.md) | Almacenamiento de Objetos | AWS S3 + CloudFront |
| [08-observability.md](08-observability.md) | Stack de Observabilidad | Prometheus + Grafana + Loki + OpenTelemetry |

## Estructura de Cada Componente

Cada componente documenta obligatoriamente:
1. **Responsabilidades** — qué hace específicamente en PropConnect.
2. **Por Qué Este Proyecto lo Necesita** — el problema concreto que resuelve.
3. **Elección de Tecnología** — tabla comparativa con al menos 2 alternativas.
4. **Trade-offs** — ventajas y desventajas honestas de la elección.
5. **Integración** — cómo se conecta al resto del sistema (referencia al diagrama de despliegue).
