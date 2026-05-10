# ISY1101-EP2 — Microservicio de Ventas

Microservicio REST de gestión de **ventas** para **Innovatech Chile**, desarrollado como parte del proyecto semestral ISY1101 EP2. Expone una API que permite crear, consultar y administrar registros de ventas, con persistencia en MySQL 8.

---

## Stack tecnológico

| Capa | Tecnología |
|---|---|
| Framework | Spring Boot 3 |
| Lenguaje | Java 17 |
| Persistencia | Spring Data JPA / Hibernate |
| Base de datos | MySQL 8.0 |
| Build | Maven Wrapper (`mvnw`) |
| Contenedor | Docker (multi-stage) |
| Orquestación local | Docker Compose |
| CI/CD | GitHub Actions |
| Registry | Docker Hub |

---

## Requisitos previos

- Docker >= 24
- Docker Compose >= 2.20

---

## Correr localmente con Docker Compose

```bash
# Clonar el repositorio
git clone https://github.com/serobless/ISY1101-EP2-back-ventas.git
cd ISY1101-EP2-back-ventas

# Levantar MySQL + aplicación
docker compose up --build
```

El servicio quedará disponible en `http://localhost:8080`.

Docker Compose levanta dos contenedores:
- `mysql-ventas` — MySQL 8.0 con healthcheck
- `app-ventas` — Spring Boot, espera a que MySQL esté healthy antes de iniciar

---

## Variables de entorno

| Variable | Descripción | Valor por defecto (compose local) |
|---|---|---|
| `SPRING_DATASOURCE_URL` | URL de conexión JDBC completa | `jdbc:mysql://mysql-ventas:3306/ventas_db?useSSL=false&serverTimezone=UTC&createDatabaseIfNotExist=true&allowPublicKeyRetrieval=true` |
| `DB_USERNAME` | Usuario de la base de datos | `ventas_user` |
| `DB_PASSWORD` | Contraseña de la base de datos | `ventas_pass` |

En producción estas variables se inyectan como secrets de GitHub Actions y se pasan al contenedor con `docker run -e`.

---

## Endpoints principales

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/api/ventas` | Listar todas las ventas |
| GET | `/api/ventas/{id}` | Obtener venta por ID |
| POST | `/api/ventas` | Crear nueva venta |
| PUT | `/api/ventas/{id}` | Actualizar venta |
| DELETE | `/api/ventas/{id}` | Eliminar venta |

---

## Pipeline CI/CD

El workflow `.github/workflows/deploy.yml` se activa con cada push a la rama `deploy`.

```
push → deploy branch
        │
        ▼
  [build-and-push]
  ┌──────────────────────────────────────┐
  │ 1. Checkout código                   │
  │ 2. Login a Docker Hub                │
  │ 3. docker build (context: ./Spring   │
  │    boot-API-REST)                    │
  │ 4. docker push → Docker Hub          │
  └──────────────────────────────────────┘
        │
        ▼ (needs: build-and-push)
  [deploy]
  ┌──────────────────────────────────────┐
  │ 1. SSH a instancia EC2               │
  │ 2. docker pull                       │
  │ 3. docker stop/rm app-ventas         │
  │ 4. docker run -p 8080:8080           │
  │    con -e SPRING_DATASOURCE_URL      │
  │       -e DB_USERNAME                 │
  │       -e DB_PASSWORD                 │
  └──────────────────────────────────────┘
```

### Secrets requeridos en GitHub

| Secret | Descripción |
|---|---|
| `DOCKER_USERNAME` | Usuario de Docker Hub |
| `DOCKER_PASSWORD` | Token de acceso Docker Hub |
| `EC2_HOST` | IP pública de la instancia EC2 |
| `EC2_USER` | Usuario SSH (ej. `ec2-user`) |
| `EC2_SSH_KEY` | Clave privada SSH (contenido del `.pem`) |
| `SPRING_DATASOURCE_URL` | URL JDBC completa con parámetros MySQL 8 |
| `DB_USERNAME` | Usuario de la base de datos |
| `DB_PASSWORD` | Contraseña de la base de datos |

---

## URL de producción

**http://13.221.44.192:8080**

---

## Decisiones técnicas

### Multi-stage build

El Dockerfile usa dos stages: `builder` (Maven + JDK 17) compila el proyecto con `mvnw package` y genera el `.jar`, y `runner` (JRE 17 slim) solo copia el artefacto compilado. La imagen final no contiene Maven, el código fuente ni las dependencias de compilación — reduce el tamaño de imagen de ~600 MB a ~200 MB y no expone el toolchain en producción.

### Usuario no root

El contenedor corre con un usuario sin privilegios definido en el Dockerfile. Reducir la superficie de ataque en caso de vulnerabilidad en la JVM o en la aplicación es una práctica estándar de hardening de contenedores.

### Named volumes vs bind mount

Se usa un named volume (`mysql_ventas_data`) en lugar de un bind mount para el directorio de datos de MySQL. Los named volumes son gestionados por Docker, evitan problemas de permisos entre el host (Windows/Mac/Linux) y el contenedor, y son portátiles sin ajustar rutas del sistema de archivos del host. Un bind mount requeriría que el directorio exista en el host y que los permisos sean compatibles con el UID del proceso MySQL dentro del contenedor.

### Docker Hub sobre ECR

Docker Hub fue elegido sobre Amazon ECR por simplicidad de integración con GitHub Actions: solo requiere dos secrets (`DOCKER_USERNAME`, `DOCKER_PASSWORD`) frente a la configuración de credenciales AWS, región y ARN que exige ECR. Para un proyecto académico o sin VPC privada, el overhead de ECR no aporta valor. ECR sería la elección correcta en producción corporativa con redes privadas o requisitos de compliance.
