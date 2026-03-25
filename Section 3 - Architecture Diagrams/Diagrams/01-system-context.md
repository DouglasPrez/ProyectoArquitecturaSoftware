# 01 — Diagrama de Contexto del Sistema (C4 Nivel 1)

## Descripción

Este diagrama muestra PropConnect como una única caja de sistema, rodeada de todos sus actores humanos y sistemas externos. No se muestran componentes internos — el objetivo es comunicar los límites del sistema y sus integraciones externas.

**Decisiones de diseño notables:**
- Stripe es el único proveedor de pagos para simplificar la integración y aprovechar su soporte nativo de webhooks.
- SendGrid y Firebase Cloud Messaging son los canales de notificación (email y push respectivamente), ambos servicios gestionados para no tener que operar infraestructura de entrega de mensajes.
- Google Maps API se usa para la visualización de ubicaciones de inmuebles en el frontend.
- OpenAI es la fuente del módulo de IA — el sistema no entrena modelos propios.
- El Administrador es el único actor que no interactúa a través del frontend web estándar, sino a través de un panel de administración separado.

```mermaid
C4Context
    title PropConnect — Diagrama de Contexto del Sistema

    Person(consultant, "Consultor", "Busca inmuebles para comprar o alquilar. Usa filtros, IA y agenda citas.")
    Person(seller, "Vendedor", "Publica inmuebles en venta o renta. Contrata asesores y tramitadores.")
    Person(advisor, "Asesor", "Ofrece servicios inmobiliarios bajo contrato. Accede a datos completos del inmueble.")
    Person(processor, "Tramitador", "Gestiona procesos legales entre comprador y vendedor bajo contrato.")
    Person(admin, "Administrador", "Supervisa la plataforma, gestiona usuarios y resuelve quejas.")

    System(propconnect, "PropConnect", "Marketplace de bienes raíces que conecta consultores, vendedores, asesores y tramitadores.")

    System_Ext(stripe, "Stripe", "Procesa pagos de anuncios boost y comisiones. Envía webhooks de confirmación.")
    System_Ext(sendgrid, "SendGrid", "Entrega notificaciones por email: bienvenida, contratos, facturas, recordatorios.")
    System_Ext(firebase, "Firebase Cloud Messaging", "Entrega notificaciones push a dispositivos móviles.")
    System_Ext(googlemaps, "Google Maps API", "Proporciona mapas y geocodificación para ubicaciones de inmuebles.")
    System_Ext(openai, "OpenAI API", "Potencia el asistente IA: recomendaciones, chatbot y valoración de inmuebles.")

    Rel(consultant, propconnect, "Busca inmuebles, agenda citas, usa IA", "HTTPS")
    Rel(seller, propconnect, "Gestiona publicaciones, contrata profesionales, paga boosts", "HTTPS")
    Rel(advisor, propconnect, "Acepta contratos, agenda citas, accede a datos de inmuebles", "HTTPS")
    Rel(processor, propconnect, "Acepta contratos, gestiona procesos legales", "HTTPS")
    Rel(admin, propconnect, "Administra usuarios, resuelve quejas, monitorea la plataforma", "HTTPS")

    Rel(propconnect, stripe, "Crea PaymentIntents y recibe webhooks de pago", "HTTPS/REST")
    Rel(stripe, propconnect, "Envía webhooks payment_intent.succeeded / failed", "HTTPS Webhook")
    Rel(propconnect, sendgrid, "Envía emails transaccionales vía API", "HTTPS/REST")
    Rel(propconnect, firebase, "Envía notificaciones push vía FCM API", "HTTPS/REST")
    Rel(propconnect, googlemaps, "Geocodifica direcciones y renderiza mapas", "HTTPS/REST")
    Rel(propconnect, openai, "Envía prompts y contexto de inmuebles para recomendaciones y chat", "HTTPS/REST")
```

## Notas sobre el Diagrama

- **Dirección de las relaciones**: Las flechas indican la dirección del flujo de comunicación iniciado. Stripe → PropConnect refleja los webhooks que Stripe envía proactivamente.
- **Actores sin sistema externo de identidad**: Se tomó la decisión de no usar un proveedor de identidad externo (Auth0, Cognito) — PropConnect gestiona su propia autenticación (ver ADR-005).
- **Sin integración de portales inmobiliarios externos** (ej. Zillow, Inmuebles24): PropConnect es un sistema cerrado en su fase inicial. La integración con portales de terceros es una evolución futura.
