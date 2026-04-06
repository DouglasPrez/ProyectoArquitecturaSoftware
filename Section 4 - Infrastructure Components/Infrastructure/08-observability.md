# 08 — Stack de Observabilidad (Prometheus + Grafana + Loki + OpenTelemetry)

## 1. Responsabilidades

- Recolectar métricas del sistema (CPU, memoria, latencia de endpoints, tasa de errores) con Prometheus mediante el endpoint `/metrics` de NestJS.
- Agregar y visualizar métricas y logs en dashboards de Grafana para monitoreo en tiempo real.
- Centralizar los logs estructurados en JSON del monolito con Loki, con búsqueda por `correlationId`, módulo y nivel de severidad.
- Instrumentar la aplicación con OpenTelemetry para generar trazas distribuidas del flujo de cada request.
- Disparar alertas por email/Slack cuando se violan los SLOs definidos (latencia P95, tasa de pagos fallidos).

## 2. Por Qué Este Proyecto lo Necesita

Los flujos de pagos de PropConnect involucran llamadas a Stripe con webhooks asíncronos. Sin observabilidad, diagnosticar un pago que "desapareció" (entre el webhook de Stripe y la activación del boost) sería extremadamente difícil. Las trazas de OpenTelemetry permiten seguir exactamente qué módulo procesó el webhook y qué ocurrió después.

## 3. Elección de Tecnología

| Dimensión | Prometheus+Grafana+Loki (elegido) | Datadog | ELK Stack |
|---|---|---|---|
| **Gestionado / Self-hosted** | Self-hosted | Totalmente gestionado | Self-hosted |
| **Complejidad operacional** | Media | Baja | Alta (Elasticsearch para logs es costoso) |
| **Costo a nuestra escala** | Gratuito | $15–$31/host/mes | Gratuito (self-hosted, pero alto en recursos) |
| **Característica diferenciadora** | Estándar open source, sin vendor lock-in, integración nativa con NestJS | APM listo, dashboards automáticos, soporte | ELK maduro pero sobredimensionado para logs simples |

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| Sin costo de licencia — escala con el proyecto | Requiere configuración inicial de dashboards y alertas |
| OpenTelemetry es vendor-neutral — portable a cualquier backend | Si el servidor de observabilidad cae, se pierden logs en tiempo real |
| Loki es mucho más ligero que Elasticsearch para logs | Curva de aprendizaje para PromQL y LogQL |
| Grafana unifica métricas y logs en una sola UI | |

## 5. Integración

Prometheus hace scraping del endpoint `/metrics` expuesto por el monolito. Los logs JSON del proceso Node.js los recoge Loki mediante un agente `promtail` en el host. Las trazas de OpenTelemetry se envían a Grafana Tempo. Todo se visualiza en Grafana con dashboards preconfigurados. En la fase inicial, los tres corren en el mismo servidor (ver `04-deployment.md`). La migración a Grafana Cloud está planificada cuando el volumen de logs supere 1GB/día.
