# 🔍 Comparativo: Cluster vs Service vs Task

> **Objetivo**: Esclarecer os 3 componentes fundamentais do ECS e suas responsabilidades distintas.

---

## 📊 Tabela Comparativa Rápida

| Aspecto | **Cluster** | **Service** | **Task Definition** |
|---------|-----------|-----------|----------------------|
| **O que é?** | Agrupamento lógico de recursos | Gerenciador de estado (desired state) | Blueprint/template de container |
| **Escopo** | Múltiplas services, múltiplos apps | Uma única aplicação | Uma única versão de imagem |
| **Responsabilidade** | Infra + Networking | Availability + Scaling | Container spec |
| **Imutável?** | ❌ Dinâmico | ❌ Dinâmico | ✅ Sim (cada versão é nova) |
| **Ciclo de vida** | Persiste indefinidamente | Persiste enquanto existir | Versionado (draft/active) |
| **Escalabilidade** | N/A (é o "container") | ✅ Principal responsável | N/A |
| **Gerencia tasks?** | ❌ Não gerencia | ✅ Sim (cria/destrói/monitora) | ❌ Não gerencia |
| **Múltiplas versões ativas?** | N/A | ❌ Uma por vez | ✅ Sim (histórico) |
| **Quantidade por app** | 1+ clusters | 1 service por app | 1+ task definitions por app |
| **Análogo ao** | Data center / Kubernetes Node | systemd / Kubernetes Deployment | Docker image configuration |

---

## 🎯 Responsabilidades e Papéis

### 1. **TASK DEFINITION** — O Blueprint

```
┌─────────────────────────────────────┐
│      Task Definition v1.2.3         │
├─────────────────────────────────────┤
│  • Image: myapp:v1.2.3              │
│  • CPU: 256                         │
│  • Memory: 512 MB                   │
│  • Port: 8080                       │
│  • Environment variables            │
│  • IAM Task Role                    │
│  • Log configuration                │
│  • Health check definition          │
│  • Volume mounts                    │
└─────────────────────────────────────┘
```

**Características:**
- ✅ **Immutable** — Cada mudança = nova revisão
- ✅ **Versionado** — Histórico completo mantido
- ✅ **Reutilizável** — Uma Task Definition pode ser usada em múltiplas Services
- ❌ **Não executa nada** — É apenas a especificação

**Exemplo:**
```hcl
resource "aws_ecs_task_definition" "app" {
  family                   = "meu-app"           # Nome do blueprint
  
  # Versão: v1 = primeira revisão, v2 = segunda mudança, etc.
  # AWS gerencia automaticamente
  
  requires_compatibilities = ["FARGATE"]
  network_mode             = "awsvpc"
  cpu                      = "256"
  memory                   = "512"
  
  container_definitions = jsonencode([{
    name      = "app-container"
    image     = "123456789.dkr.ecr.us-east-1.amazonaws.com/meu-app:v1.2.3"
    essential = true
    portMappings = [{
      containerPort = 8080
      hostPort      = 8080
    }]
    logConfiguration = {
      logDriver = "awslogs"
      options = {
        "awslogs-group"         = aws_cloudwatch_log_group.app.name
        "awslogs-region"        = "us-east-1"
        "awslogs-stream-prefix" = "ecs"
      }
    }
    environment = [
      { name = "ENV", value = "production" }
    ]
  }])
}
```

**Quando mudar Task Definition:**
- ✅ Atualizar versão de imagem
- ✅ Mudar variáveis de ambiente
- ✅ Adicionar volumes
- ✅ Alterar alocação de CPU/memory
- ✅ Atualizar health checks
- ✅ Mudar logging

**Quando NÃO criar nova Task Definition:**
- Se apenas mudar número de tasks → use Service scaling
- Se apenas mudar replicas → use Service scaling

---

### 2. **CLUSTER** — O Container (Infra)

```
┌──────────────────────────────────────┐
│        ECS Cluster: "production"     │
├──────────────────────────────────────┤
│                                      │
│  ┌───────────────────────────────┐  │
│  │  Fargate Capacity Provider    │  │
│  │  (Serverless)                 │  │
│  └───────────────────────────────┘  │
│                                      │
│  ┌───────────────────────────────┐  │
│  │  EC2 Capacity Provider        │  │
│  │  (3 instâncias)               │  │
│  └───────────────────────────────┘  │
│                                      │
│  ┌───────────────────────────────┐  │
│  │  ASG Min: 3, Max: 10          │  │
│  │  Services rodam aqui          │  │
│  └───────────────────────────────┘  │
│                                      │
└──────────────────────────────────────┘
```

**Características:**
- 🏗️ **Define infraestrutura** — Fargate, EC2, ou hybrid
- 🌐 **Agrupa networking** — VPC, subnets, security groups
- 📊 **Compartilhado** — Múltiplas services podem usar mesmo cluster
- 🔧 **Configuração estática** — Muda raramente em produção
- 🎯 **Escopo de custos** — Você paga por EC2/Fargate nesse cluster

**Exemplo Fargate:**
```hcl
resource "aws_ecs_cluster" "production" {
  name = "production"
  
  setting {
    name  = "containerInsights"
    value = "enabled"  # Monitoramento detalhado
  }
}

# Capacity Provider Fargate (automático)
resource "aws_ecs_cluster_capacity_providers" "production" {
  cluster_name = aws_ecs_cluster.production.name
  
  capacity_providers = ["FARGATE", "FARGATE_SPOT"]  # Fargate automático
  
  default_capacity_provider_strategy {
    base              = 1      # Sempre FARGATE
    weight            = 100
    capacity_provider = "FARGATE"
  }
  
  default_capacity_provider_strategy {
    weight            = 100
    capacity_provider = "FARGATE_SPOT"  # Excedente em SPOT (60% desconto)
  }
}
```

**Exemplo EC2:**
```hcl
resource "aws_ecs_cluster" "prod_ec2" {
  name = "production-ec2"
}

# Capacity Provider EC2 (manual)
resource "aws_launch_configuration" "ecs_instances" {
  image_id      = "ami-0c55b159cbfafe1f0"  # ECS-optimized AMI
  instance_type = "t3.large"
  iam_instance_profile = aws_iam_instance_profile.ecs.name
  
  user_data = base64encode(templatefile("${path.module}/ecs-init.sh", {
    cluster_name = aws_ecs_cluster.prod_ec2.name
  }))
}

resource "aws_autoscaling_group" "ecs_asg" {
  name                = "ecs-asg"
  launch_configuration = aws_launch_configuration.ecs_instances.name
  min_size            = 3
  max_size            = 10
  vpc_zone_identifier = var.private_subnets
}

resource "aws_ecs_capacity_provider" "ec2_provider" {
  name = "ec2-provider"
  
  auto_scaling_group_provider {
    auto_scaling_group_arn         = aws_autoscaling_group.ecs_asg.arn
    managed_termination_protection = "ENABLED"
    
    managed_scaling {
      maximum_scaling_step_size = 1000
      minimum_scaling_step_size = 1
      status                    = "ENABLED"
      target_capacity           = 75  # Manter 75% de utilização
    }
  }
}
```

**Decisões no Cluster:**
- **Fargate** → Sem gerenciar instâncias
- **EC2** → Controle total, customização
- **Hybrid** → Fargate para normal, EC2 para workloads específicas
- **VPC** → Privada ou pública (sempre privada em produção)
- **Subnets** → Multi-AZ para HA

---

### 3. **SERVICE** — O Gerenciador

```
┌───────────────────────────────────────────┐
│   Service: "meu-app-service"              │
├───────────────────────────────────────────┤
│                                           │
│  Desired State: 3 tasks                   │
│  Current State: 3 tasks running           │
│  Task Definition: familia:v5              │
│                                           │
│  ┌────────────────────────────────────┐  │
│  │  Task 1 [Running]                  │  │
│  │  IP: 10.0.1.45:8080                │  │
│  │  az: us-east-1a                    │  │
│  └────────────────────────────────────┘  │
│                                           │
│  ┌────────────────────────────────────┐  │
│  │  Task 2 [Running]                  │  │
│  │  IP: 10.0.2.120:8080               │  │
│  │  az: us-east-1b                    │  │
│  └────────────────────────────────────┘  │
│                                           │
│  ┌────────────────────────────────────┐  │
│  │  Task 3 [Running]                  │  │
│  │  IP: 10.0.3.89:8080                │  │
│  │  az: us-east-1c                    │  │
│  └────────────────────────────────────┘  │
│                                           │
│  Load Balancer: ALB (port 80→8080)       │
│                                           │
└───────────────────────────────────────────┘
```

**Características:**
- 🎯 **Gerencia desired state** — "Quero 3 tasks rodando"
- 🔄 **Auto-recovery** — Task morre? Service inicia nova
- 📈 **Auto-scaling** — Escala de 1 a 100 tasks
- 🚀 **Deployment strategies** — Rolling, Blue/Green, Canary
- 🎪 **Load balancing** — ALB/NLB integrado
- 🔍 **Health checks** — Monitora e substitui tasks não-saudáveis

**Exemplo Completo:**
```hcl
resource "aws_ecs_service" "app" {
  name            = "meu-app-service"
  cluster         = aws_ecs_cluster.production.id
  task_definition = "${aws_ecs_task_definition.app.family}:${aws_ecs_task_definition.app.revision}"
  
  # Scaling
  desired_count                      = 3  # Inicial
  deployment_minimum_healthy_percent = 100
  deployment_maximum_percent         = 200  # Blue/Green
  launch_type                        = "FARGATE"
  
  # Networking
  network_configuration {
    subnets          = var.private_subnets
    security_groups  = [aws_security_group.app.id]
    assign_public_ip = false  # Privado
  }
  
  # Load balancing
  load_balancer {
    target_group_arn = aws_lb_target_group.app.arn
    container_name   = "app-container"
    container_port   = 8080
  }
  
  # Health check
  health_check_grace_period_seconds = 60
  
  # Deployment strategy
  deployment_controller {
    type = "ECS"  # ou CODE_DEPLOY para Canary
  }
  
  # Logging
  enable_execute_command = true  # Para debugging via ECS Exec
  
  depends_on = [
    aws_lb_listener.app,
    aws_iam_role_policy.ecs_task_execution_policy
  ]
  
  lifecycle {
    ignore_changes = [desired_count]  # Ignorar scaling auto
  }
}

# Auto Scaling
resource "aws_appautoscaling_target" "app" {
  max_capacity       = 10
  min_capacity       = 3
  resource_id        = "service/${aws_ecs_cluster.production.name}/${aws_ecs_service.app.name}"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "app_memory" {
  policy_name               = "app-memory-scaling"
  policy_type               = "TargetTrackingScaling"
  resource_id               = aws_appautoscaling_target.app.resource_id
  scalable_dimension        = aws_appautoscaling_target.app.scalable_dimension
  service_namespace         = aws_appautoscaling_target.app.service_namespace
  
  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageMemoryUtilization"
    }
    target_value = 70.0
  }
}

resource "aws_appautoscaling_policy" "app_cpu" {
  policy_name               = "app-cpu-scaling"
  policy_type               = "TargetTrackingScaling"
  resource_id               = aws_appautoscaling_target.app.resource_id
  scalable_dimension        = aws_appautoscaling_target.app.scalable_dimension
  service_namespace         = aws_appautoscaling_target.app.service_namespace
  
  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value = 70.0
  }
}
```

**Decisões no Service:**
- **Desired count** → Quantas tasks always-on
- **Deployment strategy** → Rolling, Blue/Green (0 downtime)
- **Health checks** → Que define "task saudável"
- **Load balancer** → ALB (layer 7) ou NLB (layer 4)
- **Auto scaling** → CPU, memory, custom metrics
- **Placement strategy** → Spread across AZs

---

## 🔗 Relacionamento entre os 3

### Fluxo Conceitual

```
┌──────────────────────┐
│  Task Definition     │
│  (blueprint)         │
│  family: "app"       │
│  version: 5          │
└──────────────────────┘
           │
           │ "usa"
           ▼
┌──────────────────────┐      ┌────────────────────┐
│  Service             │──────│  Cluster           │
│  (desired state)     │      │  (infra)           │
│  desired_count: 3    │      │  type: Fargate     │
│  task_def: app:5     │      │  vpc: vpc-xyz      │
└──────────────────────┘      └────────────────────┘
           │
           │ "cria/gerencia"
           ▼
┌──────────────────────┐
│  Tasks (instâncias)  │
│  Task 1 [Running]    │
│  Task 2 [Running]    │
│  Task 3 [Running]    │
└──────────────────────┘
```

### Analogia do Mundo Real

| Componente | Analogia |
|-----------|----------|
| **Task Definition** | Receita de bolo (ingredientes, modo de preparo, tempo) |
| **Cluster** | Cozinha (equipamentos, espaço, gás) |
| **Service** | Padaria com 3 fornos ligados (garante sempre ter pão fresco) |

---

## 📋 Checklist de Decisão

### Preciso atualizar a Task Definition? ✅

```
Mude a Task Definition se:
  ✅ Versão da imagem muda (v1.0.0 → v1.1.0)
  ✅ Variáveis de ambiente mudam
  ✅ CPU/Memory allocation muda
  ✅ Health check muda
  ✅ Volume/storage muda
  
NÃO mude se:
  ❌ Apenas quer mais/menos réplicas → use Service desired_count
  ❌ Apenas quer mudar load balancer → use Service load_balancer block
  ❌ Apenas quer mudar security groups → use Service network_configuration
```

### Preciso alterar o Cluster? ❌ (raro)

```
Mude o Cluster se:
  ✅ Trocar tipo de compute (Fargate → EC2)
  ✅ Mudar VPC/subnets (migração de rede)
  ✅ Alterar capacity providers
  
NÃO mude se:
  ❌ Apenas quer escalar tasks → use Service auto scaling
  ❌ Apenas quer atualizar imagem → use Task Definition
```

### Preciso alterar o Service? ✅ (frequente)

```
Mude o Service se:
  ✅ Atualizar Task Definition (nova versão)
  ✅ Mudar desired count
  ✅ Habilitar/desabilitar auto scaling
  ✅ Trocar load balancer
  ✅ Alterar deployment strategy
  ✅ Mudar health checks de aplicação
```

---

## 🚀 Exemplo End-to-End

### Cenário: Deploy de uma aplicação web

#### Passo 1: Task Definition (qual container rodar)
```hcl
# Definir COMO a app vai rodar
resource "aws_ecs_task_definition" "api_v2" {
  family = "api"
  cpu    = "256"
  memory = "512"
  
  container_definitions = jsonencode([{
    name      = "api-container"
    image     = "myrepo/api:v2.0.0"  # ← Nova versão
    essential = true
    portMappings = [{
      containerPort = 8000
    }]
  }])
}
```

#### Passo 2: Cluster (onde rodar)
```hcl
# Definir ONDE a app vai rodar
resource "aws_ecs_cluster" "prod" {
  name = "production"
}

# Infra: Fargate automático
resource "aws_ecs_cluster_capacity_providers" "prod" {
  cluster_name       = aws_ecs_cluster.prod.name
  capacity_providers = ["FARGATE", "FARGATE_SPOT"]
}
```

#### Passo 3: Service (garantir que roda)
```hcl
# Garantir que SEMPRE está rodando
resource "aws_ecs_service" "api" {
  name            = "api-service"
  cluster         = aws_ecs_cluster.prod.id
  task_definition = "api:2"              # ← Usa Task Def v2
  desired_count   = 5
  
  load_balancer {
    target_group_arn = aws_lb_target_group.api.arn
    container_name   = "api-container"
    container_port   = 8000
  }
}
```

#### Fluxo de execução:
1. Task Definition diz: "execute myrepo/api:v2.0.0 com CPU=256, mem=512"
2. Service diz: "quero 5 instâncias dessa Task Definition rodando sempre"
3. Cluster diz: "ok, vou rodar essas 5 tasks aqui na infraestrutura Fargate"
4. Se uma task morrer → Service cria uma nova
5. Se CPU > 80% → Auto scaling cria mais tasks (até max 10)

---

## 🎓 Comparação com Kubernetes

Para quem conhece Kubernetes:

| ECS | Kubernetes |
|-----|-----------|
| **Task Definition** | Pod manifest / Deployment spec |
| **Cluster** | Kubernetes Cluster (nodes) |
| **Service** | Kubernetes Deployment / StatefulSet |
| **Task** | Pod |

---

## ⚠️ Erros Comuns

### ❌ Erro 1: Criar nova Task Definition a cada scaling
```hcl
# ERRADO ❌
resource "aws_ecs_service" "app" {
  desired_count = var.desired_count  # Essa mudança
}
# E depois criar nova Task Definition

# CORRETO ✅
# Deixe Task Definition fixa
# Apenas mude desired_count do Service
```

### ❌ Erro 2: Não entender imutabilidade de Task Definition
```hcl
# ERRADO ❌
resource "aws_ecs_task_definition" "app" {
  family = "my-app"  # Sem versionamento
  # Tentar "editar" existente
}

# CORRETO ✅
resource "aws_ecs_task_definition" "app" {
  family = "my-app"
  # AWS gerencia automaticamente: v1, v2, v3...
  # Cada Terraform apply = nova revisão (se houver mudança)
}
```

### ❌ Erro 3: Confundir responsabilidades
```
ERRADO: "Vou usar Cluster para controlar quantas tasks rodam"
CORRETO: "Vou usar Service para controlar quantas tasks rodam"

ERRADO: "Vou usar Service para configurar qual imagem rodar"
CORRETO: "Vou usar Task Definition para configurar qual imagem rodar"
```

---

## 📈 Quando Escalar Cada Um

```
┌──────────────────────────────────────────┐
│ Tráfego aumenta 10x                      │
├──────────────────────────────────────────┤
│                                          │
│ 1. AUTO SCALING do Service               │
│    ↓ (min 3 → max 30 tasks)              │
│                                          │
│ 2. Service hits capacity do Cluster      │
│    ↓ Auto Scaling do Cluster (EC2)       │
│       ou Auto Scaling do ASG             │
│                                          │
│ 3. Mudar Task Definition (mais recursos) │
│    ↓ se tasks não têm CPU suficiente     │
│                                          │
└──────────────────────────────────────────┘
```

---

## 🎯 Resumo Executivo

| Pergunta | Resposta |
|----------|----------|
| **O que é Task Definition?** | Especificação imutável de como rodar um container |
| **O que é Cluster?** | Agrupamento de infraestrutura (Fargate/EC2) |
| **O que é Service?** | Gerenciador que garante N tasks rodando |
| **Qual muda mais frequentemente?** | Service > Task Definition >> Cluster |
| **Qual é imutável?** | Task Definition (versionada) |
| **Qual gerencia tasks?** | Service |
| **Qual define hardware?** | Task Definition (CPU/memory) e Cluster (tipo de compute) |
| **Qual controla escala?** | Service (desired count + auto scaling) |
| **Quantos por aplicação?** | Task Def: 1+, Cluster: 1+, Service: 1 por aplicação |

---

## 🔗 Próximas Leituras

- [02-arquitetura-componentes.md](02-arquitetura-componentes.md) — Detalhes técnicos profundos
- [05-auto-scaling.md](05-auto-scaling.md) — Como escalar Service automaticamente
- [06-cicd-github-actions.md](06-cicd-github-actions.md) — Pipeline para atualizar Task Definition
- [04-seguranca-producao.md](04-seguranca-producao.md) — Roles e permissões em cada componente
