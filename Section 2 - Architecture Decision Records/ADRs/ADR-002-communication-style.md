# ADR-002: Estilo de Comunicación entre Módulos

| Campo | Valor |
|---|---|
| **Status** | Aceptado |
| **Fecha** | 24/03/2026 |
| **Decisores** | Carlos Daniel Martínez & Douglas Daniel Pérez Hernández |

## Contexto

Dentro del monolito modular, los 8 módulos de PropConnect necesitan comunicarse entre sí. Algunos flujos requieren respuesta inmediata, mientras que otros son reacciones asíncronas a eventos del dominio. La elección del mecanismo de comunicación afecta el acoplamiento entre módulos y la testabilidad del sistema.

## Opciones Consideradas

### Opción 1: Llamadas directas entre módulos (sincrónico puro)

| Pros | Cons |
|---|---|
| Simple y predecible — fácil de trazar en un debugger | Alto acoplamiento: el emisor debe conocer al receptor |
| Sin infraestructura adicional | Los flujos reactivos (notificaciones) requieren que el módulo principal llame explícitamente a cada dependiente |
| Transaccional por naturaleza | Difícil de extender: agregar un nuevo listener requiere modificar el módulo emisor |

### Opción 2: EventBus interno + llamadas directas para consultas (híbrido)

| Pros | Cons |
|---|---|
| Los módulos emisores no conocen a sus consumidores — bajo acoplamiento | Flujos asíncronos son más difíciles de trazar en debugging |
| Fácil de agregar nuevos listeners sin modificar el emisor | El orden de ejecución de handlers no está garantizado |
| Las consultas síncronas siguen siendo directas y predecibles | Requiere disciplina para no abusar de eventos en flujos que deberían ser síncronos |
| Compatible con extracción futura a microservicios (reemplazar EventBus por Kafka/RabbitMQ) | |

### Opción 3: Message Queue externa (RabbitMQ/Kafka) desde el inicio

| Pros | Cons |
|---|---|
| Preparado para microservicios desde el día uno | Infraestructura adicional innecesaria para un monolito |
| Durabilidad de mensajes garantizada | Complejidad operacional excesiva para el tamaño del equipo |
| 

## Decisión

Se adopta el enfoque **híbrido: EventBus interno para comunicación reactiva, llamadas síncronas directas para consultas**.

**Regla de uso sincrónico**: Un módulo A llama directamente al `public-api.ts` de un módulo B cuando necesita una respuesta inmediata para continuar su lógica. Ejemplo: `Listings` llama a `Contracts.hasActiveContract()` antes de decidir si mostrar los datos completos.

**Regla de uso asíncrono (eventos)**: Un módulo publica un evento cuando ha completado una acción y otros módulos pueden reaccionar de forma independiente. Ejemplo: `Payments` publica `PaymentCompleted` y tanto `Listings` como `Notifications` reaccionan de forma autónoma.

### Ejemplos Concretos del Proyecto

| Flujo | Tipo | Justificación |
|---|---|---|
| Listings verifica contrato activo antes de mostrar datos | Síncrono | Necesita respuesta inmediata para el request HTTP |
| Notifications envía email tras `PaymentCompleted` | Asíncrono | El pago ya completó; la notificación no bloquea al pagador |
| AI Consultant consulta listings disponibles | Síncrono | Necesita los datos para generar la respuesta IA |
| Listings actualiza boost tras `PaymentCompleted` | Asíncrono | El evento ya ocurrió; la activación del boost puede ser eventual |
| Contracts verifica que la publicación esté activa | Síncrono | Decisión de negocio que necesita dato actual |

## Consecuencias

**Habilita:**
- El módulo `Notifications` puede agregarse o quitarse sin modificar los módulos que generan eventos.
- Los módulos emisores no necesitan saber quién consume sus eventos.
- En el futuro, el EventBus interno puede reemplazarse por RabbitMQ con cambios mínimos en los handlers.

**Previene:**
- Se prohíbe usar eventos para reemplazar consultas síncronas (request/response). Si un módulo necesita un dato para procesar una petición HTTP, debe usar una llamada directa, no publicar un evento y esperar la respuesta.

**Riesgos y Mitigaciones:**
- *Eventos perdidos*: Al ser un EventBus en memoria, si el proceso cae durante el manejo de un evento, el evento se pierde. Se mitiga con idempotencia en los handlers y, para eventos críticos (como `PaymentCompleted`), se persiste el evento en la base de datos antes de publicarlo (Outbox Pattern lite).

## ADRs Relacionados

- ADR-001 (Modelo de Despliegue) — define el contexto del monolito modular.
- ADR-004 (Event Bus) — justifica la elección de EventEmitter2 vs. brokers externos.
