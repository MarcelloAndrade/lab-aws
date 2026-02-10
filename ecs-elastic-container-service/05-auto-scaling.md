# 📈 Auto Scaling em ECS

Auto scaling em ECS não é automático: PRECISA de configuração explícita.

## Target Tracking Scaling

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

## Métricas Customizadas

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

## Armadilhas Comuns

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

## Métricas Predefinidas vs Customizadas

### Predefinidas (Recomendado)

| Métrica | Espaço de Nomes | Dimensão | Unidade |
|---------|-----------------|----------|---------|
| ECSServiceAverageCPUUtilization | AWS/ECS | ServiceName | Porcentagem |
| ECSServiceAverageMemoryUtilization | AWS/ECS | ServiceName | Porcentagem |
| ALBRequestCountPerTarget | AWS/ApplicationELB | TargetGroup | Contagem |

### Customizadas

**Caso de uso**: Métrica específica da aplicação
- Número de mensagens em fila SQS
- Latência P99
- Taxa de erro
- Tamanho da fila do banco

**Exemplo: Escalar baseado em filas SQS**

```hcl
resource "aws_appautoscaling_policy" "sqs_queue_depth" {
  name               = "sqs-queue-depth-scaling"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.ecs_target.resource_id
  scalable_dimension = aws_appautoscaling_target.ecs_target.scalable_dimension
  service_namespace  = aws_appautoscaling_target.ecs_target.service_namespace

  target_tracking_scaling_policy_configuration {
    customized_metric_specification {
      metric_dimension {
        name  = "QueueName"
        value = aws_sqs_queue.app_queue.name
      }
      metric_name = "ApproximateNumberOfMessagesVisible"
      namespace   = "AWS/SQS"
      statistic   = "Average"
      unit        = "Count"
    }
    # Escalar para manter ~100 mensagens por task
    target_value = 100.0
  }
}
```

## Monitoramento de Auto Scaling

```hcl
# CloudWatch Alarm para detectar scaling que não está funcionando

resource "aws_cloudwatch_metric_alarm" "scaling_failure" {
  alarm_name          = "ecs-scaling-failure"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "GroupDesiredCapacity"
  namespace           = "AWS/ApplicationAutoScaling"
  period              = 300
  statistic           = "Average"
  threshold           = 20  # Max capacity
  alarm_description   = "Alert se scaling atingir limite"
  alarm_actions       = [aws_sns_topic.alerts.arn]

  dimensions = {
    ServiceNamespace  = "ecs"
    ResourceId        = "service/${aws_ecs_cluster.main.name}/${aws_ecs_service.app.name}"
    ScalableDimension = "ecs:service:DesiredCount"
  }
}
```
