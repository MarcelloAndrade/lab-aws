# 05 - Roteamento Avançado em VPC

## Princípios de Roteamento

### 1. Longest Prefix Match (Mais Específico Ganha)

```
Route Table:
─────────────────────────────────
Destination    Next Hop
0.0.0.0/0      IGW
10.0.0.0/16    local
10.0.1.0/24    NAT-A
10.0.1.0/25    VPN-1

Pacote para: 10.0.1.5
┌─ Matches 0.0.0.0/0?       ✓ (mas /0)
├─ Matches 10.0.0.0/16?     ✓ (mas /16)
├─ Matches 10.0.1.0/24?     ✓ (e /24)
├─ Matches 10.0.1.0/25?     ✓ (e /25)
└─ VENCE: /25 (mais específico)
   → VPN-1
```

### 2. Ordem de Avaliação

```
Route Table é avaliada em ordem de CIDR prefix
NÃO em ordem numérica de rule
NÃO em ordem de criação

AWS automaticamente ordena por /32 até /0
```

### 3. Precedência: Rota Local sempre ganha

```
Mesmo que tente adicionar:
Destination: 10.0.0.0/16
Next Hop: IGW

A rota local de VPC sempre prevalecer:
Destination: 10.0.0.0/16
Next Hop: local

Você nunca consegue rotear tráfego intra-VPC para fora
(segurança)
```

---

## Roteamento Dinâmico vs Estático

### Estático (Manual)

```
Route Table: public-rt
─────────────────────────────┐
Destination    Next Hop      │  Você manage
0.0.0.0/0      igw-xxxxx    │  manualmente
172.16.0.0/12  vpn-xxxxx    │
────────────────────────────┤
Adequado para: Poucas rotas (< 20)
```

### Dinâmico (Automático)

```
BGP (Border Gateway Protocol):

Você:              Customer Gateway          AWS
  ├─ Configura     ├─ BGP Announcements     ├─ AWS Network
  │  BGP peer      │  (subnets)             │
  └─ AS number     └─ Recebe AWS routes     └─ AS number

VPC route table é atualizada automaticamente!

Quando você:
- Adiciona subnet: BGP anuncia automaticamente
- Muda CIDR: BGP atualiza
- Deleta network: BGP revoga

Benefício: Multi-region ativo-ativo funciona automaticamente
```

---

## Casos de Uso Avançado: Roteamento Condicional

### Use Case 1: Failover de NAT Gateway

```
Problema:
┌─ Zona A falha
└─ Instâncias em Zona A ficam sem saída (NAT-A down)

Solução com roteamento condicional:

Route Table: private-app-1a
──────────────────────────────
0.0.0.0/0    nat-zone-a  (preferred)

Route Table: private-app-1a (fallback)
──────────────────────────────
0.0.0.0/0    nat-zone-b  (when zone-a fails)

Implementação manual:
1. Monitorar saúde de NAT-A (via health check)
2. Se down, mude rota para NAT-B
3. Se volta, volta a NAT-A

Automático:
Usar Transit Gateway com prioridades
```

### Use Case 2: Traffic Steering para WAF

```
Antes:
Internet → IGW → Instance [Sem proteção!]

Depois:
Internet → IGW → WAF (Network layer) → Instance

Route antes tráfego apontava direto for instance IP
Agora aponta para Network Firewall endpoint

AWS Network Firewall:
├─ Stateful inspection
├─ Geo-blocking
├─ DDoS protection
└─ Integra com Route Table
```

### Use Case 3: VPC Segregação por Projeto

```
Corporate VPC (shared services):
├─ Logging
├─ DNS
├─ NAT Gateways

Project VPC (isolated):
├─ App servers
├─ Databases
└─ Transit Gateway attachment

Routing:
Project VPC → Transit Gateway → Corporate VPC resources
(pero não direto para Internet - deve ir via Corporate)
```

---

## BGP e On-Premises Integration

### BGP Basics

```
BGP = anúncio de verdade de rede

Seu data center:
AS: 65000
Anuncia: 192.168.0.0/16 (sua rede)

AWS:
AS: 64512
Anuncia: 10.0.0.0/16 (sua VPC)

Quando estabelece VPN:
Data center aprende: 10.0.0.0/16 está via AWS
AWS aprende: 192.168.0.0/16 está via DC

Benefício: Dinâmico! Não precisa atualizar routes manualmente
```

### VPN BGP Setup

```
Customer Gateway (seu lado):
├─ IP Public: 203.0.113.50
├─ BGP ASN: 65000
└─ BGP IP: 169.254.10.1

VPN Connection:
├─ Tunnel 1: 169.254.10.1 ↔ 169.254.10.2
├─ Tunnel 2: 169.254.11.1 ↔ 169.254.11.2
└─ Both active (ECMP - Equal Cost Multi-Path)

Virtual Private Gateway (AWS side):
├─ BGP ASN: 64512
└─ BGP IPs: 169.254.10.2, 169.254.11.2

Result:
- Não precisa adicionar route 192.168.0.0/16
- BGP anuncia automaticamente
- Se túnel falha, BGP revoga rota
```

---

## Multi-Tier Routing

### Problema Clássico: How Does Return Traffic Find Its Way Back?

```
Scenario:
┌─────────┐
│ Cliente │  IP: 203.0.113.1
└────┬────┘
     │ GET / HTTP/1.1
     ↓ (source port: 54321)
┌──────────────────┐
│ Internet Gateway │
└────┬─────────────┘
     │ (traduz IP)
     │ IP privado packet
     ↓
┌──────────────────────┐
│ ALB (10.0.1.100)     │
│ Port: 80             │
└────┬─────────────────┘
     │ Encaminha para app-tier
     ↓ (destino: 10.0.11.50:8080)
┌──────────────────────┐
│ App Server           │
│ 10.0.11.50:8080      │
└────┬─────────────────┘
     │ Response: 200 OK
     │ Source: 10.0.11.50:8080
     │ Dest: 203.0.113.1:54321
     ↓

⚠️ PROBLEMA: Resposta vai onde?

Se App Server faz:
Route table lookup para 203.0.113.1
└─ Matches 0.0.0.0/0
└─ Next hop: NAT-Gateway
└─ NAT traduz de volta

203.0.113.1 recebe resposta! ✓
```

### Solução Total: Symmetric Routing

```
Request path:
Client → IGW → ALB → App Server

Response path (deve ser simétrica):
App Server → NAT → IGW → Client

Config necessária:
App Server pode mandar resposta direto para App Server
(não precisa voltar via ALB)

ALB tem:
├─ Inbound rule (0.0.0.0/0:80)
└─ Outbound rule (default to NAT ou app-sg:8080)

Result:
- Request chega Client: ALB traduz
- Response sai apfrom: App faz routing normal
- Simétrico! ✓
```

---

## VPC Peering Routing

### Setup

```
VPC-A: 10.0.0.0/16        VPC-B: 10.1.0.0/16
├─ Private Subnet          ├─ Private Subnet
│  └─ 10.0.1.0/24          │  └─ 10.1.1.0/24
└─ Route Table             └─ Route Table
   ├─ 10.0.0.0/16 local       ├─ 10.1.0.0/16 local
   └─ 10.1.0.0/16 pcx-xxxx    └─ 10.0.0.0/16 pcx-xxxx

Sem rota para peering:
EC2 A (10.0.1.5) → quer falar com EC2 B (10.1.1.5)
└─ Lookup 10.1.1.5
└─ Matches 10.1.0.0/16? Não (10.1.0.0 vs 10.0.0.0)
└─ Matches 0.0.0.0/0? Talvez, vai para NAT/IGW
└─ Pacote pega não chega em VPC-B

Com rota:
10.1.0.0/16 → pcx-xxxxx
└─ Pacote vai direto para peering (sem sair VPC)
```

---

## Transit Gateway Routing (Advanced)

### Problema Multi-VPC Sem TGW

```
VPC-A ↔ VPC-B (peering)
VPC-A ↔ VPC-C (peering)
VPC-B ↔ VPC-C (peering?)

Com N VPCs:
Pairwise connections = N*(N-1)/2
- 3 VPCs = 3 peerings
- 10 VPCs = 45 peerings
- 100 VPCs = 4.950 peerings ❌
```

### Solução: Transit Gateway (Hub-and-Spoke)

```
     VPC-A
      ↓
VPC-D → [Transit Gateway] ← VPC-B
      ↑
     VPC-C

Cada VPC:
├─ Attachment para TGW
└─ Route: 10.0.0.0/8 → tgw-xxxxx (cobre todos os 10.0.0.0-10.255.255.255)

TGW:
├─ Rota interna de VPC-A → VPC-A routes
├─ Rota interna de VPC-B → VPC-B routes
└─ Rota interna de VPC-C → VPC-C routes

Total: 3 attachments, 1 TGW, escalável!
```

---

## Route53 vs Routing VPC

### Route53 (DNS)

```
Seu aplicativo faz:
GET https://db.internal.company.com

Route53 resolver:
└─ db.internal.company.com → 10.0.21.5

Sistema operacional:
└─ Conectar a 10.0.21.5

Route Table avalia:
└─ 10.0.21.5 matches 10.0.0.0/16 local
└─ Manda para database
```

### Route53 Private Hosted Zones

```
VPC-A: 10.0.0.0/16
└─ Private Hosted Zone: internal.company.com
   └─ db IN A 10.0.21.5
   └─ api IN A 10.0.11.5

EC2 em VPC-A faz:
nslookup db.internal.company.com
└─ Route53 responde: 10.0.21.5

EC2 em VPC-B (sem private zone):
nslookup db.internal.company.com
└─ Route53 diz: (nãoexiste) ou timeout

Solução:
Associar Private Hosted Zone a múltiplas VPCs!
└─ Então VPC-B também resolve db.internal.company.com → 10.0.21.5 VPC-A
└─ Route Table em VPC-B: 10.0.0.0/16 → pcx (peering para VPC-A)
```

---

## Troubleshooting Roteamento

### Ferramenta 1: traceroute / mtr

```
$ traceroute 10.0.21.5

hop 1: 169.254.169.254 ← VPC metadata service
hop 2: 10.0.21.5 ← destination reached

Se não passa:
hop 1-2: timeout
└─ Problema: Route table não tem rota
└─ Ou NACL bloqueia
└─ Ou Security Group bloqueia
```

### Ferramenta 2: VPC Reachability Analyzer

```
AWS Console → VPC → Reachability Analyzer

Test: Can EC2-A reach RDS-B?

Resultado:
✓ Reachable via (mostra o caminho)
  ├─ Route: 10.0.21.0/24 → local
  ├─ Network ACL: allow
  └─ Security Group: allow

✗ Not reachable because:
  └─ Network ACL rule 120 blocks TCP 3306
     Suggestion: add rule 110 TCP 3306
```

### Ferramenta 3: VPC Flow Logs Analysis

```
START END SRCADDR DSTADDR SRCPORT DSTPORT PROTOCOL PACKETS BYTES ACTION
...
645251300 10.0.1.100 10.0.21.5 54321 3306 6 1 64 REJECT

⚠️ Reject = problema!

Checklist:
1. Route Table: 10.0.21.5 matched? (matches 10.0.0.0/16 local? Sim)
2. NACL Inbound: 3306 allowed? (check private-app NACL)
3. NACL Outbound: ephemeral allowed? (1024-65535?)
4. SG: 3306 from app-sg? (check rds-sg inbound)
```

---

## Best Practices

### ✅ SIM
- Route tables granular por subnet-tier (public, app, data)
- Route específicas ANTES de default (0.0.0.0/0)
- Documentar rotas: "Por que essa rota existe?"
- Usar TGW para multi-VPC (em vez de mesh peering)
- BGP para on-premises dynamic routing
- Monitorar com VPC Reachability Analyzer

### ❌ NÃO
- Default route (0.0.0.0/0) para vários próximos hops!
- Routes com CIDRs sobrepostos (confusão)
- Esquecer route quando cria subnet
- Manual routing para multi-VPC (scale nightmare)
- Ignorer symmetric routing (one-way traffic)

---

## Próximos Passos

- **[06 - NAT e VPC Endpoints](06-nat-endpoints.md)**: Otimizar próximos hops
- **[08 - VPC Peering e Transit Gateway](08-vpc-peering-transit-gateway.md)**: Advanced peering
- **[10 - Monitoramento](10-monitoramento-troubleshooting.md)**: Debug routing

---

**Anterior**: [04 - Segurança em VPC](04-seguranca-nlb-acl.md)  
**Próximo**: [06 - NAT e VPC Endpoints](06-nat-endpoints.md)
