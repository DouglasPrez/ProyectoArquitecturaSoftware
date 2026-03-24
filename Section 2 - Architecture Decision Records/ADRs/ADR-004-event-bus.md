# ADR-004: Event Bus / Message Broker

| Campo | Valor |
|---|---|
| **Status** | Aceptado |
| **Fecha** | 24/03/2026 |
| **Decisores** | Carlos Daniel Martínez & Douglas Daniel Pérez Hernández |

## Contexto

PropConnect necesita un mecanismo para la comunicación asíncrona entre módulos, especialmente para flujos reactivos como envío de notificaciones, activación de boosts y actualización de índices de búsqueda. Dado que el sistema es un monolito modular, la decisión va desde soluciones en proceso hasta brokers externos de alta disponibilidad.

## Opciones Consideradas

### Opción 1: EventEmitter2 (In-Process, NestJS)

| Pros | Cons |
|---|---|
| Sin infraestructura adicional — los eventos viajan en memoria | Si el proceso cae, los eventos en vuelo se pierden |
| Latencia de microsegundos | No escala fuera del proceso (no sirve si se migra a microservicios directamente) |
| Setup inmediato, nativo de NestJS | Sin reintentos automáticos ni dead-letter queues |
| Suficiente para volumen inicial | |

### Opción 2: RabbitMQ (Broker externo, AMQP)

| Pros | Cons |
|---|---|
| Durabilidad de mensajes: sobrevive caídas del proceso | Requiere instancia de RabbitMQ (container adicional) |
| Reintentos y dead-letter queues nativas | Latencia de red (aunque mínima en localhost) |
| Listo para microservicios si se extrae un módulo | Overhead operacional para el equipo |
| Soporte nativo en NestJS con `@nestjs/microservices` | |

### Opción 3: Kafka (Broker distribuido, alta durabilidad)

| Pros | Cons |
|---|---|
| Log persistente, replay de eventos, alta throughput | Excesivamente complejo para volumen inicial de PropConnect |
| Ideal para analítica de eventos | Requiere Zookeeper o KRaft, alta curva de aprendizaje |
| Retencion configurable de mensajes | Costo operacional muy alto para startup |

## Decisión

Se adopta **EventEmitter2 (in-process) para la fase inicial**, con una capa de abstracción (`EventBus` interface) que permita reemplazarlo por RabbitMQ sin modificar los módulos consumidores.

**Justificación específica al proyecto:**

- El volumen esperado inicial de PropConnect es bajo (decenas a cientos de eventos por minuto). Kafka y RabbitMQ están sobredimensionados para esta carga.
- Los eventos asíncronos del sistema (notificaciones, índices de búsqueda) no son críticos de misión: si una notificación de email no se envía en un segundo, el negocio no se rompe. El único evento crítico es `PaymentCompleted`, para el cual se implementa el Outbox Pattern (el evento se persiste en una tabla `outbox` dentro de la misma transacción del pago, y un worker lo procesa después).
- La abstracción `EventBus` permite que, cuando el sistema crezca o se extraigan módulos, se reemplace el proveedor por RabbitMQ con cambios solo en la capa de infraestructura, no en los handlers de dominio.

### Diseño de la Abstracción

```typescript
// shared/events/event-bus.interface.ts
export interface IEventBus {
  publish<T>(event: T): void | Promise<void>;
  subscribe<T>(eventName: string, handler: IEventHandler<T>): void;
}

// shared/events/in-memory.event-bus.ts  ← implementación actual
// shared/events/rabbitmq.event-bus.ts   ← implementación futura
```

### Eventos por Volumen Esperado

| Evento | Frecuencia Estimada | Criticidad |
|---|---|---|
| `PaymentCompleted` | Baja (decenas/día) | Alta — usa Outbox Pattern |
| `NotificationSent` | Media (cientos/día) | Baja |
| `AppointmentScheduled` | Media | Baja |
| `ListingPublished` | Baja | Media (afecta índice Elasticsearch) |

## Consecuencias

**Habilita:**
- Despliegue sin dependencias adicionales de infraestructura para comenzar.
- Migración futura a RabbitMQ sin reescribir handlers.

**Previene:**
- No se puede garantizar entrega at-least-once para todos los eventos sin Outbox Pattern. Solo `PaymentCompleted` tiene esta garantía en la fase inicial.

**Riesgos y Mitigaciones:**
- *Pérdida de eventos en memoria*: Para eventos no críticos, se acepta la pérdida eventual. Para `PaymentCompleted`, el Outbox Pattern garantiza que el evento se procese incluso si el proceso reinicia.
- *Migración a RabbitMQ*: Documentada como deuda técnica planificada cuando el sistema alcance ~1000 usuarios activos.

## ADRs Relacionados

- ADR-002 (Estilo de Comunicación) — define cuándo usar eventos vs. llamadas síncronas.
- ADR-001 (Modelo de Despliegue) — contexto del monolito modular.
