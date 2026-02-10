# 🔧 Troubleshooting Comum

## Task não inicia ("STOPPED")

### Diagnóstico Rápido

```bash
# Ver logs de erro
aws logs tail /ecs/minha-aplicacao --follow

# Detalhes da service (últimas ações)
aws ecs describe-services \
  --cluster producao-cluster \
  --services minha-aplicacao-service \
  --query 'services[0].events'

# Detalhes da task individual
aws ecs describe-tasks \
  --cluster producao-cluster \
  --tasks <task-arn> \
  --query 'tasks[0].{Task:taskArn, Status:lastStatus, Error:stoppedReason}'
```

### Causas Comuns e Soluções

#### 1. Imagem não existe no ECR

**Sintoma**: `stoppedReason: "CannotPullContainerImage"`

```bash
# Verificar se a image existe
aws ecr describe-images \
  --repository-name minha-aplicacao \
  --image-ids imageTag=v1.2.3

# Verificar URI corretamente formatada
# Correto: 123456789.dkr.ecr.us-east-1.amazonaws.com/minha-aplicacao:v1.2.3
# Errado:  minha-aplicacao:latest
```

**Fix**:
```hcl
# ✅ CORRETO
image = "123456789.dkr.ecr.${var.aws_region}.amazonaws.com/${var.ecr_repository}:${var.image_tag}"

# ❌ ERRADO
image = "minha-aplicacao"
image = "minha-aplicacao:latest"
```

#### 2. Execution Role sem permissão de ECR

**Sintoma**: `stoppedReason: "AccessDenied"`

```bash
# Verificar policy do execution role
aws iam get-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-name AmazonECSTaskExecutionRolePolicy
```

**Fix**:
```hcl
resource "aws_iam_role_policy_attachment" "ecs_task_execution_policy" {
  role       = aws_iam_role.ecs_task_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}
```

#### 3. Container falha no startup

**Sintoma**: Task inicia mas falha após alguns segundos

**Causa**: Application crasha, porta errada, memory insuficente

```bash
# Ver logs da aplicação
aws logs get-log-events \
  --log-group-name /ecs/minha-aplicacao \
  --log-stream-name ecs/minha-aplicacao/$(aws ecs task-id)
```

**Fix**: Garantir que a aplicação está pronta:
```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt

COPY app.py .

# CRÍTICO: Porta deve matches portMappings
EXPOSE 8080

# Health check integrado
HEALTHCHECK --interval=30 --timeout=5 --retries=2 --start-period=60 \
  CMD curl -f http://localhost:8080/health || exit 1

# Não execute como root
USER nobody

CMD ["python", "app.py"]
```

#### 4. Memory insuficiente

**Sintoma**: Task inicia mas é killed após minutos

```bash
# Monitorar via Container Insights
aws cloudwatch get-metric-statistics \
  --namespace ECS/ContainerInsights \
  --metric-name MemoryUtilized \
  --dimensions Name=ServiceName,Value=minha-aplicacao-service \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average,Maximum
```

**Fix**: Aumentar memory na task definition:
```hcl
resource "aws_ecs_task_definition" "app_production" {
  memory = "1024"  # Aumentar de 512
}
```

---

## Task falha em Health Check

**Sintoma**: Task fica em `RUNNING` mas nunca fica `HEALTHY`, ALB marca como `unhealthy`

### Task Health Check vs ALB Health Check

```
Task Health Check (ECS):
├─ Definido em container_definitions.healthCheck
├─ Executado DENTRO do container
├─ Se falhar 2 retries → task é stopped
└─ Usado para detecção rápida de falhas

ALB Health Check:
├─ Definido em target_group.health_check
├─ Enviado PELA ALB para a task
├─ Se responder com 200 → healthy
└─ Usado para remoção de traffic
```

### Debugging

```bash
# 1. Verificar se endpoint /health existe
curl http://<task-ip>:8080/health

# 2. Verificar status atual
aws ecs describe-tasks \
  --cluster producao-cluster \
  --tasks <task-arn> \
  --query 'tasks[0].{Health:healthStatus, Status:lastStatus}'

# 3. Ver eventos
aws ecs describe-tasks \
  --cluster producao-cluster \
  --tasks <task-arn> \
  --query 'tasks[0].healthStatus'

# 4. Logs de health check
aws logs tail /ecs/minha-aplicacao/health-checks --follow
```

### Cenários Comuns

#### Aplicação demora para iniciar

```hcl
# ✅ FIX: Aumentar startPeriod
healthCheck = {
  startPeriod = 120  # 120 segundos de grace period
  interval    = 30
  timeout     = 5
  retries     = 2
}
```

#### Endpoint /health não implementado

```python
# Python/Flask
from flask import Flask, jsonify

app = Flask(__name__)

@app.route('/health', methods=['GET'])
def health():
    return jsonify({"status": "healthy"}), 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

#### Health check timeout

```bash
# Health check matando tasks rápido?
# Aumentar timeout

healthCheck = {
  timeout = 10  # 5 segundos pode ser pouco
  interval = 30
  retries = 3   # Mais tolerância
}
```

---

## Custos Inesperados

### Scenario 1: Tasks 24/7 em Fargate

**Custo observado**: $200/mês quando esperava $50/mês

```bash
# Calcular custo real
# 2 tasks × 256 CPU × 512 MB × 730 horas/mês × $0.04536/hour
# = 2 × 0.25 × 0.5 × 730 × 0.04536 = ~$17/mês

# Se está caro, verificar:
aws ecs describe-services \
  --cluster producao-cluster \
  --services minha-aplicacao-service \
  --query 'services[0].desiredCount'
```

**Fix**: Use Fargate Spot para non-critical
```hcl
resource "aws_ecs_cluster_capacity_providers" "main" {
  capacity_providers = ["FARGATE", "FARGATE_SPOT"]
  
  default_capacity_provider_strategy {
    base              = 1
    capacity_provider = "FARGATE"
  }
  
  default_capacity_provider_strategy {
    weight            = 100  # Overflow para SPOT
    capacity_provider = "FARGATE_SPOT"
  }
}
```

### Scenario 2: Auto-scaling Flapping

**Sintoma**: Tasks constantemente escalando up e down, CPU sempre variável

```bash
# Monitorar atividade de scaling
aws application-autoscaling describe-scaling-activities \
  --service-namespace ecs \
  --query 'ScalingActivities[0:10].{Status:StatusMessage, Time:Timestamp}'
```

**Causa**: Cooldown muito curto

**Fix**:
```hcl
resource "aws_appautoscaling_policy" "cpu" {
  target_tracking_scaling_policy_configuration {
    scale_in_cooldown  = 300   # 5 minutos (não 0!)
    scale_out_cooldown = 60    # 1 minuto
  }
}
```

**Economia**: Evita 2-3x custo em scaling desnecessário.

### Scenario 3: CloudWatch Logs Infinito

**Sintoma**: CloudWatch bill > ECS bill

```bash
# Verificar tamanho de logs
aws logs describe-log-groups \
  --query 'logGroups[].{Name:logGroupName, Size:storageSizeInBytes}' \
  --output table
```

**Causa**: Logs sem retention policy

**Fix**:
```hcl
resource "aws_cloudwatch_log_group" "ecs_app" {
  name              = "/ecs/minha-aplicacao"
  retention_in_days = 30  # ~$0.03 per day para 1GB
}

# Breakdown:
# 100 MB/dia × 30 dias = 3 GB
# 3 GB × $0.50/GB-month = $1.50/month
```

---

## Performance Issues

### Tasks lentas para responder

```bash
# 1. Verificar latência P99
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name TargetResponseTime \
  --statistics Average,Maximum \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60

# 2. Verificar Container Insights
aws cloudwatch get-metric-statistics \
  --namespace ECS/ContainerInsights \
  --metric-name CPUUtilized \
  --dimensions Name=ServiceName,Value=minha-aplicacao-service
```

**Causas Possíveis**:
- CPU saturada (70%+)
- Memory pressure
- Banco de dados lento
- Rede congestionada

**Fix**: Auto-scale mais agressivamente
```hcl
resource "aws_appautoscaling_policy" "cpu" {
  target_value       = 50.0  # Mais agressivo que 70%
  scale_out_cooldown = 30    # Escala para fora mais rápido
}
```

---

## Deployment Issues

### Deploy travado ("Waiting for service stability")

```bash
# Ver progressão do deployment
aws ecs describe-services \
  --cluster producao-cluster \
  --services minha-aplicacao-service \
  --query 'services[0].deployments'
```

**Causas**:
- Health checks falhando
- Insuficiente capacity
- Timeout no ALB

**Fix**: Aumentar health check timeout
```hcl
resource "aws_ecs_service" "app" {
  health_check_grace_period_seconds = 120  # de 60
}
```

### Rollback automático não funciona

```bash
# Verificar se circuit breaker está habilitado
aws ecs describe-services \
  --cluster producao-cluster \
  --services minha-aplicacao-service \
  --query 'services[0].deploymentConfiguration'
```

**Fix**:
```hcl
deployment_configuration {
  deployment_circuit_breaker {
    enable   = true   # ✅ DEVE estar true
    rollback = true   # ✅ MUST estar true
  }
}
```

---

## Quick Reference

| Problema | Comando Debug | Fix Rápido |
|----------|---------------|-----------|
| Task não inicia | `aws logs tail /ecs/...` | Checar image URI |
| Health check falha | `aws ecs describe-tasks` | Aumentar startPeriod |
| Caro demais | `aws logs describe-log-groups` | Setar retention_in_days |
| Lento | `aws cloudwatch get-metric-statistics` | Aumentar min_capacity |
| Deploy travado | `aws ecs describe-services` | Aumentar health_check_grace_period_seconds |
