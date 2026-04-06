# 02 — Base de Datos Primaria (PostgreSQL)

## 1. Responsabilidades

- Almacenar todos los datos transaccionales del sistema: usuarios, publicaciones, contratos, pagos, citas y reseñas.
- Garantizar consistencia ACID en operaciones críticas como la firma de contratos y el procesamiento de pagos.
- Proveer las funcionalidades de búsqueda geoespacial básica (con `PostGIS`) para consultas de proximidad de inmuebles.
- Ejecutar migraciones de esquema versionadas a través de TypeORM Migrations.
- Generar backups automáticos diarios a AWS S3 via `pg_dump`.

## 2. Por Qué Este Proyecto lo Necesita

PropConnect tiene flujos de negocio que requieren transacciones ACID entre múltiples entidades. Por ejemplo: firmar un contrato implica actualizar el `Contract` a ACTIVE, crear un `AccessGrant` y descontar la comisión reservada — estas tres operaciones deben ser atómicas. Un sistema sin ACID (como MongoDB en modo básico) requeriría lógica de compensación manual para este tipo de flujos.

## 3. Elección de Tecnología

| Dimensión | PostgreSQL 16 (elegido) | MySQL 8 | MongoDB |
|---|---|---|---|
| **Gestionado / Self-hosted** | Self-hosted (Docker) | Self-hosted | Self-hosted o Atlas |
| **Complejidad operacional** | Media | Media | Baja (schema-less) |
| **Costo a nuestra escala** | Gratuito | Gratuito | Gratuito (self-hosted) |
| **Característica diferenciadora** | ACID fuerte + PostGIS + JSON nativo + extensiones ricas | ACID sólido, menor soporte de extensiones | Flexibilidad de esquema, sin JOINs eficientes |

PostgreSQL se elige por su soporte nativo de `PostGIS` para geoespacial, su madurez con ORM como TypeORM, y su robustez en operaciones transaccionales complejas.

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| ACID completo para flujos de pago y contratos | Escalabilidad horizontal de escrituras más compleja que NoSQL |
| PostGIS para consultas de ubicación sin servicio externo | Requiere gestionar migraciones cuidadosamente al evolucionar el esquema |
| TypeORM tiene soporte maduro para PostgreSQL | Un punto único de fallo si no se configura replicación (mitigado con backups frecuentes) |
| JSON nativo (`jsonb`) para campos flexibles como `preferences` en AI | |

## 5. Integración

PostgreSQL corre en su propio contenedor Docker con un volumen persistente (`pg_data`). El contenedor `propconnect-api` se conecta en la red interna del VPC. Los backups se envían a S3 mediante un cronjob nativo del contenedor. Ver `04-deployment.md`.
