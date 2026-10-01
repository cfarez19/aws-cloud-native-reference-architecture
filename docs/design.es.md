# Arquitectura cloud-native en AWS — Documento de diseño

**Autora:** Catalina Farez · AWS Certified Solutions Architect – Associate

## Alcance

Arquitectura para una aplicación web nativa de nube con cuatro componentes:

- **Frontend:** aplicación web que los clientes usan para navegar.
- **Backend:** servicios que se comunican con la base de datos y el frontend.
- **Base de datos:** sistema de gestión que almacena la información.
- **Almacenamiento de objetos:** imágenes y contenido estático.

![Diagrama de arquitectura](architecture.png)

## Proveedor: Amazon Web Services

- **Madurez y ecosistema:** la mayor cantidad de servicios administrados, para que el equipo se enfoque en el producto y no en la infraestructura base.
- **Seguridad y cumplimiento:** certificaciones PCI-DSS, SOC 2 e ISO 27001, necesarias para aplicaciones con datos sensibles de usuarios.
- **Escalabilidad nativa:** ECS Fargate, RDS y CloudFront son administrados y escalan sin gestionar servidores.
- **Presencia regional:** regiones en Latinoamérica para baja latencia y requisitos de soberanía de datos.
- **Adopción en fintech e insurtech:** el proveedor con más casos documentados en arquitecturas similares.

## Decisiones de diseño

El diseño sigue principios de alta disponibilidad, seguridad por capas y escalabilidad automática, sobre la región `us-east-1` en **tres zonas de disponibilidad**.

**Red y aislamiento — VPC.** La infraestructura vive en una VPC con subredes públicas y privadas en tres AZ, para resistir la caída de un centro de datos. La capa pública aloja el NAT Gateway para la salida a internet; la capa privada aísla el backend de cualquier acceso directo.

**Route 53.** DNS administrado para el dominio de la aplicación, enruta hacia CloudFront y permite políticas de enrutamiento avanzadas y failover.

**WAF + CloudFront.** CloudFront es el único punto de entrada y funciona como CDN global para el contenido estático, reduciendo la latencia. AWS WAF sobre CloudFront protege contra SQL injection, XSS y DDoS sin replicar esa protección en capas internas, lo que también optimiza costos.

**Certificate Manager (ACM).** Gestiona y renueva automáticamente los certificados de CloudFront y API Gateway; toda la comunicación viaja cifrada por HTTPS/TLS.

**Frontend en S3.** El frontend se despliega como contenido estático en un bucket S3 privado, servido solo a través de CloudFront. Sin servidores dedicados, con menor costo y alta disponibilidad.

**Cognito.** Autenticación y autorización de usuarios, integrado como autorizador en API Gateway: los tokens JWT se validan antes de que la petición llegue al backend.

**API Gateway + VPC Link.** API Gateway REST es el punto de entrada al backend. Como vive fuera de la VPC, se conecta a la red privada mediante VPC Link, sin exponer el backend a internet.

**Internal Load Balancer.** Un ALB interno distribuye el tráfico de API Gateway entre las tareas de ECS Fargate en las tres AZ, con tolerancia a la caída de cualquier zona.

**ECS Fargate.** Los microservicios del backend corren en contenedores en subredes privadas de tres AZ, con Auto Scaling según la demanda real y sin servidores que administrar. Se despliegan con CI/CD (AWS CodePipeline, GitHub Actions o Azure DevOps).

**RDS PostgreSQL Multi-AZ.** Instancia primaria para escrituras y réplica sincrónica en standby en otra AZ para failover automático. Datos cifrados en reposo con AWS KMS. La VPC de base de datos está separada de la VPC de aplicación y se conecta por VPC Peering, aislando por completo la capa de datos.

**S3 para objetos.** Buckets dedicados a imágenes, archivos y contenido generado por la aplicación. El backend sube los archivos y CloudFront los sirve; S3 ofrece alta durabilidad, escalabilidad y bajo costo.

**AWS Backup.** Planes de respaldo y Backup Vault centralizados para RDS y demás recursos críticos, para recuperación ante desastres y cumplimiento de políticas de retención.

**Seguridad y gobierno transversal.** CloudWatch para métricas y alarmas, GuardDuty para detección de amenazas, Security Hub para visibilidad centralizada, Inspector para vulnerabilidades, CloudTrail para auditoría, Config para cumplimiento de configuraciones e IAM con mínimo privilegio.

## Monitoreo por capas

| Capa | Métricas clave | Servicio |
| --- | --- | --- |
| Infraestructura | CPU, memoria, almacenamiento | CloudWatch (dispara el autoscaling) |
| Datos | IOPS, latencia de lectura/escritura, conexiones activas | CloudWatch / RDS |
| Red | Throughput, tasa de error de paquetes, latencia | CloudWatch |
| Observabilidad | Latencia por segmento con trazas distribuidas | AWS X-Ray |
| Seguridad | Errores de autenticación (fuerza bruta), tráfico bloqueado (DDoS) | WAF, GuardDuty, Security Hub |
| Costos | Gasto diario, costo por transacción, cobertura de Savings Plans | AWS Budgets, Cost Explorer |
