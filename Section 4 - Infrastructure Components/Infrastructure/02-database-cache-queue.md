# 02 — Base de Datos Primaria (PostgreSQL)

## 1. Responsabilidades

- Almacenar todos los datos transaccionales del sistema: usuarios, publicaciones, contratos, pagos, citas y reseñas.
- Garantizar consistencia ACID en operaciones críticas como la firma de contratos y el procesamiento de pagos.
- Proveer las funcionalidades de búsqueda geoespacial básica (con `PostGIS`) para consultas de proximidad de inmuebles.
- Ejecutar migraciones de esquema versionadas a través de TypeORM Migrations.
- Generar backups automáticos diarios a AWS S3 via `pg_dump`.

## 2. Por Qué Este Proyecto lo Necesita

PropConnect tiene flujos de negocio que requieren transacciones ACID entre múltiples entidades. Por ejemplo: firmar un contrato implica actualizar el `Contract` a ACTIVE, crear un `AccessGrant` y descontar la comisión reservada — estas tres operaciones deben ser atómicas. Un sistema sin ACID (como MongoDB en modo básico) requeriría lógica de compensación manual para este tipo de flujos.

## 3. Elección de Tecnología

| Dimensión | PostgreSQL 16 (elegido) | MySQL 8 | MongoDB |
|---|---|---|---|
| **Gestionado / Self-hosted** | Self-hosted (Docker) | Self-hosted | Self-hosted o Atlas |
| **Complejidad operacional** | Media | Media | Baja (schema-less) |
| **Costo a nuestra escala** | Gratuito | Gratuito | Gratuito (self-hosted) |
| **Característica diferenciadora** | ACID fuerte + PostGIS + JSON nativo + extensiones ricas | ACID sólido, menor soporte de extensiones | Flexibilidad de esquema, sin JOINs eficientes |

PostgreSQL se elige por su soporte nativo de `PostGIS` para geoespacial, su madurez con ORM como TypeORM, y su robustez en operaciones transaccionales complejas.

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| ACID completo para flujos de pago y contratos | Escalabilidad horizontal de escrituras más compleja que NoSQL |
| PostGIS para consultas de ubicación sin servicio externo | Requiere gestionar migraciones cuidadosamente al evolucionar el esquema |
| TypeORM tiene soporte maduro para PostgreSQL | Un punto único de fallo si no se configura replicación (mitigado con backups frecuentes) |
| JSON nativo (`jsonb`) para campos flexibles como `preferences` en AI | |

## 5. Integración

PostgreSQL corre en su propio contenedor Docker con un volumen persistente (`pg_data`). El contenedor `propconnect-api` se conecta en la red interna del VPC. Los backups se envían a S3 mediante un cronjob nativo del contenedor. Ver `04-deployment.md`.

---

# 03 — Caché (Redis)

## 1. Responsabilidades

- Cachear respuestas de búsqueda de Listings frecuentemente solicitadas (top búsquedas por zona y tipo).
- Almacenar refresh tokens con TTL automático para la revocación eficiente de sesiones.
- Guardar temporalmente los resultados de recomendaciones de IA para evitar llamadas repetidas a OpenAI con los mismos parámetros.
- Servir como almacenamiento de rate-limiting distribuido (contador de intentos de login fallidos por IP).

## 2. Por Qué Este Proyecto lo Necesita

Sin Redis, cada búsqueda de Listings generaría una query a PostgreSQL + una llamada a Elasticsearch. En hora pico (muchos consultores buscando simultáneamente en la misma zona), esto saturaría innecesariamente la base de datos. Las búsquedas por zona/tipo son altamente repetibles y sus resultados son válidos por 2–5 minutos, haciendo la caché muy efectiva.

## 3. Elección de Tecnología

| Dimensión | Redis 7 (elegido) | Memcached | PostgreSQL unlogged tables |
|---|---|---|---|
| **Gestionado / Self-hosted** | Self-hosted | Self-hosted | Ya existe |
| **Complejidad operacional** | Baja | Baja | Muy baja (sin infraestructura extra) |
| **Costo a nuestra escala** | Gratuito | Gratuito | Gratuito |
| **Característica diferenciadora** | TTL nativo + estructuras de datos + pub/sub | Solo key-value simple, sin TTL por key | No es un caché real — contención de locks |

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| TTL automático por key — invalidación sin cronjobs | Datos volátiles: si Redis cae, el sistema funciona pero más lento |
| Estructuras de datos nativas (hashes, sorted sets) para rate limiting | Consumo de RAM debe ser monitoreado (sin persistencia crítica) |
| Latencia de microsegundos para lecturas | Datos eventualmente stale si el TTL es largo — aceptable para búsquedas de listings |

### Estrategia de Invalidación de Caché

| Dato Cacheado | TTL | Invalidación Adicional |
|---|---|---|
| Búsqueda de Listings por filtros | 3 minutos | Se invalida al recibir `ListingPublished` o `ListingSoldOrRented` |
| Recomendaciones IA por usuario+preferencias | 10 minutos | Solo por TTL (preferencias cambian poco) |
| Refresh tokens | 30 días | Se elimina explícitamente en logout o suspensión |
| Rate limit counters | 15 minutos | Solo por TTL |

## 5. Integración

Redis corre en un contenedor Docker sin volumen persistente (datos efímeros). El contenedor `propconnect-api` accede a Redis en la red interna. El cliente utilizado es `ioredis` con NestJS CacheModule. Ver `04-deployment.md`.

---

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
