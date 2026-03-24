# ADR-005: Autenticación y Autorización

| Campo | Valor |
|---|---|
| **Status** | Aceptado |
| **Fecha** | 24/03/2026 |
| **Decisores** | Douglas Daniel Pérez Hernández |

## Contexto

PropConnect tiene 5 roles distintos (Consultor, Vendedor, Asesor, Tramitador, Administrador) con permisos muy diferentes. Un Asesor solo puede acceder a detalles completos de un inmueble si tiene contrato activo con ese inmueble específico — lo que significa que la autorización no es solo por rol, sino también por contexto de recurso. Se necesita una estrategia que sea segura, mantenible y que soporte este nivel de granularidad.

## Opciones Consideradas

### Opción 1: Proveedor de Identidad Administrado (Auth0 / AWS Cognito)

| Pros | Cons |
|---|---|
| Gestión de usuarios, refresh tokens, MFA y social login listos | Costo mensual proporcional al número de usuarios |
| Cumplimiento de estándares (OAuth2, OIDC) gestionado por el proveedor | Vendor lock-in: migrar implica reescribir la capa de auth |
| Reducción de superficie de ataque en la gestión de contraseñas | El modelo de roles de Auth0 no se adapta bien a permisos por recurso |
| | Requiere integración con el propio sistema para permisos contextuales de PropConnect |

### Opción 2: Auth Propio con JWT (Self-managed)

| Pros | Cons |
|---|---|
| Control total sobre el modelo de roles y permisos | Responsabilidad de gestionar bcrypt, rotación de tokens y seguridad |
| Permisos contextuales (hasActiveContract) manejados directamente en el código | Requiere implementar refresh token rotation, revocación y blacklisting |
| Sin costo variable por usuario activo | Mayor superficie de código a mantener y auditar |
| El modelo de datos de IAM está diseñado exactamente para las necesidades del negocio | |

## Decisión

Se adopta **Auth propio con JWT stateless** para autenticación, combinado con **RBAC (Role-Based Access Control) + ABAC contextual** para autorización.

**Justificación específica al proyecto:**

PropConnect tiene un requisito de autorización que los proveedores gestionados no resuelven nativamente: "un Asesor puede ver los datos completos de un inmueble solo si tiene contrato activo con ese inmueble específico." Esto es ABAC (Attribute-Based Access Control) contextual que depende del estado de la base de datos en tiempo real. Auth0 o Cognito pueden gestionar el JWT, pero la lógica de autorización siempre vivirá en el código de PropConnect. Dado que el equipo ya debe implementar esa capa, el valor agregado del proveedor externo se reduce.

El modelo de JWT propio permite controlar exactamente qué claims se incluyen en el token (userId, roles activos) y mantener el costo fijo independientemente del número de usuarios.

### Modelo de Roles y Permisos

| Rol | Capacidades Clave |
|---|---|
| `CONSULTANT` | Ver publicaciones, contactar profesionales, agendar citas, usar IA, dar reseñas |
| `SELLER` | CRUD de inmuebles, contratar asesores/tramitadores, ver historial, pagar boosts |
| `ADVISOR` | Ver contratos vigentes, acceder datos completos de inmuebles con contrato activo, agendar citas |
| `PROCESSOR` | Igual que ADVISOR — orientado a procesos legales |
| `ADMIN` | Acceso total: suspender usuarios, resolver quejas, ver métricas globales |

> Un usuario puede tener múltiples roles simultáneamente (ej. alguien que es Vendedor y también Consultor).

### Flujo de Autorización Contextual

```
Request: GET /listings/:id/full-details
  ↓
JwtAuthGuard — verifica firma del token y extrae { userId, roles }
  ↓
RolesGuard — verifica que el usuario tenga rol ADVISOR o PROCESSOR
  ↓
ListingAccessGuard — llama a Contracts.hasActiveContract(userId, listingId)
  ↓ true → 200 OK con datos completos
  ↓ false → 403 Forbidden
```

### Gestión de Tokens

- **Access Token**: JWT firmado con RS256, duración 15 minutos.
- **Refresh Token**: Opaco, almacenado en base de datos (`iam_sessions`), rotación en cada uso (Refresh Token Rotation), duración 30 días.
- **Revocación**: Los refresh tokens se invalidan al hacer logout o al suspender un usuario. Los access tokens son stateless y expiran naturalmente en 15 minutos.

## Consecuencias

**Habilita:**
- Modelo de autorización contextual (ABAC) que refleja exactamente las reglas de negocio de PropConnect.
- Sin costo variable por número de usuarios.
- Control total sobre la estructura del token y los claims.

**Previene:**
- No se implementa MFA ni social login en la fase inicial. Ambos pueden agregarse posteriormente sin cambiar la arquitectura de autorización.

**Riesgos y Mitigaciones:**
- *Vulnerabilidades en gestión de contraseñas*: Se usa `bcrypt` con factor de costo 12. Las contraseñas nunca se almacenan en texto plano.
- *Token robado*: El access token de 15 minutos limita la ventana de exposición. El refresh token rotation invalida el token anterior en cada uso, detectando reutilización.

## ADRs Relacionados

- ADR-001 (Modelo de Despliegue) — contexto del módulo IAM dentro del monolito.
- ADR-007 (Diseño de API) — los guards de autenticación aplican a todos los endpoints REST.
