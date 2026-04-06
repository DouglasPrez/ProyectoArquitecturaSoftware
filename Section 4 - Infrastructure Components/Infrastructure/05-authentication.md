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
