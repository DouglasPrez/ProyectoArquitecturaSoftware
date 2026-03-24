# ADR-006: Estrategia de Observabilidad

| Campo | Valor |
|---|---|
| **Status** | Aceptado |
| **Fecha** | 24/03/2026 |
| **Decisores** | Carlos Daniel Martínez & Douglas Daniel Pérez Hernández |

## Contexto

PropConnect necesita visibilidad sobre el comportamiento del sistema en producción: errores en pagos, latencia del módulo de IA, tasa de fallos en notificaciones, y rendimiento general. Al ser un monolito modular, la observabilidad es más simple que en microservicios, pero sigue siendo necesaria para diagnosticar problemas rápidamente y cumplir con SLOs básicos.

## Opciones Consideradas

### Opción 1: Stack Propietario (Datadog / New Relic)

| Pros | Cons |
|---|---|
| Producto listo con dashboards, APM y alertas | Costo elevado y proporcional al volumen de datos |
| Agente que instrumenta automáticamente NestJS | Vendor lock-in |
| Correlación de logs, métricas y trazas en una sola UI | Excesivo para las necesidades iniciales |

### Opción 2: Stack Open Source (Prometheus + Grafana + Loki + OpenTelemetry)

| Pros | Cons |
|---|---|
| Sin costo de licencia | Requiere configuración y mantenimiento propio |
| Estándar de la industria — portabilidad total | Curva de aprendizaje para configurar alertas y dashboards |
| OpenTelemetry es vendor-neutral | |
| Se integra bien con NestJS (`@opentelemetry/sdk-node`) | |

## Decisión

Se adopta el **stack open source: OpenTelemetry + Prometheus + Grafana + Loki**.

**Justificación:** El costo de los stacks propietarios no está justificado en la fase inicial. El stack open source es el estándar de la industria y su adopción desde el inicio facilita la migración futura a microservicios sin cambiar la estrategia de observabilidad.

### Tres Pilares

| Pilar | Herramienta | Uso en PropConnect |
|---|---|---|
| **Métricas** | Prometheus + Grafana | Tasa de pagos completados/fallidos, latencia de endpoints críticos, uso de CPU/memoria |
| **Logs** | Loki + Grafana | Logs estructurados en JSON con `correlationId` en cada request |
| **Trazas** | OpenTelemetry + Tempo | Trazar el flujo completo de un pago desde el request HTTP hasta el webhook de Stripe |

### SLOs Definidos

| Indicador | Objetivo | Alerta si |
|---|---|---|
| Disponibilidad del API | 99.5% mensual | < 99% en ventana de 1 hora |
| Latencia P95 endpoints de Listings | < 300 ms | > 500 ms sostenido por 5 min |
| Latencia P95 webhook de Stripe | < 2 s | > 5 s |
| Tasa de éxito de pagos | > 95% | < 90% en ventana de 30 min |

## Consecuencias

**Habilita:** Diagnóstico rápido de regresiones en producción. Trazabilidad del flujo de pagos para resolver disputas.

**Riesgos y Mitigaciones:** La configuración inicial del stack requiere tiempo de setup. Se mitiga con una imagen Docker preconfigurada de `prometheus + grafana + loki` en el `docker-compose.yml` de desarrollo.

