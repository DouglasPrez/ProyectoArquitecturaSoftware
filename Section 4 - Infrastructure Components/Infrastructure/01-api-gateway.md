# 01 — API Gateway (Nginx)

## 1. Responsabilidades

- Terminar conexiones HTTPS (SSL/TLS) y redirigir tráfico HTTP al backend NestJS en el puerto 3000.
- Servir los assets estáticos del frontend (build de React) directamente desde el filesystem, sin pasar por la aplicación.
- Actuar como balanceador de carga entre réplicas del contenedor `propconnect-api` cuando se escale horizontalmente.
- Limitar la tasa de requests por IP (`rate limiting`) para proteger el backend de abuso y ataques DDoS básicos.
- Reescribir headers de seguridad (`X-Frame-Options`, `X-Content-Type-Options`, `Strict-Transport-Security`).

## 2. Por Qué Este Proyecto lo Necesita

En PropConnect, el módulo de Listings puede recibir muchas solicitudes de búsqueda concurrentes de consultores (especialmente en horarios pico). Sin un reverse proxy, cada request de imagen o asset estático pasaría por NestJS, desperdiciando recursos del proceso Node.js. Nginx separa el tráfico estático del dinámico, reduciendo la carga del servidor de aplicación.

Además, el endpoint `POST /api/v1/payments/webhook` recibe tráfico de Stripe que debe autenticarse por firma. Nginx puede rechazar requests sin el header `Stripe-Signature` antes de que lleguen al backend.

## 3. Elección de Tecnología

| Dimensión | Nginx (elegido) | Caddy | HAProxy |
|---|---|---|---|
| **Gestionado / Self-hosted** | Self-hosted | Self-hosted | Self-hosted |
| **Complejidad operacional** | Baja — configuración declarativa bien documentada | Muy baja — HTTPS automático con Let's Encrypt | Media — orientado a L4/L7 avanzado |
| **Costo a nuestra escala** | Gratuito | Gratuito | Gratuito |
| **Característica diferenciadora** | Ampliamente usado, documentación extensa, módulos maduros | HTTPS automático sin configuración de certificados | Mejor rendimiento para L4, más complejo para HTTP |

Se elige Nginx por su madurez, amplio soporte en la comunidad y compatibilidad nativa con configuraciones de Docker y CI/CD. Caddy sería una alternativa válida para equipos que priorizan la simplicidad de HTTPS automático.

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| Rendimiento muy alto para contenido estático | La configuración requiere conocimiento de directivas Nginx |
| Rate limiting y headers de seguridad listos para producción | No es un API Gateway completo (sin auth, sin routing inteligente por servicio) |
| Escalado horizontal trivial con `upstream` en round-robin | Si Nginx cae, todo el tráfico se interrumpe (mitigado con health checks en el orquestador) |
| Sin costo adicional | Reinicio requerido para cambios de configuración (mitigado con `nginx -s reload`) |

## 5. Integración

Nginx es el único componente expuesto al Internet Público (puerto 443). Se conecta al contenedor `propconnect-api` en la red interna del VPC. Las imágenes de inmuebles se sirven directamente desde una URL de S3 (no pasan por Nginx). Ver diagrama `04-deployment.md` para la topología completa.
