# 01 — Arquitectura de Alto Nivel

## Descripción del Sistema

**PropConnect** es un marketplace de bienes raíces que conecta a consultores (compradores/arrendatarios), vendedores, asesores inmobiliarios y tramitadores legales en una plataforma unificada. El sistema gestiona publicaciones de inmuebles, procesos de contratación, pagos de comisiones y anuncios, agendamiento de citas, y consultas asistidas por IA.

---

## Enfoque A — Monolito Modular

### Descripción General

En este enfoque, todo el sistema vive en un único proceso desplegable, pero el código está organizado en módulos bien delimitados que se comunican a través de interfaces internas definidas. Cada módulo (por ejemplo, `Publicaciones`, `Contratos`, `Pagos`, `Usuarios`) expone una API interna y no permite acceso directo a sus capas de datos desde otros módulos.

### Aplicación al Proyecto

Para PropConnect, el monolito modular implicaría un único servicio backend (por ejemplo, NestJS o Spring Boot con módulos) que expone una API REST consumida por el frontend. Los módulos internos serían: `Auth`, `Listings`, `Users`, `Contracts`, `Payments`, `Appointments`, `Reviews`, `Notifications` y `AIConsultant`. La base de datos sería compartida, pero cada módulo accedería únicamente a sus propias tablas a través de repositorios aislados.

Dado que el equipo es pequeño y los dominios están estrechamente relacionados (por ejemplo, un contrato depende de una publicación y de dos usuarios), el monolito modular encaja naturalmente con la realidad del proyecto en sus fases iniciales.

---

## Enfoque B — Microservicios

### Descripción General

En este enfoque, cada capacidad del negocio se despliega como un servicio independiente con su propia base de datos y proceso de despliegue. Los servicios se comunican mediante APIs HTTPS para interacciones síncronas y a través de un bus de mensajes (Cómo RabbitMQ o Kafka) para eventos asíncronos. Existe un API Gateway como punto de entrada único.

### Aplicación al Proyecto

Para PropConnect, cada bounded context sería un microservicio autónomo: `user-service`, `listing-service`, `contract-service`, `payment-service`, `appointment-service`, `notification-service`, `review-service`, `ai-service`. Cada servicio tendría su propia base de datos (por ejemplo, PostgreSQL para datos transaccionales, Elasticsearch para búsqueda de inmuebles). La coordinación entre servicios se haría mediante eventos (Por ejemplo: `ContractSigned` dispara notificaciones y actualiza el estado de la publicación).

---

## Tabla Comparativa

| Dimensión | Monolito Modular | Microservicios |
|---|---|---|
| **Complejidad de despliegue** | Baja — un solo artefacto desplegable | Alta — múltiples servicios |
| **Tamaño de equipo adecuado** | 1–6 desarrolladores | 8+ desarrolladores (equipos por servicio) |
| **Escalabilidad horizontal** | Limitada — se escala todo el sistema | Alta — cada servicio escala de forma independiente |
| **Aislamiento de fallos** | Bajo — un error puede derribar todo el proceso | Alto — un servicio caído no afecta a los demás |
| **Velocidad de desarrollo (inicial)** | Alta — sin overhead de red ni contratos entre servicios | Baja — requiere definir contratos, infraestructura, CI/CD por servicio |
| **Velocidad de desarrollo (largo plazo)** | Media — puede volverse acoplado si no se disciplina | Alta — equipos autónomos, deploys independientes |
| **Costo de infraestructura** | Bajo — un servidor o contenedor principal | Alto — múltiples instancias, service mesh, monitoreo distribuido |
| **Complejidad operacional** | Baja — logs centralizados, un proceso | Alta — trazabilidad distribuida, múltiples pipelines de CI/CD |

---

## Elección: Monolito Modular

**Optamos por el Monolito Modular para PropConnect**, por las siguientes razones específicas al proyecto:

1. **Tamaño del equipo**: El proyecto se desarrolla en un contexto académico o de startup temprana con un equipo reducido. Los microservicios requieren madurez operacional que no es viable en esta fase.

2. **Interdependencia de dominios**: En PropConnect, los flujos de negocio son altamente cohesivos. Por ejemplo, publicar un anuncio pagado implica verificar el usuario, procesar el pago y actualizar la publicación en una sola operación. Distribuir esto en tres microservicios añade complejidad de consistencia distribuida sin un beneficio real todavía.

3. **Carga esperada inicial**: El sistema no anticipa tráfico masivo en sus primeras etapas. La escalabilidad granular de los microservicios sería un costo innecesario. Si la plataforma crece, los módulos bien definidos del monolito pueden extraerse como microservicios de forma progresiva utilizando el "Strangler Fig Pattern".

4. **Módulos con contratos claros**: Aunque se usa un monolito, cada módulo tendrá interfaces internas explícitas. Esto permite mantener la disciplina de bounded contexts sin pagar el costo operacional de la distribución.

5. **Iteración más rápida**: En un producto que aún está validando su modelo de negocio (tarifas de asesores, modelo de comisiones, funcionalidades de IA), la velocidad de iteración es más valiosa que la independencia de despliegue.

> **Nota de evolución**: Si PropConnect alcanza escala nacional con miles de publicaciones concurrentes, los módulos de `Listings`, `Payments` y `AIConsultant` son los candidatos naturales para extracción como microservicios, dado que son los de mayor carga y los más independientes funcionalmente.
