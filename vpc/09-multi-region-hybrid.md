# 09 - Arquiteturas Multi-Região e Híbridas

## Padrão: Active-Passive (Disaster Recovery)

```
Região Primária (us-east-1):         Região Secundária (us-west-2):
┌──────────────────────────────┐    ┌──────────────────────────────┐
│ VPC: 10.0.0.0/16             │    │ VPC: 10.0.0.0/16             │
│ ├─ EC2 instances (running)   │    │ ├─ EC2 instances (stopped)   │
│ ├─ RDS (primary)             │    │ ├─ RDS (standby, read-only)  │
│ ├─ S3 bucket                 │    │ ├─ S3 bucket (replica)       │
│ └─ Load Balancer: active     │    │ └─ Load Balancer: inactive   │
│                              │    │                              │
│ Route53: weight routing       │    │ Route53: weight routing      │
│ prod.myapp.com →             │    │ prod.myapp.com →             │
│  100% → us-east-1 (primary)  │    │  0% → us-west-2 (standby)   │
└──────────────────────────────┘    └──────────────────────────────┘
         ↓ RDS replication (async)
      Standby RDS
         ↑
      On failure:
      1. Route53 health check fails on us-east-1
      2. Weight changes to: 0% → us-east-1, 100% → us-west-2
      3. Applications reconnect to standby
      4. Failover complete (RTO: 1-5 min)
```

### Components

**Data Replication:**
- RDS: Masters-standby (async replication)
- S3: CRR (Cross-Region Replication)
- DynamoDB: Global Tables (continents)
- Aurora: Multi-region (read replicas in standby)

**Route53 Health Checks:**
```
Health Check on us-east-1:
├─ HTTP endpoint: https://prod.myapp.com/health
├─ Interval: 30 seconds
├─ Failure threshold: 3 failures
└─ Region: us-east-1

If health check fails:
└─ Route53 removes us-east-1 from DNS
└─ Next requests go to us-west-2

RTO (Recovery Time Objective): 1-5 minutes
RPO (Recovery Point Objective): 1-10 minutes (depending on replication lag)
```

---

## Padrão: Active-Active (High Availability)

```
Ingressar: CloudFront / Application Load Balancer
                ↓
    ┌───────────┴───────────┐
    ↓                       ↓
┌─────────────────┐    ┌─────────────────┐
│ us-east-1       │    │ us-west-2       │
│ VPC 10.0.0.0/16 │    │ VPC 10.0.0.0/16 │
│ ├─ EC2 running  │    │ ├─ EC2 running  │
│ ├─ RDS primary  │    │ ├─ RDS primary  │
│ └─ ALB: active  │    │ └─ ALB: active  │
└─────────────────┘    └─────────────────┘
        ↓ Writes            ↓ Writes
   VPC Peering + Transfer Gateway (replication)
        ↓                   ↓
    Global DynamoDB / Aurora Global Database
        (both can write, conflict resolution automatic)

Route53:
prod.myapp.com →
  50% to us-east-1
  50% to us-west-2
(automatic failover if health check fails)
```

### Benefits vs Drawbacks

**Benefícios:**
- ✅ Sem downtime (true redundancy)
- ✅ Melhor latência (users connect nearest)
- ✅ Escala (2x capacity)

**Drawbacks:**
- ❌ Mais caro (2x infrastructure)
- ❌ Data consistency complexity (sync replication lag)
- ❌ Application complexity (multi-region aware)

---

## Arquitetura Híbrida: Hub-and-Spoke

```
┌──────────────────────────────────────────────┐
│  Data Center (on-premises)                   │
│  192.168.0.0/16                              │
│  ├─ File servers                             │
│  ├─ Legacy application                       │
│  └─ Database (replication target)            │
└──────────────────────────────────────────────┘
              ↑ VPN + Direct Connect
              │
┌──────────────────────────────────────────────┐
│  AWS Region (Central Hub)                    │
│  VPC: 10.0.0.0/16                           │
│  ├─ Transit Gateway (central router)         │
│  ├─ VPN Connection (DC)                      │
│  ├─ Direct Connect Virtual Interface         │
│  └─ Shared Services (DNS, logging, etc)      │
└──────────────────────────────────────────────┘
        │                       │
        ├──┬───────────┬────────┤
        ↓  ↓           ↓        ↓
    ┌────────┐  ┌────────┐  ┌────────┐
    │ App DV │  │ App QA │  │ App PR │
    │ VPC    │  │ VPC    │  │ VPC    │
    │10.10.. │  │10.20.. │  │10.30.. │
    └────────┘  └────────┘  └────────┘
    │           │           │
    ├─ Spoke 1  ├─ Spoke 2  └─ Spoke 3
    │           │
    ↓           ↓
  TGW Attachment (Deve subnet)
           ↓ Propagated routes

Result:
- Developers in Dev VPC can access DC data via TGW
- Dev VPC cannot access Prod VPC directly
  (isolamento via route tables)
- DC access centralizado (único ponto de saída)
```

---

## DynamoDB Global Tables (Multi-Master)

```
┌─────────────────────────────┐
│ DynamoDB Global Table       │
│ ApplicationData             │
└─────────────────────────────┘
       ↓
   ┌───────────────┬───────────────┐
   ↓               ↓
us-east-1 Table   us-west-2 Table
└──────────────┘   └──────────────┘

Características:
- Ambas pode escrever (conflict-free replicated data type)
- Writes propagam em < 1 segundo
- Diferente que RDS (que é primary-standby)

Write path:
Application us-east-1 faz Put item
└─ DynamoDB us-east-1 accepts write
└─ Async replica para us-west-2
└─ Code never blocks on replication

Read path:
Application us-west-2 faz Get item
└─ Instant return (local read)
└─ If item was just written in us-east-1
   └─ Eventual consistency (~1 second)
   └─ Strong consistency só em same region

Perfect for: Mobile apps, gaming, IoT (distributed writes)
```

---

## Aurora Global Database (Managed Replication)

```
┌─────────────────────────────┐
│ Aurora Global Database      │
│ prod-db-cluster             │
└─────────────────────────────┘
          ↓
   ┌──────────────┬──────────────┐
   ↓              ↓
Primary Region   Secondary Region
(us-east-1)      (us-west-2)
RDS Primary      RDS Read Replica
(can write)      (read-only)

┌────────────────┐ ┌────────────────┐
│ 1. Insert row  │ │ 2. Log shipping│
│ 3. Commit      │ │ 3. Apply redo  │
│ 4. One RTT     │ │ 4. Available   │
└────────────────┘ └────────────────┘

Replication: ~100ms to < 1s
Standby for failover: automatic in secondary

If primary fails:
- Automatic promotion of secondary to primary
- Existing read replicas continue serving reads
- Writers redirect to new primary
- RTO: < 1 minute
```

---

## Route53 Multi-Region Routing

### Geolocation Routing

```
Route53:
prod.myapp.com A records:

us-east-1 ALB IP 203.0.113.1
└─ Geolocation: United States
└─ Priority: 1

eu-west-1 ALB IP 203.0.114.1
└─ Geolocation: Europe
└─ Priority: 1

ap-south-1 ALB IP 203.0.115.1
└─ Geolocation: Asia

Default: us-east-1 (if not matched)

Result:
- User in New York  → 203.0.113.1 (us-east)
- User in Dublin    → 203.0.114.1 (eu-west)
- User in Mumbai    → 203.0.115.1 (ap-south)
- User elsewhere    → 203.0.113.1 (default)
```

### Latency Routing

```
Route53:
prod.myapp.com A records:

us-east-1 ALB IP 203.0.113.1
└─ Latency: measure to user's location
└─ If latency < 50ms → use this

us-west-2 ALB IP 203.0.113.2
└─ Latency: measure to user's location
└─ If latency < 50ms → use this

Result:
- User in CA        → us-west-2 (nearest)
- User in NY        → us-east-1 (nearest)
- AWS measures real RTT to each endpoint
- Dynamic routing based on actual latency
```

---

## Failover Architecture Multi-Região

### Health Checks com Failover

```
Primary Region (us-east-1):
┌──────────────────────────┐
│ Application Running      │
│ Health check endpoint:   │
│ https://api.../health    │
└──────────────────────────┘
         ↓
Route53 Health Check:
├─ Every 30 seconds
├─ If 3 failures → mark as unhealthy
├─ Route53 DNS changes weight to 0%
└─ Queries redirect to secondary

Secondary Region (us-west-2):
┌──────────────────────────┐
│ Application Warm Standby │
│ (smaller capacity OK)    │
│ Health check endpoint:   │
│ https://api.../health ✓  │
└──────────────────────────┘

Timeline:
0:00 - Primary goes down (no response to health check)
0:30 - Failure 1 detected
1:00 - Failure 2 detected
1:30 - Failure 3 detected → Health check fails
1:35 - Route53 updates DNS (propagation 1-5 seconds)
1:40 - Queries route to secondary
1:50 - Users reconnect to secondary
RTO: ~2 minutes (most are connected to primary still)
```

---

## Custos Multi-Região

```
Single Region (us-east-1):
- EC2: $100/month
- RDS: $200/month
- NAT: $50/month
- Data transfer: $50/month
Total: $400/month

Active-Passive (Standby us-west-2):
- Primary: $400/month
- Standby (50% capacity): $200/month
- RDS replication: $0 (included)
- Data transfer (inter-region): $200/month
Total: $800/month

Active-Active (us-east-1 + us-west-2):
- Primary: $400/month
- Secondary: $400/month
- Data sync: $200/month
Total: $1000/month

Regra 80/20:
- 20% cost increase → 99.9% availability (active-passive)
- 100% cost increase → higher capacity + some resilience (active-active)
```

---

## Best Practices Multi-Região

### ✅ SIM
- Automated failover com Route53 health checks
- Replicate critical data (RDS, S3, DynamoDB)
- Test failover regularly (não é fire-and-forget)
- Monitor replication lag (ensure RPO met)
- Use Application Load Balancer with cross-region routing
- Terraform para IaC (múltiplas regions em código)

### ❌ NÃO
- Manual failover (human error)
- Assumir standby sempre está sincronizado (teste!)
- Ignorer cost implications (2x infrastructure é caro)
- Active-active sem conflict resolution (data corruption)
- Esquecer que internet connectivity pode falhar (use PrivateLink)

---

## Próximos Passos

- **[10 - Monitoramento](10-monitoramento-troubleshooting.md)**: Monitor replication lag
- **[11 - Well-Architected](11-well-architected.md)**: Reliability pillar
- **[12 - IaC com Terraform](12-iac-terraform.md)**: Codificar multi-região

---

**Anterior**: [08 - VPC Peering e Transit Gateway](08-vpc-peering-transit-gateway.md)  
**Próximo**: [10 - Monitoramento e Troubleshooting](10-monitoramento-troubleshooting.md)
