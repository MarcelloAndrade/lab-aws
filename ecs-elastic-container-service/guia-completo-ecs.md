# 🐳 ECS (Elastic Container Service) - Guia Completo para Estudos

## 📚 Sumário

1. [O que é ECS](#o-que-é-ecs)
2. [Quando Usar ECS](#quando-usar-ecs)
3. [Arquitetura e Componentes](#arquitetura-e-componentes)
4. [Task Definition](#task-definition)
5. [Cluster](#cluster)
6. [Service](#service)
7. [Comparação: ECS vs Alternativas](#comparação-ecs-vs-alternativas)
8. [Segurança em Produção](#segurança-em-produção)
9. [Auto Scaling](#auto-scaling)
10. [CI/CD com ECS](#cicd-com-ecs)
11. [Well-Architected Framework](#well-architected-framework)
12. [Troubleshooting Comum](#troubleshooting-comum)

---

## O que é ECS?

O **ECS (Elastic Container Service)** é o serviço gerenciado da AWS para orquestração de containers Docker. Diferente do Kubernetes, que requer gerenciamento fino de cada nó, o ECS abstrai a infraestrutura subjacente, permitindo que o arquiteto delegue à AWS responsabilidades operacionais significativas.

### Características Principais

- **Gerenciamento simplificado**: AWS cuida da orquestração
- **Integração nativa**: Works seamlessly com ECR, CloudWatch, ALB, IAM
- **Dois modelos de execução**: Fargate (serverless) e EC2 (full control)
- **Auto-scaling integrado**: Application Auto Scaling com métricas customizadas
- **Observabilidade nativa**: Container Insights sem overhead
- **Deployments seguros**: Automatic rollback em falhas

### Por que NÃO é Kubernetes?

- Kubernetes oferece controle fino, mas exige expertise operacional
- ECS é a escolha quando simplicidade operacional importa
- Ambos são válidos; choose based on constraints

---

## Quando Usar ECS

### ✅ ECS É Apropriado Para

| Cenário | Razão | Modelo |
|---------|-------|--------|
| APIs REST / GraphQL com demanda variável | Scaling fino, startup rápido | Fargate |
| Microserviços (< 50 serviços) | Cada serviço em seu próprio ECS Service | Fargate |
| Batch jobs agendados | Pague apenas durante execução | Fargate |
| Aplicações legadas containerizadas | Migrate & improve gradualmente | EC2 |
| Workloads com requisitos de rede específicos | VPC nativo, SG granular | EC2 |
| Processamento de imagens / vídeos | Fargate + SQS por job queue | Fargate |

### ❌ ECS É Inadequado Para

| Cenário | Alternativa | Por quê |
|---------|-------------|--------|
| Funções simples, acionadas por eventos | AWS Lambda | Menos overhead, cold start aceitável |
| Workloads distribuídas complexas (Cassandra, Kafka) | EKS ou EC2 diretamente | Requer controle fino de topologia |
| Arquitetura com 200+ microserviços | Kubernetes (EKS) | ECS pode ficr complexo no gerenciamento |
| Aplicação sem containerização | EC2, Lightsail | Containerizar tem custo de infra |

### Trade-offs: Decisão Final

```
Operacional Simplicity ← ECS Fargate → Controle fino
Custo Previsível       ← ECS Fargate → Pay-per-use
Startup Speed          ← Fargate    → ~30s | EC2 → ~1m
Escalabilidade         ← Ambos      → Ilimitada
```

**Recomendação Geral**: Comece com Fargate. Migre para EC2 apenas se benchmarks mostram necessidade.

---

## Arquitetura e Componentes

### Visão de Alto Nível

```
┌─────────────────────────────────────────────────────────┐
│                    AWS Account                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │   ECS CLU    │  │   ECS CLUS   │  │  ECS CLUS    │  │
│  │   CLUSTER    │  │   CLUSTER    │  │  CLUSTER     │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│        │                 │                  │           │
│        │                 │                  │           │
│   ┌────▼────┐       ┌────▼────┐       ┌────▼────┐      │
│   │ SERVICE  │       │ SERVICE  │       │ SERVICE  │      │
│   │  Task 1  │       │  Task 1  │       │  Task 1  │      │
│   │  Task 2  │       │  Task 2  │       │  Task 2  │      │
│   └──────────┘       └──────────┘       └──────────┘      │
│        │                 │                  │           │
│        └─────────────────┼──────────────────┘           │
│                          │                              │
│                    ┌─────▼──────┐                       │
│                    │   ALB /     │                       │
│                    │   NLB       │                       │
│                    └─────┬──────┘                       │
│                          │                              │
└──────────────────────────┼──────────────────────────────┘
                           │
                    ┌──────▼──────┐
                    │   Internet   │
                    └─────────────┘
```

### Os Três Componentes Principais

#### 1️⃣ **Task Definition**
Define como um container será executado. É um **blueprint imutável**.

```hcl
# Analogia: Task Definition = Docker Compose file versionado
# Cada alteração cria uma nova revisão
```

#### 2️⃣ **Cluster**
Agrupamento lógico de recursos. Abstrai se é Fargate ou EC2.

```hcl
# Analogia: Cluster = Grupo de máquinas (físico ou lógico)
# ECS gerencia automaticamente naquele cluster
```

#### 3️⃣ **Service**
Garante N tasks rodando continuamente. Gerencia desejado state.

```hcl
# Analogia: Service = Systemd/Supervisor em produção
# Se uma task cai, Service inicia nova
```

---

## Task Definition

A Task Definition é o artefato mais crítico em ECS. Contém tudo necessário para rodar um container: imagem, portas, variáveis, IAM, logs, health checks.

### Exemplo Completo

```hcl
resource "aws_ecs_task_definition" "app_production" {
  # Identidade única
  family                   = "minha-aplicacao"
  
  # Compatibilidade Fargate
  network_mode             = "awsvpc"  # OBRIGATÓRIO para Fargate
  requires_compatibilities = ["FARGATE"]
  
  # Recursos alocados (fixa em Fargate)
  cpu    = "256"   # vCPU em unidades: 256, 512, 1024, 2048, 4096
  memory = "512"   # Memory em MB
  
  # Roles IAM
  execution_role_arn = aws_iam_role.ecs_task_execution_role.arn  # ECR, logs
  task_role_arn      = aws_iam_role.ecs_task_role.arn             # App permissions
  
  container_definitions = jsonencode([
    {
      name      = "minha-aplicacao"
      image     = "123456789.dkr.ecr.us-east-1.amazonaws.com/minha-aplicacao:v1.2.3"
      essential = true  # Se falhar, task falha
      
      # Porta exposta
      portMappings = [
        {
          containerPort = 8080
          hostPort      = 8080  # Em Fargate, deve ser igual
          protocol      = "tcp"
        }
      ]
      
      # ✅ CORRETO: Secrets via Secrets Manager
      secrets = [
        {
          name      = "DB_PASSWORD"
          valueFrom = "${aws_secretsmanager_secret.db_password.arn}:password::"
        },
        {
          name      = "API_KEY"
          valueFrom = "${aws_secretsmanager_secret.api_key.arn}:api_key::"
        }
      ]
      
      # Variáveis de ambiente públicas
      environment = [
        { name = "ENVIRONMENT", value = "production" },
        { name = "LOG_LEVEL", value = "info" },
        { name = "SERVICE_NAME", value = "minha-aplicacao" },
        { name = "JAVA_OPTS", value = "-Xms256m -Xmx512m" }
      ]
      
      # ✅ Logging centralizado
      logConfiguration = {
        logDriver = "awslogs"
        options = {
          "awslogs-group"         = aws_cloudwatch_log_group.ecs_app.name
          "awslogs-region"        = var.aws_region
          "awslogs-stream-prefix" = "ecs"
        }
      }
      
      # ✅ Health Check (CRÍTICO)
      healthCheck = {
        command     = ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"]
        interval    = 30        # A cada 30s
        timeout     = 5         # Max 5s para responder
        retries     = 2         # 2 falhas antes de unhealthy
        startPeriod = 60        # Grace period de 60s no start
      }
      
      # Regra de interrupção
      stopTimeout = 120  # Max 120s para SIGTERM antes de SIGKILL
      
      # Mount points (se usar EBS, EFS, etc)
      # mountPoints = [... ]
      
      # Ambiente privilegiado (NÃO recomendado em produção)
      privileged = false
      
      # Resource reservations (soft limits para task packing)
      # reservedCpuUnits   = 100
      # reservedMemoryMiB  = 256
    }
  ])

  # Tags para cost allocation
  tags = {
    Environment = "production"
    ManagedBy   = "terraform"
    CostCenter  = "platform"
    Application = "minha-aplicacao"
  }
}

# CloudWatch Log Group
resource "aws_cloudwatch_log_group" "ecs_app" {
  name              = "/ecs/minha-aplicacao"
  retention_in_days = 30  # Custo-benefício: 30 dias é padrão produção

  tags = {
    Application = "minha-aplicacao"
  }
}
```

### O que NUNCA fazer em Task Definition

```hcl
# ❌ ANTI-PADRÃO 1: Hardcode secrets
container_definitions = jsonencode([{
  environment = [
    { name = "DB_PASSWORD", value = "super-secret-123" }  # NUNCA!
  ]
}])

# ❌ ANTI-PADRÃO 2: Versão "latest" em produção
image = "my-registry/app:latest"  # Não é idempotente!

# ❌ ANTI-PADRÃO 3: Sem health check
# Se container fica zombi, ECS não detecta

# ❌ ANTI-PADRÃO 4: Executar como root
# Linux user de container é root por padrão (NUNCA em produção)

# ❌ ANTI-PADRÃO 5: Sem logs centralizados
# Logs apenas em container fs, perdidos quando container morre
```

### Boas Práticas Task Definition

| Prática | Impacto |
|---------|---------|
| Usar versão específica de imagem (v1.2.3) | Reproducibilidade, rollback fácil |
| Health check sempre presente | Detecção rápida de falhas |
| Logs em CloudWatch | Observabilidade centralizada |
| Secrets via Secrets Manager | Segurança, auditoria |
| Resource limits claros | Previsibilidade de custo |
| Graceful shutdown (startPeriod) | Zero downtime deployments |

---

## Cluster

O Cluster é a abstração de infraestrutura no ECS. Pode rodar Fargate, EC2, ou ambos.

### Cluster Fargate (Recomendado)

```hcl
resource "aws_ecs_cluster" "main" {
  name = "producao-cluster"

  # Container Insights = Observabilidade em tempo real
  setting {
    name  = "containerInsights"
    value = "enabled"
  }

  tags = {
    Environment = "production"
    Terraform   = "true"
  }
}

# Capacity Providers define quais "tipos" de compute estão disponíveis
resource "aws_ecs_cluster_capacity_providers" "main" {
  cluster_name = aws_ecs_cluster.main.name

  # FARGATE: On-demand, preço full
  # FARGATE_SPOT: Until 70% discount, risco de interruption
  capacity_providers = ["FARGATE", "FARGATE_SPOT"]

  default_capacity_provider_strategy {
    base              = 1  # Sempre mínimo 1 on-demand para HA
    weight            = 100
    capacity_provider = "FARGATE"
  }

  # Overflow vai para SPOT se disponível
  default_capacity_provider_strategy {
    weight            = 100
    capacity_provider = "FARGATE_SPOT"  
  }
}
```

**Lógica de Seleção**:
- Base de 1 task em FARGATE (always available)
- Overflow em FARGATE_SPOT (70% cheaper)
- Se task Spot é interrompida, ECS relança em FARGATE

**Custo Real**:
- 2 tasks on FARGATE: 2 × CPU + 2 × Memory cost
- 1 on FARGATE + 1 on SPOT: (1 × full) + (1 × 30% do full)

### Cluster EC2 (Quando Necessário)

```hcl
# NOTA: Apenas use EC2 se realmente necessário
# Casos: GPU needed, custom kernel, 24/7 baseline load

resource "aws_ecs_cluster" "ec2_based" {
  name = "ec2-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"
  }
}

# Auto Scaling Group
resource "aws_autoscaling_group" "ecs_asg" {
  name            = "ecs-asg"
  min_size        = 2
  max_size        = 10
  desired_capacity = 2
  vpc_zone_identifier = [aws_subnet.private_1.id, aws_subnet.private_2.id]

  launch_template {
    id      = aws_launch_template.ecs_instances.id
    version = "$Latest"
  }

  tag {
    key                 = "Name"
    value               = "ecs-instance"
    propagate_launch_template = true
  }
}

# Cada EC2 precisa do ECS agent
resource "aws_launch_template" "ecs_instances" {
  name_prefix = "ecs-lt-"
  image_id    = data.aws_ami.ecs_ami.id  # Amazon ECS-optimized AMI
  instance_type = "t3.medium"  # Mínimo recomendado

  iam_instance_profile {
    arn = aws_iam_instance_profile.ecs_instance.arn
  }

  user_data = base64encode(templatefile("${path.module}/user_data.sh", {
    cluster_name = aws_ecs_cluster.ec2_based.name
  }))
}
```

**Trade-off EC2**:
- ✅ Controle fino, 24/7 baseline
- ❌ Gerenciamento operacional, custo fixo

---

## Service

O Service é o componente que **mantém o estado desejado**. Equivalente a um Kubernetes Deployment.

### Exemplo Completo

```hcl
resource "aws_ecs_service" "app" {
  name             = "minha-aplicacao-service"
  cluster          = aws_ecs_cluster.main.id
  task_definition  = aws_ecs_task_definition.app_production.arn
  desired_count    = 2  # HA mínima: 2 AZs
  launch_type      = "FARGATE"
  platform_version = "LATEST"  # Auto-updates com novos Fargate releases

  # Estratégia de Deploy (CRÍTICO)
  deployment_configuration {
    maximum_percent         = 200   # Permite 4 tasks durante deploy (2 old + 2 new)
    minimum_healthy_percent = 50    # Mantém 1 task healthy durante deploy
    
    # Automatic rollback se deployment falhar
    deployment_circuit_breaker {
      enable   = true
      rollback = true  # Volta para revision anterior automaticamente
    }
  }

  # Multiplicador de propagação de tags
  enable_ecs_managed_tags = true
  propagate_tags          = "TASK_DEFINITION"  # Task herda tags da definition

  # Rede (OBRIGATÓRIO para Fargate)
  network_configuration {
    subnets           = [
      aws_subnet.private_1.id,
      aws_subnet.private_2.id
    ]
    security_groups   = [aws_security_group.ecs_tasks.id]
    assign_public_ip  = false  # NUNCA exponha tasks diretamente
  }

  # Load Balancer (ALB ou NLB)
  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "minha-aplicacao"
    container_port   = 8080
  }

  # Grace period para tasks se registrarem no LB
  health_check_grace_period_seconds = 60

  # Scheduling strategy
  scheduling_strategy = "REPLICA"  # REPLICA (padrão) ou DAEMON (1 por nó)

  # Placement constraints (se usar EC2)
  # placement_constraints = [...]

  depends_on = [
    aws_lb_listener.app,
    aws_iam_role_policy.ecs_task_role_policy
  ]

  tags = {
    Environment = "production"
    Service     = "minha-aplicacao"
  }
}

# Security Group para tasks
resource "aws_security_group" "ecs_tasks" {
  name        = "ecs-tasks-sg"
  description = "Security group for ECS tasks"
  vpc_id      = aws_vpc.main.id

  # ✅ Ingress apenas do ALB
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]  # Apenas ALB
    description     = "From ALB"
  }

  # ✅ Egress para internet
  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "HTTPS to internet APIs"
  }

  # ✅ Egress para RDS
  egress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.rds.id]
    description     = "PostgreSQL"
  }

  tags = {
    Name = "ecs-tasks-sg"
  }
}

# Target Group (ALB)
resource "aws_lb_target_group" "app" {
  name        = "ecs-app-tg"
  port        = 8080
  protocol    = "HTTP"
  vpc_id      = aws_vpc.main.id
  target_type = "ip"  # OBRIGATÓRIO para Fargate

  health_check {
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
    path                = "/health"
    matcher             = "200"
  }

  tags = {
    Name = "ecs-app-tg"
  }
}
```

### Deployment Scenarios

```
Suponha: desired_count = 2, image v1.0 rodando

1️⃣ DEPLOY NOVO: v1.1
   T=0s:  Tasks v1.0 × 2, inicia v1.1 × 1 (max_percent = 200%)
   T=30s: v1.1 healthcheck passa, começa desligar v1.0
   T=60s: Todos com v1.1, deployment completo

2️⃣ FALHA NO DEPLOY: v1.1 health check falha
   T=30s: v1.1 não fica healthy, circuit breaker ativa
   T=35s: Rollback automático, v1.0 volta
   Result: Mitigação de falhas sem manual intervention

3️⃣ SCALING OUT: desired_count 2 → 4
   T=0s:  Inicia 2 tasks novas (replicação da task def atual)
   T=30s: Health check passa, já no ALB
   Custo IMEDIATO: +2 tasks
```

---

## Comparação: ECS vs Alternativas

### Matriz de Decisão

```
┌────────────┬──────────┬────────────┬──────────┬────────┐
│ Categoria  │ Lambda   │ ECS Fargate│ ECS EC2  │  EKS   │
├────────────┼──────────┼────────────┼──────────┼────────┤
│ Overhead   │ Mínimo   │ Baixo      │ Alto     │ MuitoAlto
│ Startup    │ 1-2s     │ 30s        │ 1m       │ 2-3m   │
│ Max Mem    │ 10GB     │ 30GB       │ 1TB+     │  Ilim  │
│ Max Dur    │ 15 min   │ Ilimitado  │ Ilimitado│ Ilim   │
│ Concur     │ 1000     │ Ilimitado  │ Limitado │ Ilim   │
│ Custo Min  │ ~$0      │ $5-10/mês  │ $30/mês  │ $73/mês│
│ Best For   │ Eventos  │ APIs, Jobs │ 24/7     │ Distrib│
└────────────┴──────────┴────────────┴──────────┴────────┘

Escolha baseada em:
1. Duração de workload (Lambda: < 15min)
2. Concorrência (Lambda: até 1000, depois ECS)
3. Estatefulidade (Lambda: stateless, ECS: tem estado)
4. Complexidade (Lambda: simples, ECS: média, EKS: complexa)
```

---

## Segurança em Produção

Segurança em ECS é uma cadeia: um elo fraco quebra tudo.

### 1. IAM Roles - Zero Credentials

```hcl
# ============================================
# EXECUTION ROLE (para ECS Agent, não container)
# ============================================

resource "aws_iam_role" "ecs_task_execution_role" {
  name = "ecsTaskExecutionRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
    }]
  })
}

# AWS managed policy (ECR pull, CloudWatch push)
resource "aws_iam_role_policy_attachment" "ecs_task_execution_policy" {
  role       = aws_iam_role.ecs_task_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# Permissão adicional: ler secrets
resource "aws_iam_role_policy" "ecs_execution_secrets" {
  name = "ecsExecutionSecretsPolicy"
  role = aws_iam_role.ecs_task_execution_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "kms:Decrypt"
        ]
        Resource = [
          aws_secretsmanager_secret.db_password.arn,
          aws_secretsmanager_secret.api_key.arn,
          aws_kms_key.secrets.arn
        ]
      }
    ]
  })
}

# ============================================
# TASK ROLE (para container app)
# ============================================

resource "aws_iam_role" "ecs_task_role" {
  name = "ecsTaskRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
    }]
  })
}

# Permissões específicas do app (LEAST PRIVILEGE)
resource "aws_iam_role_policy" "ecs_task_policy" {
  name = "ecsTaskPolicy"
  role = aws_iam_role.ecs_task_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "S3Access"
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject"
        ]
        Resource = "${aws_s3_bucket.app_data.arn}/uploads/*"
      },
      {
        Sid    = "DynamoDBAccess"
        Effect = "Allow"
        Action = [
          "dynamodb:Query",
          "dynamodb:GetItem",
          "dynamodb:PutItem"
        ]
        Resource = aws_dynamodb_table.app_table.arn
      },
      {
        Sid    = "RDSAccess"
        Effect = "Allow"
        Action = [
          "rds:DescribeDatabases"
        ]
        Resource = "*"
      },
      {
        Sid    = "CloudWatchLogs"
        Effect = "Allow"
        Action = [
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "${aws_cloudwatch_log_group.ecs_app.arn}:*"
      },
      {
        Sid    = "XRayWrite"
        Effect = "Allow"
        Action = [
          "xray:PutTraceSegments",
          "xray:PutTelemetryRecords"
        ]
        Resource = "*"
      }
    ]
  })
}

# Contas do IAM NÃO devem ser usadas em containers
# Se precisar de credenciais de terceiros, use Secrets Manager
```

### 2. Secrets Management

```hcl
resource "aws_secretsmanager_secret" "db_password" {
  name                    = "prod/rds/password"
  recovery_window_in_days = 7  # Grace period antes de delete

  tags = {
    Application = "minha-aplicacao"
    Rotation    = "manual"  # ou automática com Lambda
  }
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id = aws_secretsmanager_secret.db_password.id
  
  # NÃO hardcode! Usar variável do Terraform
  secret_string = jsonencode({
    username = "postgres"
    password = var.db_password  # De tfvars ou TFC
    host     = aws_db_instance.postgres.address
    port     = 5432
    dbname   = "prodapp"
  })
}

# Encryption com CMK customizada
resource "aws_kms_key" "secrets" {
  description             = "KMS key for ECS secrets"
  deletion_window_in_days = 10
  enable_key_rotation     = true

  tags = {
    Application = "minha-aplicacao"
  }
}

resource "aws_kms_alias" "secrets" {
  name          = "alias/ecs-secrets"
  target_key_id = aws_kms_key.secrets.key_id
}
```

### 3. Network Isolation

```hcl
# VPC com subnets privadas
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "app-vpc"
  }
}

# Subnets PRIVADAS para tasks
resource "aws_subnet" "private_1" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = data.aws_availability_zones.available.names[0]

  tags = {
    Name = "private-1"
  }
}

resource "aws_subnet" "private_2" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = data.aws_availability_zones.available.names[1]

  tags = {
    Name = "private-2"
  }
}

# NAT Gateway (para tasks acessar internet)
resource "aws_eip" "nat" {
  domain = "vpc"
  depends_on = [aws_internet_gateway.main]

  tags = {
    Name = "nat-eip"
  }
}

resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public_1.id

  tags = {
    Name = "nat-gw"
  }

  depends_on = [aws_internet_gateway.main]
}

# Route tables para privadas (via NAT)
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id
  }

  tags = {
    Name = "private-rt"
  }
}

resource "aws_route_table_association" "private_1" {
  subnet_id      = aws_subnet.private_1.id
  route_table_id = aws_route_table.private.id
}

resource "aws_route_table_association" "private_2" {
  subnet_id      = aws_subnet.private_2.id
  route_table_id = aws_route_table.private.id
}

# Security Group: Inbound restraints
resource "aws_security_group" "ecs_tasks" {
  name        = "ecs-tasks-sg"
  description = "Restrict inbound to ALB only"
  vpc_id      = aws_vpc.main.id

  # ✅ ONLY from ALB
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  # ✅ Outbound para dependências
  egress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.rds.id]
    description     = "To RDS"
  }

  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "To external APIs"
  }

  tags = {
    Name = "ecs-tasks-sg"
  }
}
```

---

## Auto Scaling

Auto scaling em ECS não é automático: PRECISA de configuração explícita.

### Target Tracking Scaling

```hcl
resource "aws_appautoscaling_target" "ecs_target" {
  max_capacity       = 20   # Limite de custo
  min_capacity       = 2    # Mínimo para HA (2 AZs)
  resource_id        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.app.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

# Escala baseada em CPU
resource "aws_appautoscaling_policy" "cpu" {
  name               = "cpu-autoscaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs_target.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs_target.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs_target.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value       = 70.0  # Manter CPU em 70%
    scale_in_cooldown  = 300   # 5 min antes de desescalar (evita flapping)
    scale_out_cooldown = 60    # 1 min antes de escalar
  }
}

# Escala baseada em memória
resource "aws_appautoscaling_policy" "memory" {
  name               = "memory-autoscaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs_target.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs_target.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs_target.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageMemoryUtilization"
    }
    target_value = 80.0
  }
}

# Escala baseada em requisições (ALB)
resource "aws_appautoscaling_policy" "request_count" {
  name               = "alb-request-count"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs_target.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs_target.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs_target.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ALBRequestCountPerTarget"
    }
    target_value = 1000.0  # 1000 req/min por task
  }
}
```

### Métricas Customizadas

```hcl
# Se CPU/memória/requests não bastam, use CloudWatch custom metrics

resource "aws_appautoscaling_policy" "custom_metric" {
  name               = "custom-metric-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs_target.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs_target.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs_target.service_namespace

  target_tracking_scaling_policy_configuration {
    customized_metric_specification {
      metric_dimension {
        name  = "MyCustomDimension"
        value = "MyValue"
      }
      metric_name = "MyCustomMetric"
      namespace   = "MyApp"
      statistic   = "Average"
      unit        = "Count"
    }
    target_value = 100.0
  }
}
```

### Armadilhas Comuns

```
❌ Scaling muito agressivo (cooldown = 0)
   Resultado: "Flapping" constante, custo explosivo, instabilidade

✅ Cooldown apropriado
   scale_in_cooldown  = 300s  (5 min)
   scale_out_cooldown = 60s   (1 min)
   Resultado: Estabilidade, custo previsível

❌ Múltiplas policies sem exclusão
   Resultado: Conflito de decisões

✅ Uma policy principal, outras complementares
   CPU rápido, memória lento
```

---

## CI/CD com ECS

### GitHub Actions + OIDC (Zero Credentials)

```yaml
name: Deploy to ECS

on:
  push:
    branches:
      - main
    paths:
      - 'src/**'
      - 'Dockerfile'
      - '.github/workflows/deploy-ecs.yml'

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: minha-aplicacao
  ECS_SERVICE: minha-aplicacao-service
  ECS_CLUSTER: producao-cluster
  ECS_TASK_DEFINITION: minha-aplicacao

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write  # OIDC token

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      # OIDC Federation (zero aws credentials!)
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsRole
          aws-region: ${{ env.AWS_REGION }}

      # Login ECR
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      # Build & Test
      - name: Build Docker image
        id: image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/${{ env.ECR_REPOSITORY }}:$IMAGE_TAG .
          docker tag $ECR_REGISTRY/${{ env.ECR_REPOSITORY }}:$IMAGE_TAG $ECR_REGISTRY/${{ env.ECR_REPOSITORY }}:latest
          echo "image=$ECR_REGISTRY/${{ env.ECR_REPOSITORY }}:$IMAGE_TAG" >> $GITHUB_OUTPUT

      # Push to ECR
      - name: Push image to ECR
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        run: |
          docker push ${{ steps.image.outputs.image }}
          docker push $ECR_REGISTRY/${{ env.ECR_REPOSITORY }}:latest

      # Update Task Definition
      - name: Update ECS task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: .aws/task-definition.json
          container-name: ${{ env.ECR_REPOSITORY }}
          image: ${{ steps.image.outputs.image }}

      # Deploy to ECS
      - name: Deploy to ECS service
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
          force-new-deployment: true

      # Notify Slack on success
      - name: Notify Slack
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "✅ ECS Deployment Success",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*ECS Deploy Successful*\nService: ${{ env.ECS_SERVICE }}\nImage: ${{ steps.image.outputs.image }}\nCommit: <https://github.com/${{ github.repository }}/commit/${{ github.sha }}|${{ github.sha }}>"
                  }
                }
              ]
            }

      # Notify Slack on failure
      - name: Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "❌ ECS Deployment Failed",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*ECS Deploy Failed*\nService: ${{ env.ECS_SERVICE }}\nCommit: <https://github.com/${{ github.repository }}/commit/${{ github.sha }}|${{ github.sha }}>\nCheck workflow: <https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}|here>"
                  }
                }
              ]
            }
```

### IAM Role para GitHub Actions

```hcl
# OIDC Provider
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]  # GitHub's thumbprint
}

# IAM Role para GitHub
resource "aws_iam_role" "github_actions" {
  name = "GitHubActionsRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.github.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
        }
        StringLike = {
          "token.actions.githubusercontent.com:sub" = "repo:seu-github-org/seu-repo:*"
        }
      }
    }]
  })
}

# Permissões necessárias para deploy
resource "aws_iam_role_policy" "github_actions_ecs" {
  name = "GitHubActionsECSPolicy"
  role = aws_iam_role.github_actions.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "ECRAccess"
        Effect = "Allow"
        Action = [
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:PutImage",
          "ecr:InitiateLayerUpload",
          "ecr:UploadLayerPart",
          "ecr:CompleteLayerUpload"
        ]
        Resource = "*"
      },
      {
        Sid    = "ECSDeployment"
        Effect = "Allow"
        Action = [
          "ecs:DescribeServices",
          "ecs:DescribeTaskDefinition",
          "ecs:DescribeTasks",
          "ecs:ListTasks",
          "ecs:UpdateService",
          "ecs:RegisterTaskDefinition"
        ]
        Resource = [
          aws_ecs_service.app.arn,
          aws_ecs_task_definition.app_production.arn
        ]
      },
      {
        Sid    = "PassRole"
        Effect = "Allow"
        Action = [
          "iam:PassRole"
        ]
        Resource = [
          aws_iam_role.ecs_task_role.arn,
          aws_iam_role.ecs_task_execution_role.arn
        ]
      }
    ]
  })
}
```

---

## Well-Architected Framework

Alinhamento com os 5 pilares da AWS:

### 1. Operational Excellence
- CloudWatch Logs centralizado
- Container Insights habilitado
- ECS Exec para debugging de tasks
- Deployment automático com rollback
- Tagging consistente

### 2. Security
- Task Roles com least privilege
- Secrets Manager para credentials
- VPC privado com NAT Gateway
- Security Groups restritivos
- Network isolation entre serviços
- Scanning de imagem ECR

### 3. Reliability
- Multi-AZ deployment (min 2 AZs)
- Health checks em task e ALB
- Auto-scaling para HA
- Deployment circuit breaker
- Graceful shutdown strategy

### 4. Performance Efficiency
- Fargate para workloads variáveis
- Container Insights para observabilidade
- ALB para distribuição de carga
- Caching na aplicação

### 5. Cost Optimization
- Fargate Spot para non-critical
- Capacity Providers strategy
- Auto-scaling baseada em demanda
- Resource reservation apropriada
- Cloudwatch Logs retention policy

---

## Troubleshooting Comum

### Task não inicia ("STOPPED")

```bash
# Ver logs
aws logs tail /ecs/minha-aplicacao --follow

# Ver service events
aws ecs describe-services \
  --cluster producao-cluster \
  --services minha-aplicacao-service

# Ver task details
aws ecs describe-tasks \
  --cluster producao-cluster \
  --tasks <task-arn>
```

**Causas Comuns**:
- Imagem não existe no ECR (verificar image URI)
- Task Role sem permissões de ECR
- Container falha no startup (health check timeout)
- Memory/CPU insuficiente

### Task falha em health check

```dockerfile
# Dockerfile - endpoint /health deve ser rápido
FROM python:3.11-slim

COPY app.py .
RUN pip install flask

HEALTHCHECK --interval=30 --timeout=5 --retries=2 --start-period=60 \
  CMD curl -f http://localhost:8080/health || exit 1

CMD ["python", "app.py"]
```

### Custos inesperados

```
Scenario 1: Tasks rodam 24/7 em Fargate
- 2 tasks × 256 CPU × 512 MB × 730 horas/mês × $0.04/hour = ~$60/mês

Fix: Use Fargate Spot para non-critical, scale down à noite

Scenario 2: Auto-scaling flapping
- Scale out 10 tasks → scale in 5 → scale out 8 → repeat
- Resultado: Custo 2-3x higher

Fix: Aumentar cooldown (scale_in_cooldown = 300s)

Scenario 3: CloudWatch Logs retention infinita
- Logs crescem sem limite, storage explode

Fix: Definir retention_in_days = 30
```

---

## Recursos Adicionais

- [AWS ECS Documentation](https://docs.aws.amazon.com/ecs/)
- [ECS Best Practices](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/best_practices.html)
- [Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [ECS Workshop](https://www.ecsworkshop.com/)

---
