# ADR-003: Estrategia de Base de Datos

| Campo | Valor |
|---|---|
| **Status** | Aceptado |
| **Fecha** | 24/03/2026 |
| **Decisores** | Carlos Daniel Martínez |

## Contexto

PropConnect tiene 8 módulos con necesidades de datos distintas. Algunos módulos requieren consistencia ACID fuerte. El módulo de Listings necesita búsqueda full-text y filtros geoespaciales eficientes. El módulo de AI Consultant consume datos de Listings pero no escribe en ellos. La decisión de estrategia de base de datos determina el nivel de aislamiento entre módulos y la complejidad operacional.

## Opciones Consideradas

### Opción 1: Base de Datos Compartida con Schemas por Módulo

| Pros | Cons |
|---|---|
| Una sola instancia que administrar y respaldar | Sin aislamiento físico — un módulo mal escrito puede leer tablas de otro |
| JOINs entre módulos posibles (aunque se prohíben por convención) | Un solo punto de fallo para toda la persistencia |
| Consistencia transaccional entre módulos sin coordinación distribuida | Puede convertirse en un bottleneck si un módulo genera muchas escrituras |
| Costo operacional muy bajo | |

### Opción 2: Base de Datos por Módulo (Polyglot Persistence)

| Pros | Cons |
|---|---|
| Aislamiento físico total: ningún módulo puede acceder a los datos de otro | Múltiples bases de datos que administrar, respaldar y monitorear |
| Cada módulo puede usar el motor más adecuado (PostgreSQL, Redis, Elasticsearch) | Sin transacciones ACID entre módulos — requiere Sagas o compensación manual |
| Escala independientemente | Complejidad operacional muy alta para equipo pequeño |

### Opción 3: Base de Datos Compartida + Motor de Búsqueda Complementario

| Pros | Cons |
|---|---|
| PostgreSQL para todos los módulos transaccionales (ACID) | Sincronización entre PostgreSQL y Elasticsearch requiere un mecanismo adicional |
| Elasticsearch exclusivamente para el módulo de búsqueda de Listings | Dos tecnologías que conocer y operar |
| Menor complejidad que polyglot completo | |
| Cada módulo solo accede a sus propias tablas por convención + linting | |

## Decisión

Se adopta la **Opción 3: PostgreSQL compartido con prefijos de tabla por módulo, más Elasticsearch exclusivamente para búsqueda de Listings**.

**Justificación específica al proyecto:**

- La consistencia transaccional entre Contracts, Listings y Payments es un requisito real. Por ejemplo, confirmar un contrato y registrar el pago deben ser atómicos. Con bases de datos separadas esto requeriría una Saga distribuida, que es excesiva para el equipo actual.
- El módulo de Listings necesita búsqueda full-text con filtros (precio, tipo, habitaciones, zona) y consultas geoespaciales (radio en km). PostgreSQL puede hacer esto con extensiones (`pg_trgm`, `PostGIS`), pero Elasticsearch ofrece mejor rendimiento y relevancia para búsquedas de texto libre. Por eso se agrega Elasticsearch solo para este caso de uso.
- La sincronización Listings → Elasticsearch se hace de forma asíncrona: cuando `ListingPublished` o `ListingSoldOrRented` se publican en el EventBus, el adaptador de Elasticsearch en el módulo `listings` actualiza el índice.

### Convención de Nombrado de Tablas

| Módulo | Prefijo | Ejemplo |
|---|---|---|
| IAM | `iam_` | `iam_users`, `iam_roles` |
| Listings | `lst_` | `lst_listings`, `lst_properties` |
| Contracts | `ctr_` | `ctr_contracts`, `ctr_applications` |
| Payments | `pay_` | `pay_payments`, `pay_invoices` |
| Appointments | `apt_` | `apt_appointments`, `apt_availability` |
| Reviews | `rev_` | `rev_reviews`, `rev_complaints` |
| Notifications | `ntf_` | `ntf_notifications`, `ntf_templates` |
| AI Consultant | `ai_` | `ai_sessions`, `ai_messages` |

### Enforcement del Aislamiento

Aunque la base de datos es compartida, ningún módulo puede importar repositorios de otro. Se usa `dependency-cruiser` en el pipeline de CI para detectar y fallar el build ante violaciones de este tipo.

## Consecuencias

**Habilita:**
- Transacciones ACID entre módulos cuando sea necesario (especialmente Payments ↔ Contracts).
- Búsqueda eficiente de inmuebles con relevancia de texto y filtros geoespaciales.
- Una sola cadena de conexión y un solo backup a gestionar.

**Previene:**
- No se permiten queries con JOINs entre tablas de diferentes módulos, aunque técnicamente sea posible. La única forma correcta de obtener datos de otro módulo es a través de su `public-api.ts`.

**Riesgos y Mitigaciones:**
- *Desincronización PostgreSQL ↔ Elasticsearch*: Si el handler del evento falla, el índice queda desactualizado. Se mitiga con un job de reconciliación periódico que compara PostgreSQL contra Elasticsearch cada 5 minutos y reindexea los registros desincronizados.
- *Bottleneck en escrituras*: Si Listings crece masivamente, se puede migrar solo ese módulo a su propia instancia de PostgreSQL sin afectar a los demás.

## ADRs Relacionados

- ADR-001 (Modelo de Despliegue) — contexto del monolito.
