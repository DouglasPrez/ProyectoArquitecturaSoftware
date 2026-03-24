# ADR-007: Diseño de API

| Campo | Valor |
|---|---|
| **Status** | Aceptado |
| **Fecha** | 24/03/2026 |
| **Decisores** | Carlos Daniel Martínez & Douglas Daniel Pérez Hernández |

## Contexto

PropConnect necesita definir el estilo de API para sus consumidores. Los consumidores principales son: una aplicación web (SPA), una aplicación móvil futura, y el webhook de Stripe. Los módulos internos se comunican directamente en proceso (no necesitan una API de red entre sí). La decisión afecta la experiencia del desarrollador frontend, la flexibilidad de las consultas y la documentación.

## Opciones Consideradas

### Opción 1: REST (HTTP + JSON)

| Pros | Cons |
|---|---|
| Estándar universal, conocido por todos los desarrolladores frontend | Over-fetching / under-fetching en consultas complejas |
| Versionado sencillo (`/v1`, `/v2`) | Múltiples endpoints para datos relacionados |
| Cacheable por naturaleza (GET es idempotente) | |
| Documentación automática con OpenAPI/Swagger | |
| Fácil de consumir desde cualquier cliente HTTP | |

### Opción 2: GraphQL

| Pros | Cons |
|---|---|
| El cliente solicita exactamente los campos que necesita | Curva de aprendizaje para el equipo backend |
| Un solo endpoint para todas las consultas | Sin caché HTTP nativo — requiere caché a nivel de resolver |
| Ideal para datos relacionados complejos (ej. listing + seller + reviews en un query) | Complejidad adicional para file uploads |
| Subscriptions para actualizaciones en tiempo real | Harder to secure (introspection, DoS por queries profundas) |

### Opción 3: gRPC

| Pros | Cons |
|---|---|
| Alto rendimiento y tipos fuertes (Protobuf) | No soportado nativamente por navegadores sin grpc-web |
| Ideal para comunicación interna entre servicios | Overkill para una API pública de un monolito |
| Streaming bidireccional | |

## Decisión

Se adopta **REST con JSON** para la API externa, con **OpenAPI 3.0** como contrato de documentación.

**Justificación específica al proyecto:**

- Los consumidores de PropConnect (web + móvil) son clientes HTTP estándar que se benefician de la cacheabilidad de REST. Las búsquedas de inmuebles (`GET /listings?type=SALE&zone=10&maxPrice=300000`) son perfectamente modelables como recursos REST.
- El problema de over-fetching de REST se mitiga con query parameters de selección de campos en endpoints de detalle (`?fields=id,title,price`) para las operaciones más comunes.
- GraphQL aportaría beneficios reales si los clientes necesitaran componer datos de muchos módulos en un solo request. En PropConnect, la mayoría de las pantallas consumen datos de 1–2 recursos relacionados, lo que no justifica la complejidad.
- gRPC no es viable para el frontend web sin grpc-web, lo que añade una capa de proxy innecesaria.

### Convenciones REST

| Convención | Aplicación |
|---|---|
| Sustantivos en plural para recursos | `/listings`, `/contracts`, `/appointments` |
| Verbos HTTP semánticos | `GET` (leer), `POST` (crear), `PATCH` (actualizar parcialmente), `DELETE` (eliminar) |
| Respuestas de error estructuradas | `{ error: { code, message, details } }` |
| Paginación con cursor | `{ data: [...], nextCursor: "...", hasMore: true }` |
| Versionado en URL | `/api/v1/...` |

### Documentación

Todos los endpoints se documentan con decoradores de `@nestjs/swagger`. La especificación OpenAPI se expone en `/api/docs` en entornos de staging y desarrollo.

## Consecuencias

**Habilita:** Integración sencilla desde frontend web y apps móviles. Cacheabilidad de responses de Listings en el CDN.

**Previene:** No se implementa GraphQL en esta fase. Si el frontend necesita componer datos complejos en el futuro, se evaluará agregar un BFF (Backend for Frontend) o GraphQL encima del REST existente.

**Riesgos y Mitigaciones:** El endpoint de búsqueda de Listings puede volverse un cuello de botella si se permiten queries sin filtros. Se mitiga requiriendo al menos un filtro (tipo o zona) en la búsqueda paginada.

## ADRs Relacionados

- ADR-005 (Autenticación) — todos los endpoints REST usan los guards de JWT definidos en IAM.
