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
