# 05 — Proveedor de Autenticación (Auth Propio — NestJS + JWT)

## 1. Responsabilidades

- Registrar usuarios y gestionar credenciales con `bcrypt` (factor de costo 12).
- Emitir access tokens JWT firmados con RS256 (duración 15 min) y refresh tokens opacos (duración 30 días).
- Validar tokens en cada request entrante a través de `JwtAuthGuard`.
- Implementar Refresh Token Rotation: invalidar el token anterior en cada renovación para detectar reutilización.
- Gestionar la suspensión de cuentas con invalidación inmediata de todos los refresh tokens activos.

## 2. Por Qué Este Proyecto lo Necesita

PropConnect necesita controlar el acceso por rol Y por contexto de recurso (ej. un Asesor solo accede a un inmueble si tiene contrato activo). Los proveedores gestionados como Auth0 o Cognito gestionan la autenticación, pero la lógica de autorización contextual siempre vivirá en el código de PropConnect. Dado que el equipo debe implementar esa capa de todas formas, el valor de delegar solo el JWT a un tercero no justifica el costo ni el vendor lock-in.

## 3. Elección de Tecnología

| Dimensión | Auth Propio JWT (elegido) | Auth0 | AWS Cognito |
|---|---|---|---|
| **Gestionado / Self-hosted** | Self-managed | Totalmente gestionado | Totalmente gestionado |
| **Complejidad operacional** | Media (responsabilidad de seguridad propia) | Baja | Baja-media |
| **Costo a nuestra escala** | Gratuito | $0–$23/mes (hasta 7500 MAU gratis) | $0–variable por MAU |
| **Característica diferenciadora** | Control total, sin vendor lock-in, ABAC contextual nativo | MFA, social login, compliance listo | Integración nativa con AWS |

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| Sin costo variable por usuario activo | Responsabilidad de seguridad en contraseñas y tokens recae en el equipo |
| ABAC contextual implementable directamente en el código | MFA y social login requieren desarrollo adicional (no incluidos en fase inicial) |
| Control total sobre los claims del JWT | Requiere auditorías de seguridad periódicas |
| Sin vendor lock-in | |

## 5. Integración

El módulo `iam` gestiona todo el ciclo de autenticación. `JwtAuthGuard` y `RolesGuard` se aplican globalmente en la capa `api/v1`. Los refresh tokens se almacenan en la tabla `iam_sessions` de PostgreSQL. Ver ADR-005 y `04-deployment.md`.

---

# 06 — Sistema de Notificaciones (SendGrid + Firebase FCM)

## 1. Responsabilidades

- Entregar emails transaccionales a través de SendGrid: bienvenida, confirmación de cita, contrato firmado, factura, pago fallido.
- Entregar notificaciones push a través de Firebase Cloud Messaging para eventos urgentes (nueva aplicación de asesor, cita confirmada).
- Renderizar templates con variables dinámicas (nombre del usuario, detalles del inmueble, monto de factura).
- Respetar las preferencias de notificación por usuario y canal (`ntf_preferences`).
- Registrar el resultado de cada envío (éxito/fallo) para auditoría y reintentos.

## 2. Por Qué Este Proyecto lo Necesita

PropConnect tiene múltiples actores que necesitan mantenerse informados del estado de sus operaciones sin estar activamente en la plataforma. Un asesor necesita saber que su contrato fue aceptado. Un vendedor necesita saber que su pago falló. Sin notificaciones, los usuarios tendrían que refrescar manualmente la plataforma para enterarse de cambios críticos, lo que degradaría gravemente la experiencia.

## 3. Elección de Tecnología

| Dimensión | SendGrid + FCM (elegido) | AWS SES + SNS | Postmark + OneSignal |
|---|---|---|---|
| **Gestionado / Self-hosted** | Totalmente gestionados | Totalmente gestionados | Totalmente gestionados |
| **Complejidad operacional** | Baja | Baja-media (configuración de dominio en AWS) | Baja |
| **Costo a nuestra escala** | SendGrid: gratis hasta 100 emails/día; FCM: gratis | SES: $0.10/1000 emails | Postmark: $15/mes; OneSignal: gratis hasta 10k suscriptores |
| **Característica diferenciadora** | SendGrid: alta deliverability, templates visuales; FCM: push estándar en Android/iOS | SES: integración nativa AWS | Postmark: mejor deliverability transaccional |

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| Ambos servicios son líderes de mercado con alta deliverability | Dependencia de dos proveedores externos distintos |
| SDKs oficiales para Node.js bien mantenidos | FCM requiere registro de la app móvil (relevante cuando se lance la app) |
| FCM es gratuito sin límite de mensajes push | SendGrid tiene límites en el plan gratuito (100 emails/día) — se necesita plan pagado en producción |

## 5. Integración

El módulo `notifications` consume eventos del EventBus y llama a SendGrid o FCM según el canal configurado en `ntf_preferences` del usuario. No hay llamadas síncronas desde otros módulos — todo es reactivo a eventos. Ver `02-bounded-contexts.md` (BC-7) y `04-deployment.md`.

---

# 07 — Almacenamiento de Objetos (AWS S3)

## 1. Responsabilidades

- Almacenar las fotos de los inmuebles publicadas por los vendedores.
- Generar URLs prefirmadas (presigned URLs) para que los clientes suban imágenes directamente sin pasar por el servidor de la aplicación.
- Almacenar los PDFs de facturas generados por el módulo de Payments.
- Gestionar el ciclo de vida de archivos: eliminar imágenes cuando una publicación es eliminada (S3 Lifecycle Policy).
- Servir imágenes a través de CloudFront CDN para distribución geográfica eficiente.

## 2. Por Qué Este Proyecto lo Necesita

Las publicaciones de inmuebles en PropConnect requieren fotos de alta calidad (múltiples imágenes por propiedad). Almacenarlas en el servidor de aplicación o en PostgreSQL sería un antipatrón: saturarían el disco del servidor, bloquearían el proceso Node.js durante uploads, y harían los backups innecesariamente grandes. S3 está diseñado específicamente para este tipo de almacenamiento binario de gran volumen.

## 3. Elección de Tecnología

| Dimensión | AWS S3 (elegido) | Google Cloud Storage | MinIO (self-hosted) |
|---|---|---|---|
| **Gestionado / Self-hosted** | Totalmente gestionado | Totalmente gestionado | Self-hosted |
| **Complejidad operacional** | Baja | Baja | Media (administrar servidor de almacenamiento) |
| **Costo a nuestra escala** | ~$0.023/GB/mes + $0.09/GB transferencia | Similar a S3 | Solo costo de servidor |
| **Característica diferenciadora** | Integración nativa con CloudFront, IAM de AWS, SDK maduro | Mejor integración con GCP | Sin costo de almacenamiento, control total |

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| Presigned URLs evitan que el servidor sea cuello de botella en uploads | Costo variable según volumen almacenado y transferencia |
| Integración nativa con CloudFront para CDN | Vendor lock-in con AWS (mitigado por la API S3-compatible de alternativas) |
| Durabilidad 99.999999999% (11 nueves) | Requiere configurar correctamente las políticas IAM para seguridad |
| Lifecycle policies automáticas para limpiar archivos huérfanos | |

## 5. Integración

El módulo `listings` genera presigned URLs para uploads directos desde el frontend. Las URLs resultantes se almacenan en `lst_properties.mediaUrls`. El módulo `payments` sube los PDFs de facturas a S3 y guarda la URL en `pay_invoices.downloadUrl`. Las imágenes se sirven a través de CloudFront, no directamente desde S3. Ver `04-deployment.md`.

---

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
