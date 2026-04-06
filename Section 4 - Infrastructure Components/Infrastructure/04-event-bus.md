# 04 — Event Bus / Cola de Mensajes (EventEmitter2)

## 1. Responsabilidades

- Propagar eventos de dominio entre módulos del monolito de forma asíncrona y desacoplada.
- Permitir que múltiples módulos reaccionen al mismo evento sin que el emisor los conozca (ej. `PaymentCompleted` es consumido por Listings, Notifications y AI Consultant).
- Proveer la interfaz `IEventBus` que abstraiga la implementación concreta, permitiendo migrar a RabbitMQ sin cambiar los handlers de dominio.
- Implementar el Outbox Pattern para el evento `PaymentCompleted`, garantizando entrega at-least-once.

## 2. Por Qué Este Proyecto lo Necesita

Sin un mecanismo de eventos, el módulo `Payments` tendría que llamar directamente a `Listings`, `Notifications` y cualquier otro módulo que reaccione al pago. Esto crearía acoplamiento fuerte: Payments necesitaría saber quién existe y cómo llamarlo. El EventBus invierte esta dependencia: Payments publica el evento y no sabe quién lo consume.

## 3. Elección de Tecnología

| Dimensión | EventEmitter2 (elegido, fase inicial) | RabbitMQ | Kafka |
|---|---|---|---|
| **Gestionado / Self-hosted** | In-process (sin infraestructura) | Self-hosted | Self-hosted o cloud |
| **Complejidad operacional** | Nula | Baja-media | Alta |
| **Costo a nuestra escala** | Gratuito | Gratuito | Gratuito (self-hosted) |
| **Característica diferenciadora** | Sin overhead de red, setup inmediato | Dead-letter queues, reintentos, durabilidad | Log persistente, replay, alta throughput |

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| Sin infraestructura adicional para arrancar | Eventos en memoria: se pierden si el proceso reinicia |
| Latencia de microsegundos (en proceso) | Sin reintentos automáticos ni dead-letter queues |
| Nativo en NestJS, documentado y testeado | No escala fuera del proceso (no sirve para microservicios) |
| Interfaz abstracta facilita migración futura | El Outbox Pattern debe implementarse manualmente para eventos críticos |

## 5. Integración

EventEmitter2 corre dentro del proceso `propconnect-api`. No requiere contenedor propio. Los eventos críticos (`PaymentCompleted`) se persisten en la tabla `pay_outbox` de PostgreSQL dentro de la misma transacción antes de ser publicados. Un worker interno los lee y republica con reintentos si el handler falla. Ver ADR-004 y `04-deployment.md`.
