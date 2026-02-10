# ⚖️ Comparação: ECS vs Alternativas

## Matriz de Decisão

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

## Análise Detalhada

### AWS Lambda
✅ **Quando usar**:
- Processamento de eventos (S3 uploads, DynamoDB streams)
- Webhooks simples
- Batch jobs curtos (< 15 minutos)
- Spikes impredíveis de tráfego
- Prototipagem rápida

❌ **Quando evitar**:
- Workloads de longa duração (> 15 minutos)
- Aplicações stateful
- Consumo consistente de CPU
- Requisitos de rede específicos

**Custo Real**: 
- Incluso: 1 milhão requests/mês
- Extra: $0.20 por 1 milhão requests

### ECS Fargate
✅ **Quando usar**:
- APIs REST com demanda variável
- Microserviços < 50 serviços
- Batch processing com duração variada
- Demanda previsível mas com picos
- Containerização sem gerenciamento de infra

❌ **Quando evitar**:
- Workloads 24/7 constantes (use EC2)
- Requisitos de hardware específico (GPU, memória alta)
- Taxa de dados muito altas (throughput)

**Custo Real**:
- 2 tasks × 256 CPU × 512 MB × 730 horas × $0.04536/hour ≈ $67/mês

### ECS EC2
✅ **Quando usar**:
- Workloads 24/7 com baseline consistente
- Requisitos de GPU ou hardware especial
- Taxa de dados muito alta (throughput)
- Controle fino sobre instâncias

❌ **Quando evitar**:
- Variabilidade alta de carga
- Demanda impredível
- Começar um novo projeto

**Custo Real**:
- t3.medium × 2 = ~$30/mês

### Amazon EKS
✅ **Quando usar**:
- 100+ microserviços
- Workloads distribuídos complexas (Cassandra, Kafka)
- Multi-cloud ou on-premises
- Expertise Kubernetes existente

❌ **Quando evitar**:
- Projeto novo, não containerizado
- Menos de 10 serviços
- Operacional simplicity importa

**Custo Real**:
- Control plane: $73/mês + EC2/Fargate compute
- Total mínimo: ~$150/mês

## Matriz de Trade-offs

| Aspecto | Lambda | ECS Fargate | ECS EC2 | EKS |
|---------|--------|-------------|---------|-----|
| **Observabilidade** | CloudWatch simples | Container Insights | Container Insights | Prometheus/Grafana |
| **Deployment** | Functions Automático | Service + ALB | Service + ALB | Helm/kubectl |
| **Secrets** | Secrets Manager | Secrets Manager | Secrets Manager | Vault/Sealed Secrets |
| **Roadmap** | Evolui rápido | Estável | Estável | Complexo |
| **Community** | Documentação excelente | Bom | Bom | Enorme |
| **Cost Predictability** | Imprevisível | Previsível | Previsível | Previsível |

## Fluxograma de Decisão

```
┌─────────────────────────────┐
│ Qual é o triggernda workload?│
└──────────────┬──────────────┘
               │
     ┌─────────┴──────────┐
     │                    │
     ▼                    ▼
  Evento         Contínua / Batch
  (S3, SNS)      
     │                    │
  < 15min?              Duração?
     │                    │
    Sim ◄──────────────┐  │
     │               < 5min
   LAMBDA              │
     │           Sim ◄─┴──┐
     │            │       │
     │           LAMBDA   > 5min
     │                    │
     │                    ▼
     │              Throughput
     │              alto?
     │                    │
     No                  Sim
      │                   │
      ▼                   ▼
    ECS            ECS EC2
  Fargate           (GPU?)
```
