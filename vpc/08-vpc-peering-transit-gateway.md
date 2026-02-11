# 08 - VPC Peering e Transit Gateway

## VPC Peering (1:1 Connection)

### O que é?

VPC Peering é uma conexão **point-to-point** entre 2 VPCs.

```
┌──────────────┐
│  VPC-A       │
│ 10.0.0.0/16  │
└──────┬───────┘
       │ Peering
       │ (pcx-xxxxx)
       │
┌──────┴───────┐
│  VPC-B       │
│ 10.1.0.0/16  │
└──────────────┘

Características:
- Direto (sem intermediário)
- Low latency (AWS internal)
- Cross-region permitido
- Same account ou cross-account
```

### Setup Peering

#### Step 1: Criar Peering Connection

```
AWS Console → VPC → Peering Connections → Create

┌────────────────────────────────────┐
│ Create VPC Peering Connection      │
├────────────────────────────────────┤
│ My VPC ID: vpc-a (10.0.0.0/16)    │
│ VPC ID to peer with: vpc-b         │
│ (10.1.0.0/16)                      │
│ Peer region: us-east-1             │
│ (pode ser outra region)            │
└────────────────────────────────────┘

Resultado: pcx-xxxxx (pending acceptance)
State: pending-acceptance
```

#### Step 2: Aceitar Peering (no VPC-B)

```
Se cross-account, owner VPC-B precisa aceitar

AWS Console → VPC → Peering Connections → pcx-xxxxx

Actions → Accept request

State: active ✓
```

#### Step 3: Adicionar Rotas

```
VPC-A Route Table:
────────────────────────────────
Destination    Target
10.0.0.0/16    local
10.1.0.0/16    pcx-xxxxx (manual)

VPC-B Route Table:
────────────────────────────────
Destination    Target
10.1.0.0/16    local
10.0.0.0/16    pcx-xxxxx (manual)

Nota: Roteamento manual (não automático como VPN com BGP)
Você precisa adicionar routes em ambas VPCs
```

### Limitações de Peering

```
❌ NÃO é transitivo:

VPC-A ←→ VPC-B ←→ VPC-C

VPC-A não consegue falar com VPC-C!
Precisa:
- VPC-A ↔️ VPC-C peering direto

↳ Com N VPCs: N*(N-1)/2 peerings
↳ 100 VPCs = 4.950 peerings! (inviável)
```

### Segurança em Peering

```
✅ CIDR não pode ser igual
   VPC-A: 10.0.0.0/16
   VPC-B: 10.1.0.0/16 ✓

❌ CIDR sobreposto:
   VPC-A: 10.0.0.0/16
   VPC-B: 10.0.1.0/18 ✗ (overlaps 10.0.1.0/24)
   
✅ Security Groups funcionam:
   EC2 em VPC-A com SG-A
   EC2 em VPC-B com SG-B
   
   SG-B can say:
   Inbound: TCP 3306 from sg-a (cross-VPC reference)

❌ Inbound SG rules cross-VPC
   SG-A não pode citar sg-b se VPCs différentes
   (precisa CIDR de VPC-B subnet)
```

---

## Transit Gateway (Hub-and-Spoke)

### Problema: Mesh Peering

```
10 VPCs = 45 peering connections
100 VPCs = 4.950 peering connections!

Muito complexo para operação:
- Cada novo VPC precisa peer com todos
- Roteamento manual em cada VPC
- Monitoramento de 45 + connections
```

### Solução: Transit Gateway

```
         VPC-A
          ↓
VPC-B ← [TGW] ← VPC-C
          ↓
         VPC-D

Beneficios:
- Sem VPC management ↔️ cada outra
- Apenas attach ao TGW
- Roteamento centralizado
- Escalável (<1ms latência)
```

### TGW Arquitetura

```
┌─────────────────────────────────┐
│      Transit Gateway            │
│  (AWS managed service)          │
│                                 │
│  ┌─ Route Table TGW-RT-PROD    │
│  │  Destination   Target       │
│  │  10.0.0.0/16   vpc-a        │
│  │  10.1.0.0/16   vpc-b        │
│  │  10.2.0.0/16   vpc-c        │
│  │  192.168.0.0/16 vpn-conn    │
│  │  (rotas internas do TGW)    │
│  └──────────────────────────    │
│                                 │
│  ┌─ Route Table TGW-RT-DEV     │
│  │  10.20.0.0/16  vpc-dev      │
│  │  (isolamento de VPCs)       │
│  └──────────────────────────    │
│                                 │
└─────────────────────────────────┘
    ↓         ↓         ↓
  [VPC-A]   [VPC-B]   [VPC-C]
  Attached  Attached  Attached
```

### Setup Transit Gateway

#### Step 1: Criar TGW

```
AWS Console → VPC → Transit Gateways → Create

┌──────────────────────────────────┐
│ Create Transit Gateway           │
├──────────────────────────────────┤
│ Name: prod-transit-gw            │
│ Amazon ASN: 64512 (default)      │
│ Default route table: Yes         │
│ Enable DNS support: Yes          │
│ Enable multicast: No             │
│ Description: Hub for prod VPCs   │
└──────────────────────────────────┘

Resultado: tgw-xxxxx (available)
```

#### Step 2: Criar TGW Route Table

```
AWS Console → TGW → Transit Gateway Route Tables

┌────────────────────────────────┐
│ Route Table: tgw-rtb-prod      │
├────────────────────────────────┤
│ Name: tgw-rtb-prod             │
│ Enable propagation: Yes        │
│ VPCs attached: (manual)        │
└────────────────────────────────┘

Propagation:
- VPC attachments automatically propagate routes
- Ex: VPC-A (10.0.0.0/16) announces itself
- TGW learns 10.0.0.0/16 → vpc-attach-a
```

#### Step 3: Attach VPCs ao TGW

```
AWS Console → TGW → Transit Gateway Attachments → Create

┌────────────────────────────────┐
│ TGW Attachment: VPC-A          │
├────────────────────────────────┤
│ Transit Gateway: tgw-xxxxx     │
│ VPC: vpc-a (10.0.0.0/16)       │
│ Subnets: private-1a, private-2a│
│ Route table association:       │
│   tgw-rtb-prod                 │
│ (para propagar rotas)          │
└────────────────────────────────┘

Repeat para VPC-B, VPC-C, etc.
```

#### Step 4: Atualizar VPC Route Tables

```
VPC-A Route Table:
──────────────────────────────
Destination    Target
10.0.0.0/16    local
0.0.0.0/0      nat-xxxxx
10.999.0.0/8   tgw-xxxxx
               (covers 10.1.0.0/16, 10.2.0.0/16, etc)

VPC-B Route Table:
──────────────────────────────
Destination    Target
10.1.0.0/16    local
0.0.0.0/0      nat-xxxxx
10.999.0.0/8   tgw-xxxxx

Agora:
VPC-A EC2 (10.0.1.5) quer falar com VPC-B EC2 (10.1.1.5)
├─ Route: 10.1.1.5 matches 10.999.0.0/8? Não, matches 0.0.0.0/0
└─ Próximo: Espera, temos 10.1.0.0/16 no TGW!

Melhor:
Destination    Target
10.0.0.0/16    local
10.1.0.0/16    tgw-xxxxx (específico)
10.2.0.0/16    tgw-xxxxx (específico)
```

### TGW Route Tables para Isolamento

```
Cenário: VPCs de projetos diferentes não devem falar

SolÇão: Múltiplas route tables on TGW

TGW Route Table PROD:
┌────────────────────────┐
│ Attachments:          │
│ - vpc-a               │
│ - vpc-b               │
│ - vpn-connection (DC) │
│                        │
│ Routes:               │
│ 10.0.0.0/16 → vpc-a  │
│ 10.1.0.0/16 → vpc-b  │
│ 192.168.0.0/16 → vpn │
└────────────────────────┘

TGW Route Table DEV:
┌────────────────────────┐
│ Attachments:          │
│ - vpc-dev-1           │
│ - vpc-dev-2           │
│ - ec2-instance        │
│                        │
│ Routes:               │
│ 10.20.0.0/16 → dev-1 │
│ 10.21.0.0/16 → dev-2 │
└────────────────────────┘

Resultado:
VPC-A (PROD) não consegue falar com VPC-DEV-1
(porque não está na mesma route table)
```

### Segurança de TGW

```
✅ Isolamento por Route Table
   └─ Prod VPCs em tgw-rtb-prod
   └─ Dev VPCs em tgw-rtb-dev
   └─ Prod não consegue acessar Dev

✅ Security Groups ainda vale
   └─ TGW apenas roteia
   └─ SG filters final entry

❌ Não confie APENAS no TGW
   └─ Ataques internos podem usar TGW
   └─ Monitorar tráfego via VPC Flow Logs

✅ Enable TGW Flow Logs
   └─ Monitor tráfego passando no TGW
   └─ Detect anomalias
```

---

## Peering vs Transit Gateway

| Aspecto | Peering | TGW |
|--------|---------|-----|
| Setup | Manual | Managed |
| Scale (max VPCs) | ~20 | 1000+ |
| Roteamento | Manual | Propagation + centralized |
| Latência | < 1ms | < 1ms |
| Multi-region | ✓ | ✓ |
| Isolamento | Basic (CIDR) | Advanced (Route Tables) |
| Custo | $0 | $0.05/hour + data |

**Regra**: 
- < 5 VPCs: Peering ok
- 5+ VPCs: Use Transit Gateway

---

## TGW + VPN + Multi-Region

### Arquitetura Completa

```
Data Center (on-prem)
     ↓ VPN
┌─────────────────────────────────────┐
│     AWS Region 1 (us-east-1)        │
│  ┌──────────────────────────────┐   │
│  │ Transit Gateway (tgw-us-east)│   │
│  │ ├─ vpc-prod-1 (prod)         │   │
│  │ ├─ vpc-prod-2 (prod)         │   │
│  │ ├─ vpn-connection (DC)       │   │
│  │ └─ tgw-peering (to us-west) │   │
│  └──────────────────────────────┘   │
│         (VPC RT: 0.0.0.0/0 → tgw)   │
└─────────────────────────────────────┘
           ↓ TGW Peering (inter-region)
┌─────────────────────────────────────┐
│     AWS Region 2 (us-west-2)        │
│  ┌──────────────────────────────┐   │
│  │ Transit Gateway (tgw-us-west)│   │
│  │ ├─ vpc-prod-3 (us-west)      │   │
│  │ ├─ vpc-dr-1 (disaster recover)
│  │ └─ tgw-peering (to us-east)  │   │
│  └──────────────────────────────┘   │
│         (VPC RT: 0.0.0.0/0 → tgw)   │
└─────────────────────────────────────┘

Resultado:
- Prod-1 em us-east fala com Prod-3 us-west
- Prod-1 fala com DC via VPN
- us-west herda acesso a DC automaticamente (via tgw peering)
- Multi-region ativo-ativo!
```

---

## Troubleshooting TGW

### Attachment não aparece em route table

```
Checklist:
1. Attachment criado e active?
   AWS Console → TGW Attachments
   State: available

2. Route table association?
   Attachment → edit route table association
   Escolher: tgw-rtb-prod

3. Propagation habilitado?
   TGW route table → Edit propagation
   Attachment deve estar listado

4. Inspection da rota:
   TGW route table → Routes → search 10.0.0.0
   Deve aparecer com target: attachment-id
```

### Connectivity falha

```
VPC-A EC2 (10.0.1.5) não consegue ping VPC-B EC2 (10.1.1.5)

Checklist:
1. TGW route table tem ambas VPCs?
   10.0.0.0/16 → vpc-attach-a ✓
   10.1.0.0/16 → vpc-attach-b ✓

2. VPC route tables?
   VPC-A: 10.1.0.0/16 → tgw-xxxxx ✓
   VPC-B: 10.0.0.0/16 → tgw-xxxxx ✓

3. Subnets asociadas ao TGW attachment?
   VPC-A att: subnets [private-1a, private-1b] ✓
   VPC-B att: subnets [private-2a, private-2b] ✓

4. Security Groups?
   EC2-A SG outbound: all to 0.0.0.0/0 ✓
   EC2-B SG inbound: ping (ICMP) from 10.0.0.0/16 ✓

5. NACLs permitindo tráfego?
   Inbound: ICMP from 10.0.0.0/16
   Outbound: ICMP to 10.1.0.0/16
```

---

## Best Practices

### ✅ SIM
- Usar TGW para 5+ VPCs
- Múltiplas route tables para isolamento
- Habilitar TGW flow logs para security
- TGW no center, VPCs nas edges
- One attachment per VPC (per az melhor)

### ❌ NÃO
- Mesh peering em escala
- TGW sem isolamento (todos em 1 route table)
- Esquecer que TGW não é segurança (usar SGs)
- Default route (0.0.0.0/0) no TGW (pode enviar wrongly)
- Route loops (circular references)

---

## Próximos Passos

- **[09 - Multi-Região](09-multi-region-hybrid.md)**: TGW peering inter-region
- **[05 - Roteamento Avançado](05-roteamento-avancado.md)**: BGP propagation
- **[12 - IaC com Terraform](12-iac-terraform.md)**: Codificar TGW

---

**Anterior**: [07 - VPN e Site-to-Site](07-vpn-connectivity.md)  
**Próximo**: [09 - Arquiteturas Multi-Região e Híbridas](09-multi-region-hybrid.md)
