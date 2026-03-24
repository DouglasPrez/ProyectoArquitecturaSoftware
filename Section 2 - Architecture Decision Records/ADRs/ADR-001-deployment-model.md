# ADR-001: Modelo de Despliegue

| Campo | Valor |
|---|---|
| **Status** | Aceptado |
| **Fecha** | 24/03/2026 |
| **Decisores** | Carlos Daniel Martínez & Douglas Daniel Pérez Hernández |

## Contexto

PropConnect necesita un modelo de despliegue que se adapte a un equipo reducido, un dominio con 8 bounded contexts interconectados, y una carga inicial moderada. Los dominios comparten datos frecuentemente: un contrato referencia un inmueble, que a su vez pertenece a un usuario con roles. La consistencia transaccional entre estos contextos es un requisito real del negocio.

## Opciones Consideradas

### Opción 1: Monolito Modular

| Pros | Cons |
|---|---|
| Un solo artefacto desplegable — pipeline de CI/CD simple | Escala como unidad — no se puede escalar solo el módulo de Listings |
| Transacciones ACID entre módulos sin coordinación distribuida | Un bug crítico puede tirar todo el proceso |
| Iteración rápida: no hay latencia de red entre módulos | Requiere disciplina de equipo para respetar boundaries internos |
| Debugging local directo | Riesgo de acoplamiento gradual si no se hacen code reviews rigurosos |

### Opción 2: Microservicios

| Pros | Cons |
|---|---|
| Escalabilidad granular por servicio | Requiere service mesh, API Gateway, observabilidad distribuida |
| Aislamiento de fallos: un servicio caído no afecta a los demás | Consistencia eventual complica flujos como firma de contrato + acceso |
| Equipos completamente autónomos por servicio | Overhead de infraestructura muy alto para equipo pequeño |
| Despliegues independientes sin coordinación | Latencia de red en cada llamada entre servicios |

## Decisión

Se adopta el **Monolito Modular**.

La interdependencia transaccional entre Contracts, Listings y Payments hace que la consistencia eventual de los microservicios introduzca complejidad sin beneficio real en esta etapa. Además, el equipo no tiene la madurez operacional para gestionar un service mesh, múltiples pipelines de CI/CD y trazabilidad distribuida desde el día uno.

El monolito modular permite mantener los bounded contexts bien definidos (como se documenta en `02-bounded-contexts.md`) y migrar módulos específicos a microservicios cuando el volumen lo justifique, usando el patrón Strangler Fig.

## Consecuencias

**Habilita:**
- Desarrollo rápido con un solo comando de arranque local (`docker-compose up`).
- Transacciones ACID entre módulos de forma natural.
- Onboarding sencillo para nuevos desarrolladores.

**Previene / Restringe:**
- No se puede escalar horizontalmente solo el módulo de Listings (debe escalar todo el proceso).
- El módulo de AI Consultant hace llamadas a OpenAI que pueden ser lentas; esto se mitiga con timeouts estrictos y colas asíncronas para evitar bloquear el proceso principal.

**Riesgos y Mitigaciones:**
- *Acoplamiento gradual*: Se mitiga con linting de arquitectura (`eslint-plugin-boundaries` o `dependency-cruiser`) que falla el build si un módulo importa desde las capas internas de otro.
- *Escalabilidad futura*: Los módulos `Listings` y `Payments` son los candidatos prioritarios para extracción si el sistema crece.

## ADRs Relacionados

- ADR-002 (Estilo de Comunicación) — define cómo los módulos se comunican internamente.
- ADR-003 (Estrategia de Base de Datos) — decisión sobre base de datos compartida vs. por módulo.
