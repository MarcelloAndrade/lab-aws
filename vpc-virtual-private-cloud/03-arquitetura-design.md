# 03 - Arquitetura e Design Patterns

## Princípios de Design de VPC

### 1. Design para Falha
- Sempre assume que qualquer componente pode falhar
- Múltiplas AZs mesmo que custe mais
- Sem single point of failure crítico

### 2. Least Privilege
- Bloqueia tudo por padrão, abre apenas o necessário
- Menor CIDR que funciona
- Menor Security Group que funciona

### 3. Simplicidade Operacional
- Menos subnets = menos para gerenciar
- Menos SGs = menos para entender
- Documentação é parte do design

### 4. Escalabilidade
- Pense do início: CIDR grande
- Estrutura permite adicionar tiers sem redesign
- Roteamento centralizado (Transit Gateway) para multi-VPC

---

## Padrão 1: 3-Tier Tradicional (Mais Comum)

```
┌─────────────────────────────────────────────────────────┐
│  VPC: 10.0.0.0/16                                      │
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │ AZ: us-east-1a                                  │  │
│  │                                                  │  │
│  │ Web Tier:                                       │  │
│  │ └─ Public Subnet: 10.0.1.0/24                  │  │
│  │    ├─ ALB (10.0.1.100)                         │  │
│  │    └─ Bastion Host (10.0.1.50)                 │  │
│  │                                                  │  │
│  │ App Tier:                                       │  │
│  │ └─ Private Subnet: 10.0.11.0/24                │  │
│  │    └─ Auto Scaling Group with EC2 instances   │  │
│  │                                                  │  │
│  │ Data Tier:                                      │  │
│  │ └─ Private Subnet: 10.0.21.0/24                │  │
│  │    └─ RDS Multi-AZ (primary here)              │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │ AZ: us-east-1b                                  │  │
│  │                                                  │  │
│  │ Web Tier:                                       │  │
│  │ └─ Public Subnet: 10.0.2.0/24                  │  │
│  │    └─ NAT Gateway (10.0.2.200 + EIP)           │  │
│  │                                                  │  │
│  │ App Tier:                                       │  │
│  │ └─ Private Subnet: 10.0.12.0/24                │  │
│  │    └─ Auto Scaling Group with EC2 instances   │  │
│  │                                                  │  │
│  │ Data Tier:                                      │  │
│  │ └─ Private Subnet: 10.0.22.0/24                │  │
│  │    └─ RDS Multi-AZ (secondary here)            │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │ AZ: us-east-1c                                  │  │
│  │                                                  │  │
│  │ Web Tier:                                       │  │
│  │ └─ Public Subnet: 10.0.3.0/24                  │  │
│  │    └─ NAT Gateway (10.0.3.200 + EIP)           │  │
│  │                                                  │  │
│  │ App Tier:                                       │  │
│  │ └─ Private Subnet: 10.0.13.0/24                │  │
│  │    └─ Auto Scaling Group with EC2 instances   │  │
│  │                                                  │
│  │ Data Tier:                                      │  │
│  │ └─ Private Subnet: 10.0.23.0/24                │  │
│  │    └─ (RDS não coloca aqui, usa DB subnet group)
│  └──────────────────────────────────────────────────┘  │
│                                                         │
│  ┌─────────────────────────────────────────────┐       │
│  │  [Internet Gateway]                         │       │
│  │        ↓                                    │       │
│  │  [Route Tables]           [Security Groups]│       │
│  │                                             │       │
│  │  Public-RT:               Web-SG:          │       │
│  │  0.0.0.0/0 → IGW          80,443 ← 0.0.0.0/0  │
│  │                          22 ← Office IP   │       │
│  │  Private-RT:              App-SG:         │       │
│  │  0.0.0.0/0 → NAT          8080 ← Web-SG  │       │
│  │                                            │       │
│  │  Data-RT:                 Data-SG:       │       │
│  │  0.0.0.0/0 → NAT          3306 ← App-SG │       │
│  └─────────────────────────────────────────────┘       │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Quando Usar
- ✅ APIs REST tradicionais
- ✅ Aplicações web com banco de dados
- ✅ Padrão mais documentado na comunidade
- ✅ Fácil de entender e troubleshoot

### Subnets CIDR Breakdown

```
VPC: 10.0.0.0/16 (65,536 endereços)

Web Tier (Public):
  10.0.1.0/24   - AZ a
  10.0.2.0/24   - AZ b
  10.0.3.0/24   - AZ c
  = 768 endereços

App Tier (Private):
  10.0.11.0/24  - AZ a
  10.0.12.0/24  - AZ b
  10.0.13.0/24  - AZ c
  = 768 endereços

Data Tier (Private):
  10.0.21.0/24  - AZ a
  10.0.22.0/24  - AZ b
  10.0.23.0/24  - AZ c
  = 768 endereços

Sobra: ~61,728 endereços para futuro crescimento
```

### Route Tables

```
Public Route Table (public-rt):
────────────────────────────────────
Destination          Target
10.0.0.0/16          local
0.0.0.0/0            igw-xxxxx

Associado com: Public subnets (10.0.1/24, 10.0.2/24, 10.0.3/24)


Private Route Table (private-app-rt):
──────────────────────────────────────
Destination          Target
10.0.0.0/16          local
0.0.0.0/0            nat-xxxxx (NAT em zona a)

Associado com: App subnets (10.0.11/24, 10.0.12/24)


Private Route Table (private-data-rt):
──────────────────────────────────────
Destination          Target
10.0.0.0/16          local
0.0.0.0/0            nat-yyyyy (NAT em zona b)

Associado com: Data subnets (10.0.21/24, 10.0.22/24)
```

**Nota**: Data tier também precisa de NAT para:
- Backup para S3
- Updates de sistema operacional
- Chamadas para AWS services

---

## Padrão 2: DMZ (Desmilitarized Zone) com Bastion

```
┌────────────────────────────────────────────┐
│  VPC: 10.0.0.0/16                         │
│                                            │
│  ┌──────────────────────────────────────┐ │
│  │ DMZ Subnet: 10.0.0.0/25             │ │
│  │                                      │ │
│  │ [Bastion Host]─────┐               │ │
│  │     (Public)       │               │ │
│  │                    │               │ │
│  │                    ↓               │ │
│  │   [IGW] ←──────────┘               │ │
│  └──────────────────────────────────────┘ │
│                    ↓                       │
│  ┌──────────────────────────────────────┐ │
│  │ Internal Subnet: 10.0.1.0/25         │ │
│  │                                      │ │
│  │ [EC2 App]←──────[Bastion]           │ │
│  │    (Private, port 22 only)          │ │
│  └──────────────────────────────────────┘ │
│                                            │
│ Security:                                  │
│ - SSH only from Bastion to Internal      │
│ - Developers never SSH directly          │
│ - All SSH keypairs can be on Bastion     │
│ - Easier to audit/control                │
│                                            │
└────────────────────────────────────────────┘
```

### Quando Usar
- ✅ Hardened security (ex: financial/healthcare)
- ✅ Compliance (SOC 2, HIPAA)
- ✅ Controlled access requirements
- ❌ Não use se tiver Systems Manager Sessions Manager (melhor alternativa)

### Security Groups

```
Bastion SG:
- Inbound: TCP 22 from office.com (single IP)
- Outbound: TCP 22 to Internal SG

Internal SG:
- Inbound: TCP 22 from Bastion SG (by SG reference)
- Outbound: All (ou restritivo conforme necessário)
```

---

## Padrão 3: Microserviços com Service Discovery

```
┌─────────────────────────────────────────────┐
│  VPC: 10.0.0.0/16                          │
│                                             │
│  ┌────────────────────────────────────────┐ │
│  │ Load Balancer Subnet: 10.0.1.0/24     │ │
│  │ (Public, span all AZs)                │ │
│  │                                        │ │
│  │    [ALB/NLB]  ← External Traffic     │ │
│  └────────────────────────────────────────┘ │
│           ↓                                  │
│  ┌────────────────────────────────────────┐ │
│  │ Container/Service Subnets              │ │
│  │ (Private, service-discovery enabled)   │ │
│  │                                        │ │
│  │ [User Service]  [Order Service]      │ │
│  │ [Auth Service]  [Payment Service]    │ │
│  │ [API Gateway]   [Cache Cluster]      │ │
│  │                                        │ │
│  │ Service Discovery:                    │ │
│  │ - Services find each other via DNS  │ │
│  │ - service.internal domain          │ │
│  │ - No hardcoded IP addresses        │ │
│  └────────────────────────────────────────┘ │
│                                             │
│  Route Table:                               │
│  0.0.0.0/0 → NAT (for docker pulls, etc) │ │
│                                             │
└─────────────────────────────────────────────┘
```

### Quando Usar
- ✅ ECS Fargate com service discovery
- ✅ EKS (Kubernetes)
- ✅ Container-heavy workloads
- ✅ API gateways/Microservices

### Key Differences
```
Tradicional (3-tier):
- IP direto = fácil de debugar
- Pouco dinâmico = menos mudanças
- Escala vertical (bigger boxes)

Microserviços:
- DNS-based service discovery
- Altamente dinâmico = containers aparecem/desaparecem
- Escala horizontal (muitas pequenas instâncias)
```

---

## Padrão 4: VPC Endpoints para Segurança Máxima

```
┌─────────────────────────────────────────────┐
│  VPC: 10.0.0.0/16                          │
│                                             │
│  [EC2 in Private Subnet]                   │
│         ↓                                   │
│  Quer acessar S3/DynamoDB/SQS              │
│         ↓                                   │
│  Opção A (ruim):                           │
│    EC2 → NAT → IGW → Internet → AWS service
│    Caro (per GB), lento, exposto           │
│                                             │
│  Opção B (bom):                            │
│    EC2 → [VPC Endpoint] → AWS service      │
│    Dentro AWS network, mais rápido, barato │
│                                             │
│  ┌──────────────────────────────────────┐  │
│  │ VPC Endpoints (Gateway type)         │  │
│  │ - S3 Gateway Endpoint                │  │
│  │ - DynamoDB Gateway Endpoint          │  │
│  │                                       │  │
│  │ Route Table:                         │  │
│  │ s3.amazonaws.com → vpce-s3           │  │
│  │ dynamodb... → vpce-dynamodb          │  │
│  └──────────────────────────────────────┘  │
│                                             │
│  ┌──────────────────────────────────────┐  │
│  │ VPC Endpoints (Interface type)       │  │
│  │ - SQS, SNS, Secrets Manager, etc     │  │
│  │ - Via ENI + Security Group           │  │
│  │ - Acessado por DNS privado          │  │
│  └──────────────────────────────────────┘  │
│                                             │
└─────────────────────────────────────────────┘
```

### Benefícios
- ✅ Não sai de AWS network (segurança)
- ✅ Sem NAT Gateway charges
- ✅ Mais rápido (mais próximo)
- ✅ Com policy, você controla acesso fino

### Exemplo de Caso de Uso
```
Aplicação precisa:
1. Carregar dados de S3
2. Ler/escrever em DynamoDB
3. Publicar em SNS

Se usar NAT Gateway:
- Traffic: Subir NAT → IGW → Internet (lento, caro)
- Segurança: Dados saem da AWS network

Se usar VPC Endpoints:
- Traffic: Direto para S3/DynamoDB/SNS (rápido, barato)
- Segurança: Tudo na AWS network
- Controle: Policies por endpoint
```

---

## Decisão: Qual Padrão Escolher?

| Critério | 3-Tier | DMZ | Microserviços | VPC Endpoints |
|----------|--------|-----|-------------|---|
| Complexidade | Média | Alta | Alta | Baixa |
| Segurança | Boa | Ótima | Ótima | Ótima |
| Escalabilidade | Boa | Boa | Excelente | N/A |
| Custo NAT | Alto | Alto | Alto | Eliminado |
| Ideal para | APIs web | Compliance | Containers | Data-heavy apps |
| Tempo setup | 1 dia | 2 dias | 2-3 dias | 2-4 horas |

### Recomendação para Novos Projetos

```
Start with:
1. 3-Tier (web + app + data)
   └─ Adicione DMZ (bastion) se compliance exigir
   └─ Adicione VPC Endpoints para S3/DynamoDB

Quando escalar:
2. Se vira microserviços → Mude para padrão Microserviços
3. Se data-heavy → Maximize uso de VPC Endpoints
4. Se multi-VPC → Adicione Transit Gateway
```

---

## Boas Práticas de Design

### ✅ SIM
- Múltiplas AZs desde o início
- Documentar CIDR planning
- Route Tables separadas por tier
- Security Groups granulares
- NAT Gateway redundant (1 por AZ)

### ❌ NÃO
- CIDR muito pequeno (deve crescer)
- Uma AZ (falha = downtime)
- Tudo em 1 Security Group
- Abrir tudo como 0.0.0.0/0 sem necessidade
- Bastion Host sem Systems Manager Session Manager

---
