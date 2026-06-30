# Infraestructura AWS — Backend (EP3)

Este documento registra los comandos AWS CLI ejecutados para aprovisionar la infraestructura del backend en AWS ECS Fargate. Sirve como evidencia reproducible y referencia técnica para la defensa oral.

> **Nota:** todos los comandos se ejecutaron desde AWS CloudShell en la región `us-east-1`, bajo la cuenta del AWS Academy Learner Lab. El rol `LabRole` se usó como execution role y task role por restricción de la plataforma educativa (no permite `iam:CreateRole`).

---

## 1. Repositorio ECR

```bash
# Creado vía consola AWS (Amazon ECR → Crear repositorio privado)
# Nombre: ep3-backend
# Mutabilidad: Mutable
# Cifrado: AES-256
```

URI resultante: `593376755101.dkr.ecr.us-east-1.amazonaws.com/ep3-backend`

---

## 2. Clúster ECS

```bash
# Habilitar el rol de servicio de ECS (si no existe en la cuenta)
aws iam create-service-linked-role --aws-service-name ecs.amazonaws.com

# Crear el clúster Fargate
aws ecs create-cluster \
  --cluster-name ep3-cluster \
  --capacity-providers FARGATE \
  --default-capacity-provider-strategy capacityProvider=FARGATE,weight=1
```

---

## 3. Networking base

```bash
# VPC por defecto
aws ec2 describe-vpcs --query 'Vpcs[?IsDefault==`true`].{VpcId:VpcId,CIDR:CidrBlock}' --output table
# → vpc-05249474cb55efe84 (172.31.0.0/16)

# Subredes disponibles (se usaron 2 en distintas AZ para alta disponibilidad)
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-05249474cb55efe84" \
  --query 'Subnets[*].{SubnetId:SubnetId,AZ:AvailabilityZone,CIDR:CidrBlock}' --output table
# → subnet-0f5c1a809cced7b33 (us-east-1e), subnet-01ed48dca89140505 (us-east-1b)
```

---

## 4. Security Groups

```bash
# SG del backend
aws ec2 create-security-group --group-name ep3-backend-sg \
  --description "EP3 Backend Security Group" --vpc-id vpc-05249474cb55efe84
# → sg-0d0e510606e807d7f

# Permitir tráfico desde el frontend
aws ec2 authorize-security-group-ingress --group-id sg-0d0e510606e807d7f \
  --protocol tcp --port 8080 --source-group sg-04d3232ded8fcdf76

# Permitir tráfico desde el ALB (agregado en el paso 6)
aws ec2 authorize-security-group-ingress --group-id sg-0d0e510606e807d7f \
  --protocol tcp --port 8080 --source-group sg-07e521c077e38de2e
```

---

## 5. Task Definition (backend + MySQL sidecar)

Versión final (`ep3-backend:5`), con healthcheck en MySQL y dependencia condicionada en el backend:

```bash
aws ecs register-task-definition \
  --family ep3-backend \
  --network-mode awsvpc \
  --requires-compatibilities FARGATE \
  --cpu 512 \
  --memory 1024 \
  --execution-role-arn arn:aws:iam::593376755101:role/LabRole \
  --task-role-arn arn:aws:iam::593376755101:role/LabRole \
  --container-definitions '[
    {
      "name": "mysql",
      "image": "mysql:8.0",
      "essential": true,
      "environment": [
        {"name": "MYSQL_ROOT_PASSWORD", "value": "rootpass"},
        {"name": "MYSQL_DATABASE", "value": "ventas_db"},
        {"name": "MYSQL_USER", "value": "ventas_user"},
        {"name": "MYSQL_PASSWORD", "value": "ventas_pass"}
      ],
      "portMappings": [{"containerPort": 3306, "protocol": "tcp"}],
      "healthCheck": {
        "command": ["CMD-SHELL", "mysqladmin ping -h 127.0.0.1 -u root -prootpass || exit 1"],
        "interval": 10, "timeout": 5, "retries": 10, "startPeriod": 30
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/ep3-backend",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "mysql",
          "awslogs-create-group": "true"
        }
      }
    },
    {
      "name": "backend",
      "image": "593376755101.dkr.ecr.us-east-1.amazonaws.com/ep3-backend:latest",
      "essential": true,
      "dependsOn": [{"containerName": "mysql", "condition": "HEALTHY"}],
      "environment": [
        {"name": "SPRING_DATASOURCE_URL", "value": "jdbc:mysql://127.0.0.1:3306/ventas_db?useSSL=false&serverTimezone=UTC&createDatabaseIfNotExist=true&allowPublicKeyRetrieval=true"},
        {"name": "DB_USERNAME", "value": "ventas_user"},
        {"name": "DB_PASSWORD", "value": "ventas_pass"}
      ],
      "portMappings": [{"containerPort": 8080, "protocol": "tcp"}],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/ep3-backend",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs",
          "awslogs-create-group": "true"
        }
      }
    }
  ]'
```

---

## 6. Application Load Balancer

```bash
# Target Group
aws elbv2 create-target-group \
  --name ep3-backend-tg --protocol HTTP --port 8080 \
  --vpc-id vpc-05249474cb55efe84 --target-type ip \
  --health-check-protocol HTTP --health-check-path / \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 --unhealthy-threshold-count 3
# → arn:.../targetgroup/ep3-backend-tg/ec9ccd79b3d7a8f7

# Ajuste del healthcheck: la API no expone ruta pública en "/", por lo que
# devuelve 404 en vez de 200. Se acepta 404 como respuesta saludable.
aws elbv2 modify-target-group \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:593376755101:targetgroup/ep3-backend-tg/ec9ccd79b3d7a8f7 \
  --matcher '{"HttpCode":"200,404"}'

# Security Group del ALB
aws ec2 create-security-group --group-name ep3-alb-sg \
  --description "EP3 ALB Security Group" --vpc-id vpc-05249474cb55efe84
# → sg-07e521c077e38de2e
aws ec2 authorize-security-group-ingress --group-id sg-07e521c077e38de2e \
  --protocol tcp --port 80 --cidr 0.0.0.0/0

# Load Balancer
aws elbv2 create-load-balancer \
  --name ep3-backend-alb \
  --subnets subnet-0f5c1a809cced7b33 subnet-01ed48dca89140505 \
  --security-groups sg-07e521c077e38de2e \
  --scheme internet-facing --type application
# → DNS: ep3-backend-alb-2080595026.us-east-1.elb.amazonaws.com

# Listener
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:593376755101:loadbalancer/app/ep3-backend-alb/e5ce76d48b27b41d \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:593376755101:targetgroup/ep3-backend-tg/ec9ccd79b3d7a8f7
```

---

## 7. Servicio ECS vinculado al ALB

```bash
aws ecs create-service \
  --cluster ep3-cluster \
  --service-name ep3-backend-service \
  --task-definition ep3-backend:5 \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-0f5c1a809cced7b33,subnet-01ed48dca89140505],securityGroups=[sg-0d0e510606e807d7f],assignPublicIp=ENABLED}"

aws ecs update-service \
  --cluster ep3-cluster \
  --service ep3-backend-service \
  --load-balancers targetGroupArn=arn:aws:elasticloadbalancing:us-east-1:593376755101:targetgroup/ep3-backend-tg/ec9ccd79b3d7a8f7,containerName=backend,containerPort=8080 \
  --force-new-deployment
```

---

## 8. Auto Scaling (Target Tracking)

```bash
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/ep3-cluster/ep3-backend-service \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 1 --max-capacity 3

aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/ep3-cluster/ep3-backend-service \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name ep3-backend-cpu-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 20.0,
    "PredefinedMetricSpecification": {"PredefinedMetricType": "ECSServiceAverageCPUUtilization"},
    "ScaleInCooldown": 60,
    "ScaleOutCooldown": 60
  }'
```

> El umbral inicial se probó en 50% y 10% antes de fijarse en 20%, como balance entre sensibilidad de reacción y estabilidad frente al ruido de CPU base del sistema (ver README, sección "Problemas resueltos").

---

## 9. Verificación de autoscaling (evidencia)

```bash
# Simulación de carga (50 procesos concurrentes)
for i in {1..50}; do
  (while true; do curl -s -o /dev/null http://ep3-backend-alb-2080595026.us-east-1.elb.amazonaws.com/; done) &
done

# Monitoreo en tiempo real
watch -n 10 'aws ecs describe-services --cluster ep3-cluster --services ep3-backend-service \
  --query "services[0].{Running:runningCount,Desired:desiredCount,Pending:pendingCount}"'

# Detener la carga
kill $(jobs -p) 2>/dev/null
```

**Resultado observado (30-06-2026):** CPU subió de ~7% a ~33% → servicio escaló de 1 a 3 tareas. Al detener la carga, CPU bajó a ~7% → servicio redujo automáticamente de 3 a 1 tarea.
