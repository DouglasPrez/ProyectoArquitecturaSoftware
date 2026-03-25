# 01 — Diagrama de Contexto del Sistema (C4 Nivel 1)

## Descripción

Este diagrama muestra PropConnect como una única caja de sistema, rodeada de todos sus actores humanos y sistemas externos. No se muestran componentes internos — el objetivo es comunicar los límites del sistema y sus integraciones externas.

**Decisiones de diseño notables:**
- Stripe es el único proveedor de pagos para simplificar la integración y aprovechar su soporte nativo de webhooks.
- SendGrid y Firebase Cloud Messaging (agrupados visualmente como "Notificaciones") son los canales de email y push respectivamente. Ambos son servicios gestionados para no operar infraestructura propia.
- Google Maps API se usa para la visualización de ubicaciones de inmuebles en el frontend.
- OpenAI es la fuente del módulo de IA — el sistema no entrena modelos propios.
- El Administrador es el único actor que no interactúa a través del frontend web estándar, sino a través de un panel de administración separado.

```mermaid
C4Context
    title PropConnect — Diagrama de Contexto del Sistema

    Person(consultant, "Consultor", "Busca inmuebles.<br/>Usa filtros e IA.")
    Person(seller, "Vendedor", "Publica inmuebles.<br/>Contrata servicios.")
    Person(advisor, "Asesor", "Ofrece servicios.<br/>Accede a datos.")
    Person(processor, "Tramitador", "Gestiona<br/>procesos legales.")
    Person(admin, "Administrador", "Supervisa la<br/>plataforma.")

    System(propconnect, "PropConnect", "Marketplace inmobiliario.<br/>Conecta a los usuarios.")

    System_Ext(stripe, "Stripe", "Procesa pagos<br/>y webhooks")
    System_Ext(notifications, "Notificaciones", "SendGrid (Email) y<br/>FCM (Push)")
    System_Ext(googlemaps, "Google Maps", "Geocodificación<br/>y mapas")
    System_Ext(openai, "OpenAI API", "Motor de IA<br/>y recomendaciones")

    Rel(consultant, propconnect, "Busca y agenda", "HTTPS")
    Rel(seller, propconnect, "Publica y paga", "HTTPS")
    Rel(advisor, propconnect, "Acepta contratos", "HTTPS")
    Rel(processor, propconnect, "Gestiona trámites", "HTTPS")
    Rel(admin, propconnect, "Administración", "HTTPS")

    BiRel(propconnect, stripe, "Pagos y Webhooks", "REST")
    Rel(propconnect, notifications, "Emails y push", "REST")
    Rel(propconnect, googlemaps, "Pide mapas", "REST")
    Rel(propconnect, openai, "Envía prompts", "REST")
```

## Notas sobre el Diagrama

- **Dirección de las relaciones**: Las flechas indican la dirección del flujo de comunicación iniciado. Stripe → PropConnect refleja los webhooks que Stripe envía proactivamente.
- **Actores sin sistema externo de identidad**: Se tomó la decisión de no usar un proveedor de identidad externo (Auth0, Cognito) — PropConnect gestiona su propia autenticación (ver ADR-005).
- **Sin integración de portales inmobiliarios externos** (ej. Zillow, Inmuebles24): PropConnect es un sistema cerrado en su fase inicial. La integración con portales de terceros es una evolución futura.
