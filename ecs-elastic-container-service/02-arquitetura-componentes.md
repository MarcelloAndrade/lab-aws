# 🏗️ Arquitetura e Componentes de ECS

## Visão de Alto Nível

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

## Os Três Componentes Principais

### 1️⃣ **Task Definition**
Define como um container será executado. É um **blueprint imutável**.

```hcl
# Analogia: Task Definition = Docker Compose file versionado
# Cada alteração cria uma nova revisão
```

### 2️⃣ **Cluster**
Agrupamento lógico de recursos. Abstrai se é Fargate ou EC2.

```hcl
# Analogia: Cluster = Grupo de máquinas (físico ou lógico)
# ECS gerencia automaticamente naquele cluster
```

### 3️⃣ **Service**
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
