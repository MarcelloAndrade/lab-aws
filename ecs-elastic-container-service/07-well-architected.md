# 🏛️ Well-Architected Framework

Alinhamento com os 5 pilares da AWS para arquitetura ECS em produção.

## 1. Operational Excellence

**Objetivo**: Executar e monitorar sistemas para entregar valor e melhorar continuamente.

### Observabilidade

```hcl
# Container Insights para visibilidade em tempo real
resource "aws_ecs_cluster" "main" {
  name = "producao-cluster"

  setting {
    name  = "containerInsights"
    value = "enabled"  # OBRIGATÓRIO
  }
}

# CloudWatch Logs centralizado
resource "aws_cloudwatch_log_group" "ecs_app" {
  name              = "/ecs/minha-aplicacao"
  retention_in_days = 30

  tags = {
    Application = "minha-aplicacao"
    Purpose     = "operational-logs"
  }
}

# Custom Metrics para insights de negócio
resource "aws_cloudwatch_log_group" "ecs_app_metrics" {
  name              = "/ecs/minha-aplicacao/metrics"
  retention_in_days = 7  # Curta retenção para reduzir custo
}
```

### Deployment Estratégia

```hcl
# Automatic rollback em falhas
resource "aws_ecs_service" "app" {
  # ...
  
  deployment_configuration {
    deployment_circuit_breaker {
      enable   = true
      rollback = true  # CRÍTICO: Volta automaticamente em falha
    }
  }
}
```

### Tagging e Cost Allocation

```hcl
locals {
  common_tags = {
    Environment = "production"
    ManagedBy   = "terraform"
    CostCenter  = "platform"
    Application = "minha-aplicacao"
    Team        = "platform-team"
  }
}

resource "aws_ecs_service" "app" {
  # ...
  tags = merge(local.common_tags, {
    Component = "api-service"
  })
}
```

---

## 2. Security

**Objetivo**: Proteger dados, systems, assets usando Access Control e encryption.

### Identity & Access Management

```hcl
# Execution Role: ECS Agent (pull images, push logs)
resource "aws_iam_role" "ecs_task_execution_role" {
  name = "ecsTaskExecutionRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal.Service = "ecs-tasks.amazonaws.com"
      Action = "sts:AssumeRole"
    }]
  })
}

# Task Role: Container Application (app permissions)
resource "aws_iam_role" "ecs_task_role" {
  name = "ecsTaskRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal.Service = "ecs-tasks.amazonaws.com"
      Action = "sts:AssumeRole"
    }]
  })
}

# Least Privilege: Apenas o necessário
resource "aws_iam_role_policy" "ecs_task_policy" {
  name = "ecsTaskPolicy"
  role = aws_iam_role.ecs_task_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = ["s3:GetObject"]
        Resource = "${aws_s3_bucket.app_data.arn}/public/*"
      },
      {
        Effect = "Allow"
        Action = ["dynamodb:Query"]
        Resource = aws_dynamodb_table.app_table.arn
      }
    ]
  })
}
```

### Data Protection

```hcl
# Encryption at rest com KMS
resource "aws_kms_key" "secrets" {
  description             = "KMS key for ECS secrets"
  deletion_window_in_days = 10
  enable_key_rotation     = true
}

resource "aws_secretsmanager_secret" "db_password" {
  name                    = "prod/rds/password"
  recovery_window_in_days = 7

  tags = {
    Application = "minha-aplicacao"
    Encryption  = "kms"
  }
}

# Secrets no container (não environment variables)
container_definitions = jsonencode([{
  secrets = [
    {
      name      = "DB_PASSWORD"
      valueFrom = aws_secretsmanager_secret.db_password.arn
    }
  ]
}])
```

### Network Security

```hcl
# VPC privado
resource "aws_ecs_service" "app" {
  network_configuration {
    subnets          = [aws_subnet.private_1.id, aws_subnet.private_2.id]
    security_groups  = [aws_security_group.ecs_tasks.id]
    assign_public_ip = false  # NUNCA exponha tasks
  }
}

# Security Group restritivo
resource "aws_security_group" "ecs_tasks" {
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]  # APENAS ALB
  }

  egress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.rds.id]  # Específico
  }
}
```

---

## 3. Reliability

**Objetivo**: Recuperar-se de falhas e funcionar com consistência conforme esperado.

### Design for Failure

```hcl
# Multi-AZ mínima
resource "aws_ecs_service" "app" {
  desired_count = 2  # Mínimo para HA entre 2 AZs

  network_configuration {
    subnets = [
      aws_subnet.private_1.id,      # AZ-1
      aws_subnet.private_2.id       # AZ-2
    ]
  }
}

# Spread tasks entre AZs
resource "aws_ecs_service" "app" {
  placement_constraints {
    type       = "distinctInstance"      # Máx 1 task por EC2
    expression = "attribute:ecs.os-type =~ Linux"
  }
}
```

### Resilience Patterns

```hcl
# Health checks: task + ALB
healthCheck = {
  command     = ["CMD-SHELL", "curl -f http://localhost:8080/health || exit 1"]
  interval    = 30
  timeout     = 5
  retries     = 2
  startPeriod = 60  # Grace period
}

# ALB health check
resource "aws_lb_target_group" "app" {
  health_check {
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 30
    path                = "/health"
    matcher             = "200"
  }
}

# Auto Scaling para alta demanda
resource "aws_appautoscaling_target" "ecs_target" {
  min_capacity = 2   # HA mínima
  max_capacity = 20  # Limite de custo
}
```

### Graceful Shutdown

```hcl
# Grace period para SIGTERM antes de SIGKILL
stopTimeout = 120  # 120 segundos

# Task definition readiness: conheça seu startup
startPeriod = 60   # 60 segundos de grace period
```

---

## 4. Performance Efficiency

**Objetivo**: Usar recursos computacionais eficientemente para atender demanda.

### Right Sizing

```hcl
# Fargate é ideal para variabilidade
# Pague apenas pelo que usa

resource "aws_ecs_task_definition" "app_production" {
  cpu    = "256"   # 0.25 vCPU
  memory = "512"   # 512 MB
  
  # Ajustar conforme observações de Container Insights
}

# Auto-scale baseado em demanda
resource "aws_appautoscaling_policy" "cpu" {
  target_tracking_scaling_policy_configuration {
    target_value       = 70.0  # Manter CPU em 70%
    scale_out_cooldown = 60
    scale_in_cooldown  = 300
  }
}
```

### Caching

```hcl
# Application-level caching
# ElastiCache (Redis) para sessões, rate limits
# S3 CloudFront para assets

resource "aws_elasticache_cluster" "session_cache" {
  cluster_id           = "session-cache"
  engine               = "redis"
  node_type            = "cache.t3.micro"
  num_cache_nodes      = 1
  parameter_group_name = "default.redis7"
  engine_version       = "7.0"
  port                 = 6379

  tags = {
    Application = "minha-aplicacao"
  }
}
```

---

## 5. Cost Optimization

**Objetivo**: Eliminar gastos desnecessários e otimizar recursos.

### Fargate Spot

```hcl
resource "aws_ecs_cluster_capacity_providers" "main" {
  cluster_name = aws_ecs_cluster.main.name

  capacity_providers = ["FARGATE", "FARGATE_SPOT"]

  default_capacity_provider_strategy {
    base              = 1  # Sempre 1 on-demand
    weight            = 100
    capacity_provider = "FARGATE"
  }

  default_capacity_provider_strategy {
    weight            = 100
    capacity_provider = "FARGATE_SPOT"  # Overflow aqui = 70% discount
  }
}
```

**Economia Real**:
- 2 tasks on FARGATE: ~$67/mês
- 1 on FARGATE + 1 on SPOT: ~$43/mês (36% savings)

### CloudWatch Logs Retention

```hcl
resource "aws_cloudwatch_log_group" "ecs_app" {
  name              = "/ecs/minha-aplicacao"
  retention_in_days = 30  # Não infinito!

  # Custo: ~$0.50 per GB-month
  # 1 serviço × 100 MB/dia × 30 dias = 3GB = $1.50/mês
}
```

### Resource Reservation

```hcl
# Não over-provision
cpu    = "256"   # Baseado em observações
memory = "512"   # Não máximo disponível

# Auto-scale permite pagar por uso real
resource "aws_appautoscaling_target" "ecs_target" {
  min_capacity = 2
  max_capacity = 20  # Limite máximo = controle de custo
}
```

---

## Checklist Completo

### Operational Excellence
- [ ] Container Insights habilitado
- [ ] CloudWatch Logs com retention apropriado
- [ ] ECS Exec para debugging de tasks
- [ ] Deployment automático com circuit breaker
- [ ] Tagging consistente para cost allocation

### Security
- [ ] Execution Role com ECR + CloudWatch
- [ ] Task Role com least privilege
- [ ] Secrets via Secrets Manager (não env vars)
- [ ] VPC privado com NAT Gateway
- [ ] Security Groups restritivos
- [ ] No hardcoded credentials

### Reliability
- [ ] Multi-AZ deployment (min 2)
- [ ] Health checks (task + ALB)
- [ ] Auto-scaling configured
- [ ] Deployment circuit breaker enabled
- [ ] Graceful shutdown strategy

### Performance Efficiency
- [ ] Right-sized container resources
- [ ] Auto-scaling based on metrics
- [ ] Caching strategy (ElastiCache/CloudFront)
- [ ] Container Insights monitoring

### Cost Optimization
- [ ] Fargate Spot para non-critical
- [ ] CloudWatch Logs retention limits
- [ ] Resource right-sizing
- [ ] Auto-scaling to match demand
- [ ] No over-provisioning
