# 🐳 ECS (Elastic Container Service)

Este guia foi dividido em 9 módulos especializados para facilitar aprendizado focado e referência rápida.

### 1. [📖 Introdução ao ECS](01-introducao-ecs.md)
- O que é ECS e por que usar
- Quando ECS é apropriado vs inadequado
- Comparação com Lambda, EKS, EC2
- Trade-offs de decisão

**Leia se**: Começando com containers na AWS ou avaliando qual serviço usar.

---

### 2. [🏗️ Arquitetura e Componentes](02-arquitetura-componentes.md)
- Os 3 componentes principais: Task Definition, Cluster, Service
- Exemplos completos de código Terraform
- Task Definition: best practices, anti-patterns
- Cluster Fargate vs EC2
- Service: deployment strategies, scaling
- Scenarios de deployment (novo, falha, scaling)

**Leia se**: Quer entender como ECS funciona internamente e implementar arquitetura sólida.

---

### 3. [🔍 Comparativo: Cluster vs Service vs Task](09-comparativo-cluster-service-task.md)
- Distinção clara entre Cluster, Service e Task Definition
- Tabela comparativa de responsabilidades
- Quando atualizar cada componente
- Relacionamento e fluxo entre componentes
- Exemplos Terraform completos
- Erros comuns e anti-patterns
- Analogias com Kubernetes e mundo real

**Leia se**: Quer entender nitidamente qual é o papel de cada componente ECS e evitar confusões.

---

### 4. [⚖️ Comparação com Alternativas](03-comparacao-alternativas.md)
- Matriz de decisão: Lambda vs ECS Fargate vs ECS EC2 vs EKS
- Análise detalhada de cada opção
- Matriz de trade-offs
- Fluxograma de decisão
- Custo real de cada alternativa

**Leia se**: Precisa justificar por que escolheu ECS vs outra solução.

---

### 5. [🔐 Segurança em Produção](04-seguranca-producao.md)
- IAM Roles: Execution Role vs Task Role
- Least privilege (OBRIGATÓRIO)
- Secrets Management com Secrets Manager
- Network isolation com VPC privado
- Security Groups restritivos
- KMS encryption
- Checklist de segurança completo

**Leia se**: Colocando ECS em produção ou auditando segurança.

---

### 6. [📈 Auto Scaling](05-auto-scaling.md)
- Target Tracking Scaling (predefinido)
- Scaling baseado em CPU, memória, requisições
- Métricas customizadas
- Armadilhas comuns (flapping, conflicts)
- Monitoramento de auto scaling
- Cooldown appropriado

**Leia se**: Precisa fazer ECS escalar automaticamente com a demanda.

---

### 7. [🚀 CI/CD com GitHub Actions](06-cicd-github-actions.md)
- Pipeline GitHub Actions completo
- OIDC Federation (zero credentials)
- Build, push ECR, deploy ECS
- IAM Role para GitHub Actions
- Notifications (Slack/Teams)
- Best practices de pipeline

**Leia se**: Configurando deployments automáticos via GitHub.

---

### 8. [🏛️ Well-Architected Framework](07-well-architected.md)
- Alinhamento com 5 pilares AWS:
  - Operational Excellence (observabilidade, deployment)
  - Security (IAM, data protection, network)
  - Reliability (multi-AZ, health checks, auto-scaling)
  - Performance Efficiency (right-sizing, caching)
  - Cost Optimization (Fargate Spot, CloudWatch Logs)
- Checklist completo de implementação

**Leia se**: Quer arquitetura robusta e production-ready.

---

### 9. [🔧 Troubleshooting](08-troubleshooting.md)
- Task não inicia: diagnóstico e soluções
- Health check failures
- Custos inesperados
- Performance issues
- Deployment problems
- Quick reference table

**Leia se**: Enfrentando problemas em staging/produção.

---

### Custos Estimados (por mês)

| Serviço | Task Size | Tasks | Custo |
|---------|-----------|-------|-------|
| Fargate | 256 CPU, 512 MB | 2 | ~$67 |
| Fargate + Spot | 50% on + 50% spot | 2 | ~$43 |
| ECS EC2 | t3.medium | 2 | ~$30 |
| EKS | Fargate | 2 | ~$150+ |
| Lambda | - | unlimited | ~$0 (1M req gratis) |

---

## 🔍 Conceitos-Chave

### Task Definition
**O blueprint.** Define como um container será executado (imagem, porta, logs, IAM, health checks). Imutável.

### Cluster
**O agrupador.** Abstração que agrupa capacidade (Fargate/EC2). Gerenciado por AWS.

### Service
**O orquestrador.** Mantém N tasks rodando continuamente, gerencia deployment, scaling.

### Fargate
**Serverless para containers.** Você define resources (CPU/memory), AWS gerencia infraestrutura. Pague por uso.

### Container Insights
**Observabilidade integrada.** Métricas de CPU, memory, network em tempo real. CloudWatch.

---

## ⚠️ Armadilhas Comuns

1. **Usar `latest` em produção**
   - Problema: Não é idempotente, deployment imprevisível
   - Solução: Sempre usar versão específica (v1.2.3)

2. **Sem health checks**
   - Problema: Containers zombis não detectados
   - Solução: Adicionar healthCheck em task definition

3. **Hardcode secrets em variables**
   - Problema: Vazamento em logs, Git history
   - Solução: Usar Secrets Manager

4. **Single AZ**
   - Problema: Falha de AZ = downtime total
   - Solução: Sempre multi-AZ (min 2)

5. **Auto-scaling flapping**
   - Problema: Scaling constantemente, custo 2-3x
   - Solução: scale_in_cooldown = 300s, scale_out_cooldown = 60s

6. **Sem logs retention**
   - Problema: CloudWatch bill explode
   - Solução: retention_in_days = 30

---

## 📚 Recursos Externos

- [AWS ECS Documentation](https://docs.aws.amazon.com/ecs/)
- [ECS Best Practices Guide](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/best_practices.html)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [ECS Workshop Hands-On](https://www.ecsworkshop.com/)
- [Container Security Best Practices](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/security.html)

---

## 🤔 FAQ

**P: Devo usar Fargate ou EC2?**  
A: Comece com Fargate. Migre para EC2 apenas se benchmarks mostram necessidade (GPU, constant 24/7, throughput muito alto).

**P: Quantas AZs preciso?**  
A: Mínimo 2 para HA. 3 para aplicações críticas.

**P: Qual tamanho de container?**  
A: Comece com 256 CPU + 512 MB. Ajuste observando Container Insights.

**P: Como debug uma task?**  
A: Use `aws ecs execute-command` (ECS Exec) ou veja logs em CloudWatch.

**P: Quanto custa?**  
A: Fargate 256/512: ~$0.04536/hour. 2 tasks × 730h = ~$67/mês.

---