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
