# 04 — Flujos de Datos e Interacciones

Este documento describe los flujos end-to-end más importantes del sistema PropConnect. Para cada flujo se incluye el escenario de negocio, diagrama de secuencia, camino feliz y al menos un camino de fallo o compensación.

---

## Flujo 1 — Registro y Verificación de Usuario

### Descripción del Escenario

Un nuevo usuario se registra en la plataforma como Consultor. El sistema valida los datos, crea la cuenta, asigna el rol correspondiente y envía un email de bienvenida.

### Diagrama de Secuencia

```mermaid
sequenceDiagram
    actor U as Usuario
    participant GW as API Gateway
    participant IAM as IAM Module
    participant DB as Database
    participant EB as Event Bus
    participant NTF as Notifications Module
    participant Email as SendGrid

    U->>GW: POST /api/v1/auth/register {email, password, role: CONSULTANT}
    GW->>IAM: RegisterUser(command)

    IAM->>DB: SELECT * FROM iam_users WHERE email = ?
    alt Email ya existe
        DB-->>IAM: user found
        IAM-->>GW: 409 Conflict — email already registered
        GW-->>U: 409 Conflict
    else Email disponible
        DB-->>IAM: no rows
        IAM->>DB: INSERT INTO iam_users (...)
        IAM->>DB: INSERT INTO iam_user_roles (userId, role: CONSULTANT)
        DB-->>IAM: OK
        IAM->>EB: publish(UserRegistered { userId, email, role })
        IAM-->>GW: 201 Created { userId, token }
        GW-->>U: 201 Created { userId, token }

        EB-->>NTF: UserRegistered event received
        NTF->>Email: send(template: WELCOME_EMAIL, to: email)
        Email-->>NTF: 200 OK
    end
```

### Camino Feliz

1. Usuario envía email, contraseña y rol deseado.
2. IAM verifica que el email no exista.
3. Se crea el usuario y se asigna el rol.
4. Se publica el evento `UserRegistered` en el EventBus.
5. Se retorna un JWT con el userId y rol.
6. El módulo de Notifications recibe el evento y envía el email de bienvenida de forma asíncrona.

### Camino de Fallo

- **Email duplicado**: IAM retorna `409 Conflict` antes de insertar en la base de datos. No se publican eventos.
- **Error de base de datos al insertar**: IAM retorna `500 Internal Server Error`. Al no haberse insertado el registro, no se publica el evento y no hay inconsistencia.
- **Fallo en SendGrid**: Como la notificación es asíncrona y post-evento, un fallo en el envío de email no afecta el registro. El módulo de Notifications puede reintentar con backoff exponencial.

---

## Flujo 2 — Publicación de Inmueble con Boost Pagado

### Descripción del Escenario

Un Vendedor crea una publicación de inmueble y decide pagar un boost "Destacado" para tener mayor visibilidad. El sistema procesa el pago con Stripe y activa el boost al confirmarse el cobro.

### Diagrama de Secuencia

```mermaid
sequenceDiagram
    actor V as Vendedor
    participant GW as API Gateway
    participant LST as Listings Module
    participant PAY as Payments Module
    participant Stripe as Stripe API
    participant EB as Event Bus
    participant NTF as Notifications Module

    V->>GW: POST /api/v1/listings {title, price, type, ...}
    GW->>LST: CreateListing(command)
    LST->>LST: Validate + persist listing (status: DRAFT)
    LST-->>GW: 201 { listingId }
    GW-->>V: 201 { listingId }

    V->>GW: POST /api/v1/payments/boost {listingId, boostType: FEATURED}
    GW->>PAY: InitiatePayment(listingId, BOOST)
    PAY->>Stripe: POST /payment_intents {amount, currency, metadata}
    Stripe-->>PAY: { clientSecret, paymentIntentId }
    PAY-->>GW: 200 { clientSecret }
    GW-->>V: 200 { clientSecret }

    V->>Stripe: Confirmar pago con tarjeta (client-side)
    Stripe->>GW: POST /api/v1/payments/webhook {event: payment_intent.succeeded}
    GW->>PAY: ProcessWebhook(event)
    PAY->>PAY: Persist Payment (status: COMPLETED)
    PAY->>PAY: Generate Invoice
    PAY->>EB: publish(PaymentCompleted { paymentId, listingId, type: BOOST })
    PAY->>EB: publish(InvoiceGenerated { invoiceId, userId })

    EB-->>LST: PaymentCompleted received
    LST->>LST: Create ListingBoost + set listing status: PUBLISHED
    LST->>EB: publish(ListingBoosted { listingId })
    LST->>EB: publish(ListingPublished { listingId })

    EB-->>NTF: InvoiceGenerated received
    NTF->>V: Email con factura adjunta
```

### Camino Feliz

1. Vendedor crea la publicación (queda en estado `DRAFT`).
2. Vendedor solicita un boost; el sistema crea un PaymentIntent en Stripe y retorna el `clientSecret`.
3. El cliente (frontend) confirma el pago directamente con Stripe.
4. Stripe envía un webhook `payment_intent.succeeded` a PropConnect.
5. Payments procesa el webhook, persiste el pago y publica `PaymentCompleted`.
6. Listings recibe el evento, activa el boost y publica la publicación.
7. Notifications envía la factura al vendedor por email.

### Camino de Fallo — Pago Rechazado

```mermaid
sequenceDiagram
    actor V as Vendedor
    participant Stripe as Stripe API
    participant GW as API Gateway
    participant PAY as Payments Module
    participant EB as Event Bus
    participant NTF as Notifications Module

    V->>Stripe: Confirmar pago (tarjeta rechazada)
    Stripe->>GW: POST /webhook {event: payment_intent.payment_failed}
    GW->>PAY: ProcessWebhook(event)
    PAY->>PAY: Persist Payment (status: FAILED)
    PAY->>EB: publish(PaymentFailed { paymentId, reason: "insufficient_funds" })

    EB-->>NTF: PaymentFailed received
    NTF->>V: Email notificando fallo de pago con enlace para reintentar

    Note over V,NTF: La publicación permanece en estado DRAFT
    Note over V,NTF: El vendedor puede intentar con otra tarjeta
```

---

## Flujo 3 — Firma de Contrato con Asesor

### Descripción del Escenario

Un Asesor aplica a una publicación activa. El Vendedor acepta la solicitud, se firma el contrato y el Asesor obtiene acceso completo a los datos del inmueble.

### Diagrama de Secuencia

```mermaid
sequenceDiagram
    actor A as Asesor
    actor VEN as Vendedor
    participant GW as API Gateway
    participant CTR as Contracts Module
    participant LST as Listings Module
    participant EB as Event Bus
    participant NTF as Notifications Module

    A->>GW: POST /api/v1/contracts/apply {listingId, message}
    GW->>CTR: CreateApplication(advisorId, listingId, message)
    CTR->>LST: isListingActive(listingId)
    LST-->>CTR: true
    CTR->>CTR: Persist ContractApplication (status: PENDING)
    CTR->>EB: publish(ContractApplicationReceived { applicationId, listingId, advisorId })
    EB-->>NTF: notificar al Vendedor
    CTR-->>GW: 201 { applicationId }
    GW-->>A: 201 { applicationId }

    VEN->>GW: PATCH /api/v1/contracts/applications/:id/accept
    GW->>CTR: AcceptApplication(applicationId, sellerId)
    CTR->>CTR: Validate seller owns listing
    CTR->>CTR: Create Contract (status: ACTIVE)
    CTR->>CTR: Create AccessGrant
    CTR->>EB: publish(ContractSigned { contractId, advisorId, listingId })
    CTR-->>GW: 200 { contractId }
    GW-->>VEN: 200 { contractId }

    EB-->>NTF: ContractSigned → notificar al Asesor
    NTF->>A: Email "Tu contrato fue aceptado"

    A->>GW: GET /api/v1/listings/:listingId/full-details
    GW->>LST: GetFullListingDetails(listingId, requesterId: A.id)
    LST->>CTR: hasActiveContract(advisorId, listingId)
    CTR-->>LST: true
    LST-->>GW: 200 { fullDetails including private contacts }
    GW-->>A: 200 { fullDetails }
```

### Camino Feliz

1. El Asesor aplica a una publicación activa.
2. Se notifica al Vendedor.
3. El Vendedor acepta la aplicación.
4. Se crea el Contrato en estado `ACTIVE` y el `AccessGrant`.
5. Se notifica al Asesor.
6. El Asesor puede ahora acceder a los detalles completos del inmueble.

### Camino de Fallo — Aplicación a Publicación Inactiva

Si el Vendedor pausó o cerró la publicación antes de que el Asesor aplique, `CTR` consulta `LST.isListingActive()` y recibe `false`. El sistema retorna `422 Unprocessable Entity` con el mensaje `"La publicación no está disponible para contratos"`. No se crea ninguna aplicación ni se envían notificaciones.

### Camino de Fallo — Acceso Sin Contrato

Si un Asesor intenta acceder a `/listings/:id/full-details` sin un contrato activo, `CTR.hasActiveContract()` retorna `false` y el endpoint retorna `403 Forbidden`. Este control se ejecuta siempre, independientemente del rol del usuario.

---

## Flujo 4 — Consulta con Asistente IA

### Descripción del Escenario

Un Consultor inicia una sesión de IA para pedir recomendaciones de inmuebles basadas en sus preferencias (presupuesto, tipo, zona).

### Diagrama de Secuencia

```mermaid
sequenceDiagram
    actor C as Consultor
    participant GW as API Gateway
    participant AIC as AI Consultant Module
    participant LST as Listings Module
    participant OpenAI as OpenAI API

    C->>GW: POST /api/v1/ai/session
    GW->>AIC: StartSession(userId)
    AIC->>AIC: Persist AISession
    AIC-->>GW: 201 { sessionId }
    GW-->>C: 201 { sessionId }

    C->>GW: POST /api/v1/ai/session/:id/message {content: "Busco apartamento en zona 10, max $300k"}
    GW->>AIC: SendMessage(sessionId, content)
    AIC->>LST: SearchListings(filters: { type: SALE, location: "zona 10", maxPrice: 300000 })
    LST-->>AIC: [listing1, listing2, listing3]
    AIC->>OpenAI: POST /chat/completions {messages: [...history, listings_context, user_message]}
    OpenAI-->>AIC: { content: "Basado en tus preferencias, te recomiendo..." }
    AIC->>AIC: Persist AIMessage (user + assistant)
    AIC-->>GW: 200 { reply, recommendedListings: [id1, id2, id3] }
    GW-->>C: 200 { reply, recommendedListings }
```

### Camino de Fallo — OpenAI No Disponible

Si la llamada a OpenAI falla (timeout o error 5xx), el módulo `ai-consultant` retorna un mensaje de error genérico al usuario: `"El asistente no está disponible en este momento. Intenta de nuevo en unos segundos."` con código `503 Service Unavailable`. Los mensajes previos de la sesión se persisten correctamente. El usuario puede reintentar.
