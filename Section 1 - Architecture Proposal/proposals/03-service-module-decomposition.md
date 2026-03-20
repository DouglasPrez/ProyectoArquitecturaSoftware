# 03 — Descomposición en Módulos y Servicios

## Arquitectura Elegida: Monolito Modular

En base a la decisión del documento `01-high-level-architecture.md`, la organización del código refleja un monolito modular donde cada bounded context es un módulo independiente con sus propias capas internas. Los módulos se comunican entre sí únicamente a través de sus interfaces públicas (`PublicAPI`), nunca mediante importaciones cruzadas de capas internas.

---

## Estructura del Repositorio

```
propconnect/
├── README.md
├── package.json
├── tsconfig.json
├── docker-compose.yml
│
├── src/
│   ├── main.ts                          # Entry point — bootstrap de NestJS
│   ├── app.module.ts                    # Módulo raíz — importa todos los módulos
│   │
│   ├── shared/                          # Código compartido entre módulos
│   │   ├── decorators/                  # @CurrentUser, @Roles, etc.
│   │   ├── guards/                      # JwtAuthGuard, RolesGuard
│   │   ├── events/                      # EventBus interno (EventEmitter2)
│   │   ├── types/                       # Tipos e interfaces globales
│   │   └── utils/                       # Helpers genéricos
│   │
│   ├── modules/
│   │   ├── iam/                         # BC: Identity & Access
│   │   │   ├── iam.module.ts
│   │   │   ├── public-api.ts            # ← ÚNICA interfaz visible externamente
│   │   │   ├── application/
│   │   │   │   ├── commands/            # RegisterUser, AssignRole, SuspendUser
│   │   │   │   └── queries/             # GetUserById, ValidateToken
│   │   │   ├── domain/
│   │   │   │   ├── entities/            # User, Role, Session
│   │   │   │   └── events/              # UserRegistered, UserSuspended
│   │   │   └── infrastructure/
│   │   │       ├── persistence/         # UserRepository (TypeORM)
│   │   │       └── auth/                # JwtStrategy, BcryptService
│   │   │
│   │   ├── listings/                    # BC: Listings
│   │   │   ├── listings.module.ts
│   │   │   ├── public-api.ts
│   │   │   ├── application/
│   │   │   │   ├── commands/            # CreateListing, UpdateListing, BoostListing
│   │   │   │   └── queries/             # SearchListings, GetListingById
│   │   │   ├── domain/
│   │   │   │   ├── entities/            # Listing, Property, ListingBoost, ListingView
│   │   │   │   └── events/              # ListingPublished, ListingBoosted, ListingSoldOrRented
│   │   │   └── infrastructure/
│   │   │       ├── persistence/         # ListingRepository
│   │   │       └── search/              # ElasticsearchAdapter (búsqueda por filtros)
│   │   │
│   │   ├── contracts/                   # BC: Contracts
│   │   │   ├── contracts.module.ts
│   │   │   ├── public-api.ts
│   │   │   ├── application/
│   │   │   │   ├── commands/            # CreateContract, SignContract, CancelContract
│   │   │   │   └── queries/             # GetContractsByListing, CheckAccessGrant
│   │   │   ├── domain/
│   │   │   │   ├── entities/            # Contract, ContractApplication, AccessGrant
│   │   │   │   └── events/              # ContractSigned, ContractCompleted, ContractCancelled
│   │   │   └── infrastructure/
│   │   │       └── persistence/         # ContractRepository
│   │   │
│   │   ├── payments/                    # BC: Payments
│   │   │   ├── payments.module.ts
│   │   │   ├── public-api.ts
│   │   │   ├── application/
│   │   │   │   ├── commands/            # InitiatePayment, ProcessWebhook, RefundPayment
│   │   │   │   └── queries/             # GetInvoices, GetPaymentHistory
│   │   │   ├── domain/
│   │   │   │   ├── entities/            # Payment, Invoice, PaymentMethod
│   │   │   │   └── events/              # PaymentCompleted, PaymentFailed, InvoiceGenerated
│   │   │   └── infrastructure/
│   │   │       ├── persistence/         # PaymentRepository
│   │   │       └── stripe/              # StripeAdapter (Anti-Corruption Layer)
│   │   │
│   │   ├── appointments/                # BC: Appointments
│   │   │   ├── appointments.module.ts
│   │   │   ├── public-api.ts
│   │   │   ├── application/
│   │   │   │   ├── commands/            # ScheduleAppointment, ConfirmAppointment, CancelAppointment
│   │   │   │   └── queries/             # GetAppointmentsByUser, GetAvailability
│   │   │   ├── domain/
│   │   │   │   ├── entities/            # Appointment, Availability, AppointmentReminder
│   │   │   │   └── events/              # AppointmentScheduled, AppointmentConfirmed, AppointmentCancelled
│   │   │   └── infrastructure/
│   │   │       └── persistence/         # AppointmentRepository
│   │   │
│   │   ├── reviews/                     # BC: Reviews & Ratings
│   │   │   ├── reviews.module.ts
│   │   │   ├── public-api.ts
│   │   │   ├── application/
│   │   │   │   ├── commands/            # SubmitReview, FileComplaint
│   │   │   │   └── queries/             # GetReviewsByUser, GetRatingSummary
│   │   │   ├── domain/
│   │   │   │   ├── entities/            # Review, RatingSummary, Complaint
│   │   │   │   └── events/              # ReviewSubmitted, ComplaintFiled
│   │   │   └── infrastructure/
│   │   │       └── persistence/         # ReviewRepository
│   │   │
│   │   ├── notifications/               # BC: Notifications
│   │   │   ├── notifications.module.ts
│   │   │   ├── public-api.ts
│   │   │   ├── application/
│   │   │   │   └── handlers/            # Handlers de cada evento del sistema
│   │   │   ├── domain/
│   │   │   │   └── entities/            # Notification, NotificationTemplate, NotificationPreference
│   │   │   └── infrastructure/
│   │   │       ├── email/               # SendGridAdapter
│   │   │       └── push/                # FirebaseAdapter
│   │   │
│   │   └── ai-consultant/               # BC: AI Consultant
│   │       ├── ai-consultant.module.ts
│   │       ├── public-api.ts
│   │       ├── application/
│   │       │   ├── commands/            # StartAISession, SendMessage
│   │       │   └── queries/             # GetRecommendations, EstimatePrice
│   │       ├── domain/
│   │       │   └── entities/            # AISession, AIMessage, RecommendationRequest
│   │       └── infrastructure/
│   │           └── openai/              # OpenAIAdapter
│   │
│   └── api/                             # Capa HTTP — controllers y DTOs
│       ├── v1/
│       │   ├── auth/                    # POST /auth/register, /auth/login
│       │   ├── listings/                # CRUD /listings, GET /listings/search
│       │   ├── contracts/               # POST /contracts, PATCH /contracts/:id
│       │   ├── payments/                # POST /payments, GET /payments/invoices
│       │   ├── appointments/            # CRUD /appointments
│       │   ├── reviews/                 # POST /reviews, POST /complaints
│       │   └── ai/                      # POST /ai/session, POST /ai/recommend
│       └── health/                      # GET /health (liveness & readiness)
│
├── test/
│   ├── unit/                            # Tests por módulo
│   ├── integration/                     # Tests de módulo completo
│   └── e2e/                             # Tests de flujo completo
│
└── infrastructure/
    ├── migrations/                      # Migraciones de TypeORM
    └── seeds/                           # Datos de desarrollo
```

---

## Descripción de Módulos

### `iam` — Identity & Access
- **Qué posee**: Registro, login, JWT, gestión de roles y suspensiones.
- **Qué expone** (`public-api.ts`): `IamPublicApi` con métodos `getUserById(id)`, `validateToken(token)`, `getUserRoles(id)`.
- **Bounded Context**: Identity & Access.

### `listings` — Publicaciones
- **Qué posee**: CRUD de inmuebles, motor de búsqueda con filtros (precio, tipo, ubicación, habitaciones), historial de vistas, boosts.
- **Qué expone**: `ListingsPublicApi` con `getListingById(id)`, `isListingActive(id)`, `getListingsBySeller(sellerId)`.
- **Bounded Context**: Listings.

### `contracts` — Contratos
- **Qué posee**: Creación y gestión de contratos entre vendedores y profesionales, control de acceso a datos completos del inmueble.
- **Qué expone**: `ContractsPublicApi` con `hasActiveContract(professionalId, listingId)`, `getContractById(id)`.
- **Bounded Context**: Contracts.

### `payments` — Pagos
- **Qué posee**: Integración con Stripe, procesamiento de webhooks, generación de facturas, historial de pagos.
- **Qué expone**: `PaymentsPublicApi` con `createPaymentIntent(amount, userId, type)`.
- **Bounded Context**: Payments.

### `appointments` — Citas
- **Qué posee**: Agendamiento, disponibilidad por usuario, recordatorios automáticos.
- **Qué expone**: `AppointmentsPublicApi` con `getUpcomingByUser(userId)`.
- **Bounded Context**: Appointments.

### `reviews` — Reseñas
- **Qué posee**: Calificaciones 1–5, comentarios, sistema de quejas, promedio recalculado.
- **Qué expone**: `ReviewsPublicApi` con `getRatingSummary(userId)`.
- **Bounded Context**: Reviews & Ratings.

### `notifications` — Notificaciones
- **Qué posee**: Templates, preferencias por canal, envío a través de SendGrid (email) y Firebase (push).
- **Qué expone**: `NotificationsPublicApi` con `sendNotification(userId, templateId, payload)` — pero en la práctica es un consumidor de eventos, no es invocado directamente por otros módulos.
- **Bounded Context**: Notifications.

### `ai-consultant` — Asistente IA
- **Qué posee**: Sesiones de chat, recomendaciones basadas en preferencias, estimación de precio de inmueble.
- **Qué expone**: `AIConsultantPublicApi` con `getRecommendations(userId, preferences)`.
- **Bounded Context**: AI Consultant.

---

## Enforcement de Límites de Contexto

### Regla de No-Cruce de Módulos

Ningún módulo puede importar directamente las capas `domain/`, `application/` o `infrastructure/` de otro módulo. Solo está permitido importar desde `public-api.ts`.

```typescript
// ❌ INCORRECTO — viola el boundary
import { ListingRepository } from '../listings/infrastructure/persistence/listing.repository';

// ✅ CORRECTO — usa la interfaz pública del módulo
import { ListingsPublicApi } from '../listings/public-api';
```

### Comunicación por Eventos

Para interacciones donde un módulo reacciona a cambios en otro (por ejemplo, Notifications escuchando `PaymentCompleted`), se usa un EventBus interno (`EventEmitter2` de NestJS). Esto desacopla a los módulos sin necesidad de llamadas directas.

```typescript
// En payments/domain/events
this.eventBus.publish(new PaymentCompletedEvent(payment));

// En notifications/application/handlers
@EventsHandler(PaymentCompletedEvent)
class OnPaymentCompleted implements IEventHandler<PaymentCompletedEvent> {
  handle(event: PaymentCompletedEvent) { /* enviar notificación */ }
}
```

### Acceso a Datos

Cada módulo tiene sus propias tablas en la base de datos compartida. La convención de nombrado usa el prefijo del módulo para evitar colisiones: `iam_users`, `iam_roles`, `listings_listings`, `payments_invoices`, etc. Ningún módulo ejecuta queries sobre tablas de otro módulo.

---

## API Gateway (Capa `src/api`)

La capa `api/v1` actúa como el punto de entrada HTTP. Los controllers no contienen lógica de negocio: únicamente delegan a los módulos correspondientes a través de sus `public-api.ts`. Esto permite que la capa HTTP sea delgada y fácilmente reemplazable.

El versionado `/v1` asegura que futuras versiones de la API no rompan clientes existentes.
