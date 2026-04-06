# 07 — Almacenamiento de Objetos (AWS S3 + CloudFront)

## 1. Responsabilidades

- Almacenar las fotos de los inmuebles publicadas por los vendedores.
- Generar URLs prefirmadas (presigned URLs) para que los clientes suban imágenes directamente sin pasar por el servidor de la aplicación.
- Almacenar los PDFs de facturas generados por el módulo de Payments.
- Gestionar el ciclo de vida de archivos: eliminar imágenes cuando una publicación es eliminada (S3 Lifecycle Policy).
- Servir imágenes a través de CloudFront CDN para distribución geográfica eficiente.

## 2. Por Qué Este Proyecto lo Necesita

Las publicaciones de inmuebles en PropConnect requieren fotos de alta calidad (múltiples imágenes por propiedad). Almacenarlas en el servidor de aplicación o en PostgreSQL sería un antipatrón: saturarían el disco del servidor, bloquearían el proceso Node.js durante uploads, y harían los backups innecesariamente grandes. S3 está diseñado específicamente para este tipo de almacenamiento binario de gran volumen.

## 3. Elección de Tecnología

| Dimensión | AWS S3 (elegido) | Google Cloud Storage | MinIO (self-hosted) |
|---|---|---|---|
| **Gestionado / Self-hosted** | Totalmente gestionado | Totalmente gestionado | Self-hosted |
| **Complejidad operacional** | Baja | Baja | Media (administrar servidor de almacenamiento) |
| **Costo a nuestra escala** | ~$0.023/GB/mes + $0.09/GB transferencia | Similar a S3 | Solo costo de servidor |
| **Característica diferenciadora** | Integración nativa con CloudFront, IAM de AWS, SDK maduro | Mejor integración con GCP | Sin costo de almacenamiento, control total |

## 4. Trade-offs

| Ventajas | Desventajas |
|---|---|
| Presigned URLs evitan que el servidor sea cuello de botella en uploads | Costo variable según volumen almacenado y transferencia |
| Integración nativa con CloudFront para CDN | Vendor lock-in con AWS (mitigado por la API S3-compatible de alternativas) |
| Durabilidad 99.999999999% (11 nueves) | Requiere configurar correctamente las políticas IAM para seguridad |
| Lifecycle policies automáticas para limpiar archivos huérfanos | |

## 5. Integración

El módulo `listings` genera presigned URLs para uploads directos desde el frontend. Las URLs resultantes se almacenan en `lst_properties.mediaUrls`. El módulo `payments` sube los PDFs de facturas a S3 y guarda la URL en `pay_invoices.downloadUrl`. Las imágenes se sirven a través de CloudFront, no directamente desde S3. Ver `04-deployment.md`.
