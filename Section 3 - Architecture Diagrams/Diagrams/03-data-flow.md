# 03 — Diagramas de Flujo de Datos (Secuencias)

Este documento contiene los diagramas de secuencia de los flujos más importantes de PropConnect. Los flujos están divididos en partes cuando tienen más de 6 participantes para mantener la legibilidad.

---

## Flujo A — Pago de Boost (Parte 1: Iniciación del Pago)

Este flujo muestra cómo un Vendedor paga para destacar su publicación. La Parte 1 cubre desde la solicitud del boost hasta la entrega del `clientSecret` al cliente.

```mermaid
sequenceDiagram
    actor V as Vendedor
    participant GW as API Gateway
    participant PAY as Payments
    participant Stripe as Stripe API

    V->>GW: POST /api/v1/payments/boost {listingId, boostType: FEATURED}
    GW->>PAY: InitiatePayment(listingId, BOOST, userId)
    PAY->>PAY: Validar que listing existe y pertenece al vendedor
    PAY->>Stripe: POST /v1/payment_intents {amount, currency: "USD", metadata: {listingId}}
    
    alt Stripe responde OK
        Stripe-->>PAY: 200 {clientSecret, paymentIntentId}
        PAY->>PAY: Persist Payment (status: PENDING, externalId: paymentIntentId)
        PAY-->>GW: 200 {clientSecret}
        GW-->>V: 200 {clientSecret}
        Note over V,Stripe: El cliente confirma el pago con el clientSecret (Parte 2)
    else Stripe no disponible
        Stripe-->>PAY: 500 / timeout
        PAY-->>GW: 503 Service Unavailable
        GW-->>V: 503 "Servicio de pagos no disponible. Intenta en unos minutos."
    end
```

---

## Flujo A — Pago de Boost (Parte 2: Confirmación vía Webhook)

La Parte 2 cubre el procesamiento del webhook de Stripe y la activación del boost.

```mermaid
sequenceDiagram
    actor V as Vendedor (browser)
    participant Stripe as Stripe API
    participant GW as API Gateway
    participant PAY as Payments
    participant EB as Event Bus
    participant LST as Listings
    participant NTF as Notifications

    V->>Stripe: Confirmar pago con tarjeta (Stripe.js client-side)
    
    alt Pago aprobado
        Stripe->>GW: POST /api/v1/payments/webhook {event: payment_intent.succeeded, paymentIntentId}
        GW->>PAY: ProcessWebhook(event)
        PAY->>PAY: Verify Stripe signature
        PAY->>PAY: Update Payment status → COMPLETED
        PAY->>PAY: Generate Invoice
        PAY->>EB: publish(PaymentCompleted {paymentId, listingId, type: BOOST})
        PAY->>EB: publish(InvoiceGenerated {invoiceId, userId})
        PAY-->>GW: 200 OK
        EB-->>LST: PaymentCompleted received
        LST->>LST: Create ListingBoost + update listing status → PUBLISHED
        LST->>EB: publish(ListingPublished {listingId})
        EB-->>NTF: InvoiceGenerated received
        NTF->>V: Email con factura PDF adjunta

    else Pago rechazado
        Stripe->>GW: POST /webhook {event: payment_intent.payment_failed, reason: "insufficient_funds"}
        GW->>PAY: ProcessWebhook(event)
        PAY->>PAY: Update Payment status → FAILED
        PAY->>EB: publish(PaymentFailed {paymentId, reason})
        EB-->>NTF: PaymentFailed received
        NTF->>V: Email "Tu pago no pudo procesarse. Intenta con otra tarjeta."
        Note over LST: Listing permanece en DRAFT
    end
```

---

## Flujo B — Firma de Contrato con Asesor (Camino Completo)

Este flujo muestra el ciclo completo: aplicación del asesor → aceptación del vendedor → acceso a datos completos.

```mermaid
sequenceDiagram
    actor A as Asesor
    actor VEN as Vendedor
    participant GW as API Gateway
    participant CTR as Contracts
    participant LST as Listings
    participant EB as Event Bus
    participant NTF as Notifications

    A->>GW: POST /api/v1/contracts/apply {listingId, message: "Tengo experiencia en zona 10"}
    GW->>CTR: CreateApplication(advisorId, listingId, message)
    CTR->>LST: isListingActive(listingId)
    
    alt Listing inactivo
        LST-->>CTR: false
        CTR-->>GW: 422 Unprocessable Entity
        GW-->>A: 422 "La publicación no está disponible para contratos"
    else Listing activo
        LST-->>CTR: true
        CTR->>CTR: Persist ContractApplication (status: PENDING)
        CTR->>EB: publish(ContractApplicationReceived {applicationId, listingId, advisorId})
        EB-->>NTF: notificar al vendedor
        NTF->>VEN: Push + Email "Nuevo asesor aplicó a tu propiedad"
        CTR-->>GW: 201 {applicationId}
        GW-->>A: 201 {applicationId}
    end

    VEN->>GW: PATCH /api/v1/contracts/applications/:id/accept
    GW->>CTR: AcceptApplication(applicationId, sellerId)
    CTR->>CTR: Validar que el vendedor es dueño del listing del contrato
    CTR->>CTR: Create Contract (status: ACTIVE) + Create AccessGrant
    CTR->>EB: publish(ContractSigned {contractId, advisorId, listingId, sellerId})
    CTR-->>GW: 200 {contractId}
    GW-->>VEN: 200 {contractId}
    EB-->>NTF: ContractSigned received
    NTF->>A: Email "Tu contrato fue aceptado. Ya tienes acceso completo al inmueble."

    A->>GW: GET /api/v1/listings/:listingId/full-details
    GW->>LST: GetFullDetails(listingId, requesterId: advisorId)
    LST->>CTR: hasActiveContract(advisorId, listingId)
    
    alt Sin contrato activo
        CTR-->>LST: false
        LST-->>GW: 403 Forbidden
        GW-->>A: 403 "No tienes acceso a los detalles completos de esta publicación"
    else Con contrato activo
        CTR-->>LST: true
        LST-->>GW: 200 {fullDetails: {address, ownerPhone, cadastralInfo, ...}}
        GW-->>A: 200 {fullDetails}
    end
```

---

## Flujo C — Consulta con Asistente IA (Con Fallo de OpenAI)

```mermaid
sequenceDiagram
    actor C as Consultor
    participant GW as API Gateway
    participant AIC as AI Consultant
    participant LST as Listings
    participant OpenAI as OpenAI API

    C->>GW: POST /api/v1/ai/messages {sessionId, content: "Busco casa en zona 14, 3 habitaciones, max $250k"}
    GW->>AIC: SendMessage(sessionId, userId, content)
    AIC->>AIC: Load session history from DB
    AIC->>LST: SearchListings({zone: "14", rooms: 3, maxPrice: 250000, type: SALE})
    LST-->>AIC: [{id, title, price, address, rooms}] (máx 5 resultados)
    AIC->>AIC: Build prompt con historial + contexto de listings
    AIC->>OpenAI: POST /v1/chat/completions {model, messages, max_tokens: 1000}

    alt OpenAI disponible
        OpenAI-->>AIC: 200 {content: "Encontré 3 propiedades que coinciden con tus criterios..."}
        AIC->>AIC: Persist AIMessage (user) + AIMessage (assistant)
        AIC-->>GW: 200 {reply, recommendedListingIds: [id1, id2, id3]}
        GW-->>C: 200 {reply, recommendedListingIds}
    else OpenAI timeout o error 5xx
        OpenAI-->>AIC: timeout / 503
        AIC->>AIC: Persist AIMessage (user) con error flag
        AIC-->>GW: 503 {error: "assistant_unavailable"}
        GW-->>C: 503 "El asistente no está disponible en este momento. Tus datos fueron guardados, intenta de nuevo."
    end
```
