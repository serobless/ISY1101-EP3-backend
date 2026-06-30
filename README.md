# ISY1101 EP3 - Backend (Orquestación ECS)

## Descripción
Backend Spring Boot REST API desplegado en AWS ECS Fargate como parte de la Evaluación Parcial N°3 de Introducción a Herramientas DevOps (ISY1101). El servicio forma parte de la solución de orquestación y automatización en la nube para la empresa ficticia Innovatech Chile.

## Arquitectura de despliegue

El backend corre en una **Task Definition de ECS Fargate** que contiene 2 contenedores:
- **mysql**: base de datos MySQL 8.0 (sidecar pattern, comparte red `awsvpc` con el backend vía `127.0.0.1`)
- **backend**: API REST Spring Boot 3.4.4 / Java 17

Ambos contenedores se comunican por `localhost` ya que comparten el mismo namespace de red de la tarea Fargate.

## Componentes AWS utilizados
- **Amazon ECS (Fargate)**: orquestación de contenedores sin servidor
- **Amazon ECR**: registro de imágenes Docker (`ep3-backend`)
- **Application Load Balancer**: balanceo de carga con healthcheck en `/` (acepta 200 y 404 como saludable, ya que la API no expone una ruta raíz)
- **Application Auto Scaling**: Target Tracking sobre CPU (umbral 20%, min 1 - max 3 tareas)
- **CloudWatch Logs**: grupo `/ecs/ep3-backend`, streams separados para `mysql` y `ecs` (backend)
- **IAM LabRole**: rol compartido de ejecución y de tarea (restricción del AWS Academy Learner Lab, que no permite `iam:CreateRole`)

## Variables de entorno (Task Definition)
| Variable | Valor | Propósito |
|---|---|---|
| SPRING_DATASOURCE_URL | jdbc:mysql://127.0.0.1:3306/ventas_db?useSSL=false&serverTimezone=UTC&createDatabaseIfNotExist=true&allowPublicKeyRetrieval=true | Conexión JDBC al sidecar MySQL |
| DB_USERNAME | ventas_user | Usuario de aplicación |
| DB_PASSWORD | ventas_pass | Password de aplicación |

## Pipeline CI/CD (GitHub Actions)
Workflow `.github/workflows/deploy.yml`, disparado en cada push a `main`:
1. Checkout del código
2. Configuración de credenciales AWS (vía GitHub Secrets)
3. Login a Amazon ECR
4. Build de la imagen Docker (contexto: `./Springboot-API-REST`) y push a ECR con tag `latest` y tag del commit SHA
5. Descarga de la Task Definition actual
6. Actualización de la Task Definition con la nueva imagen
7. Deploy al servicio ECS con espera de estabilidad (`wait-for-service-stability: true`)

## Cómo correr el proyecto localmente
```bash
cd Springboot-API-REST
./mvnw spring-boot:run
```
Requiere una instancia MySQL local o remota accesible, configurada vía `SPRING_DATASOURCE_URL`, `DB_USERNAME`, `DB_PASSWORD`.

## Problemas resueltos durante la implementación
1. **403/Permission denied en Fargate**: el contenedor backend necesitó ajustes de puertos no privilegiados (resuelto en el repo de frontend para nginx; el backend ya escuchaba en 8080).
2. **Variables de entorno sin resolver (`${DB_PORT}`)**: causado por usar variables sueltas que ECS no interpola igual que docker-compose; solucionado consolidando todo en una única variable `SPRING_DATASOURCE_URL`.
3. **"Public Key Retrieval is not allowed"**: error característico de MySQL 8 con `caching_sha2_password`; resuelto agregando `allowPublicKeyRetrieval=true` a la cadena de conexión JDBC.
4. **Timing de arranque MySQL vs Spring Boot**: se agregó un `healthCheck` al contenedor MySQL (`mysqladmin ping`) y `dependsOn` con `condition: HEALTHY` en el backend, para que Spring Boot espere a que MySQL esté realmente listo antes de intentar conectar.
5. **ALB healthcheck fallando (404)**: la API no tiene una ruta pública en `/`; se ajustó el `Matcher` del Target Group para aceptar tanto `200` como `404` como respuesta saludable, confirmando que el servidor está vivo sin requerir un endpoint público dedicado.
