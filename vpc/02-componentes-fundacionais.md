# 02 - Componentes Fundacionais de VPC

## Visão Geral

Os componentes principais de uma VPC são como as estruturas físicas de um edifício:

```
VPC (o edifício)
├─ Subnets (andares)
│  ├─ Route Table (planta baixa com rotas)
│  └─ Network ACL (segurança do andar)
├─ Internet Gateway (entrada/saída do edifício)
├─ NAT Gateway/Instance (saída filtrada)
└─ VPC Endpoints (conexões internas com AWS)
```

---

## 1. Subnets

### O que é uma Subnet?

Subnet é uma **subdivisão lógica da VPC** sempre associada a **uma única Availability Zone (AZ)**.

### Criação de Subnet

```
VPC CIDR: 10.0.0.0/16
└─ Public Subnet-1A:   10.0.1.0/24    (256 endereços)
└─ Public Subnet-1B:   10.0.2.0/24    (256 endereços)
└─ Private Subnet-1A:  10.0.11.0/24   (256 endereços)
└─ Private Subnet-1B:  10.0.12.0/24   (256 endereços)
```

### Endereços Reservados (AWS reserva 5)

Para cada subnet `/24`:

```
10.0.1.0   - Network Address (AWS)
10.0.1.1   - VPC Router (AWS)
10.0.1.2   - DNS Resolver (AWS)
10.0.1.3   - Reserved for future use (AWS)
10.0.1.255 - Broadcast Address (AWS)

Usáveis: 10.0.1.4 até 10.0.1.254 = 251 endereços
```

### Public vs Private Subnet

| Característica | Public | Private |
|---|---|---|
| Route para IGW | ✅ Sim | ❌ Não |
| Route para NAT | ❌ Não | ✅ Sim |
| IP Público (auto) | Pode ter | Não tem |
| Acesso Internet (saída) | ✅ Direto | ✅ Via NAT |
| Acesso Internet (entrada) | ✅ Direto | ❌ Não |
| Uso típico | Web, ALB, NAT | Database, Cache, App |

### Estrutura Recomendada

```
Produção com alta disponibilidade:

VPC: 10.0.0.0/16

Tier 1 (Web/Public):
- Public Subnet 1A:   10.0.1.0/24
- Public Subnet 1B:   10.0.2.0/24
- Public Subnet 1C:   10.0.3.0/24

Tier 2 (Application):
- Private Subnet 1A:  10.0.11.0/24
- Private Subnet 1B:  10.0.12.0/24
- Private Subnet 1C:  10.0.13.0/24

Tier 3 (Database):
- Private Subnet 1A:  10.0.21.0/24
- Private Subnet 1B:  10.0.22.0/24
- Private Subnet 1C:  10.0.23.0/24
```

**Por que 3 AZs?**
- RDS Multi-AZ precisa de 2+ subnets em 2+ AZs
- ECS/ASG pode fazer rollout com 3 AZs
- Falha de 1 AZ = mantém 66% da capacidade

---

## 2. Route Table

### O que é Route Table?

Route Table é uma **tabela de decisão**: "SE destino é X, ENVIAR PARA Y".

### Estrutura de uma Rota

```
Destination CIDR          Next Hop              Origin
────────────────────────────────────────────────────────
10.0.0.0/16              local                 local
0.0.0.0/0 (default)      igw-xxxxx             manual
172.16.0.0/12            pcx-xxxxx (peering)   manual
```

### Matching: Mais Específico Ganha

Se você tem rotas:
```
10.0.0.0/8   → IGW
10.0.1.0/24  → NAT Gateway
10.0.1.0/25  → VPN
```

Um pacote para `10.0.1.5`:
- Matches `/8` ✓ (10.0.1.5 está em 10.0.0.0/8)
- Matches `/24` ✓ (10.0.1.5 está em 10.0.1.0/24)
- Matches `/25` ✗ (10.0.1.5 não está em 10.0.1.0/25)

**Vence**: `/24` (mais específico ganha)
**Resultado**: Pacote vai para NAT Gateway

### Rota Local (sempre presente)

```
Destination: 10.0.0.0/16
Next Hop: local
Origin: local
Type: local
State: blackhole (se endpoint não existe)
```

**Importância**: Rota local permite comunicação entre subnets da mesma VPC.

### Route Table Padrão vs Custom

Toda VPC tem uma **Main Route Table** (padrão):

```
AWS cria automaticamente:
Destination: 10.0.0.0/16
Next Hop: local

Você precisa adicionar:
- IGW para public subnets
- NAT para private subnets
```

### Associação de Route Table

```
Diagram:
        Route Table A
       /     |       \
    Sub 1   Sub 2   Sub 3
    
Uma Route Table pode servir múltiplas subnets.
Mas cada subnet é associada a exatamente 1 Route Table.
(A padrão é a fallback se não especificada)
```

### Exemplo de Roteamento Completo

```
Route Table: Public-Web-RT

Destination          Next Hop          Uso
─────────────────────────────────────────────
10.0.0.0/16         local             Intra-VPC
0.0.0.0/0           igw-abcd1234      Internet default
172.16.0.0/12       vpn-connection    On-premises
10.20.0.0/16        pcx-peering       Outra VPC


Route Table: Private-App-RT

Destination          Next Hop          Uso
─────────────────────────────────────────────
10.0.0.0/16         local             Intra-VPC
0.0.0.0/0           nat-xyz789        Internet via NAT
172.16.0.0/12       vpn-connection    On-premises
```

---

## 3. Internet Gateway (IGW)

### O que é IGW?

IGW é um **componente VPC que permite comunicação com Internet**.

Características:
- ✅ Escalável e redundante
- ✅ Sem tax (sem cobrar por dados)
- ✅ Oferece IP público/EIP
- ✅ Uma VPC = um IGW (máximo)

### Funcionamento

```
┌─────────────────────────────────────┐
│  EC2 (10.0.1.50)                    │
│  Subnet: Public-1A                  │
│  EIP: 203.0.113.5                   │
│                                     │
│  Pacote saindo:                     │
│  From: 10.0.1.50:12345              │
│  To: 8.8.8.8:53                     │
└─────────────────────────────────────┘
              ↓
      Route Table lookup
      Destino 8.8.8.8 → 0.0.0.0/0 → IGW
              ↓
┌─────────────────────────────────────┐
│  Internet Gateway (NAT translation) │
│                                     │
│  From: 10.0.1.50:12345              │
│  To:   203.0.113.5:xxxxx            │
│  (porta re-mapeada automaticamente)  │
└─────────────────────────────────────┘
              ↓
         🌐 Internet
```

### Pré-requisitos

Para IGW funcionar: **3 espaços**

1. **IGW criado e attached à VPC**
   ```
   Resource: Internet Gateway
   State: attached
   VPC: vpc-xxxxx
   ```

2. **Subnet tem rota para IGW**
   ```
   Route Table: public-rt
   Destination: 0.0.0.0/0
   Next Hop: igw-xxxxx
   ```

3. **EC2 tem IP público (EIP ou auto-assign)**
   ```
   Primary private IPv4: 10.0.1.50
   Elastic IP: 203.0.113.5
   ```

   Se EC2 não tiver IP público:
   ```
   Pacote sai (local route ok)
   → Chega no IGW
   → IGW não encontra mapeamento
   → Pacote é dropped
   ```

---

## 4. NAT Gateway

### O que é NAT?

NAT = Network Address Translation. Traduz endereços IP privados em públicos.

### NAT Gateway (Managed by AWS)

```
Características:
✅ Gerenciado pela AWS
✅ Alta disponibilidade (redundância interna)
✅ Escalável automaticamente
✅ Precisa de EIP
✅ Precisa de subnet pública (para ter acesso ao IGW)
❌ Caro (~$32/mês por NAT Gateway)
```

### NAT Instance (Self-Managed com EC2)

```
Características:
✅ Barato se usar t2.micro free tier
✅ Mais controle fino
❌ Você gerencia a instância
❌ Deve desabilitar source/destination check
❌ Sem redundância automática
```

### Funcionamento PAT (Port Address Translation)

```
Private Subnet EC2:
IP: 10.0.11.5
Port: 54321
Enviando para: 8.8.8.8:53

↓ (passa pelo NAT Gateway)

NAT Gateway traduz:
IP: 203.0.113.100 (EIP)
Port: 45678 (aleatório)
Enviando para: 8.8.8.8:53

↓ (resposta volta)

8.8.8.8 responde para: 203.0.113.100:45678

↓ (NAT reversa)

EC2 recebe de: 8.8.8.8:53
Local: 10.0.11.5:54321
```

### Localização NAT Gateway

```
⚠️ IMPORTANTE:

NAT Gateway precisava estar em subnet PUBLIC
         ↓
   Porque precisa de EIP
         ↓
   EIP é público
         ↓
   É preciso de rota para IGW
         ↓
   Subnet com rota para IGW = Public Subnet
```

### Roteamento para NAT

```
VPC: 10.0.0.0/16

Public Subnet (10.0.1.0/24):
Route Table:
  10.0.0.0/16     → local
  0.0.0.0/0       → igw-xxxxx

Private Subnet (10.0.11.0/24):
Route Table:
  10.0.0.0/16     → local
  0.0.0.0/0       → nat-yyyyy
        (NAT está no public subnet, mas roteia para lá)
```

### Multi-NAT para HA

```
Produção:

Public Subnet 1A:
  NAT Gateway A → EIP A
  
Public Subnet 1B:
  NAT Gateway B → EIP B

Private Subnet 1A:
  Route: 0.0.0.0/0 → NAT A
  
Private Subnet 1B:
  Route: 0.0.0.0/0 → NAT B

Benefício: Se AZ 1A falha, 1B ainda funciona
```

---

## 5. Network ACL (NACL)

### O que é NACL?

NACL é um **firewall stateless em nível de subnet**.

| Aspecto | NACL | Security Group |
|---|---|---|
| Nível | Subnet | Instance (ENI) |
| Estado | Stateless | Stateful |
| Default | Nega tudo | Nega tudo |
| Performance | Rápido | Rápido |
| Ordem | Importa (números) | Não importa |

### Estrutura NACL

```
Inbound Rules:
Rule #  Protocol  Port Range  CIDR         Allow/Deny
────────────────────────────────────────────────────
100     TCP       80          0.0.0.0/0    Allow
110     TCP       443         0.0.0.0/0    Allow
120     TCP       22          10.0.0.0/8   Allow
*       *         *           *            Deny (default)

Outbound Rules:
Rule #  Protocol  Port Range  CIDR         Allow/Deny
────────────────────────────────────────────────────
100     TCP       1024-65535  0.0.0.0/0    Allow  (ephemeral)
110     SQL       3306        10.0.0.0/16  Allow  (database)
*       *         *           *            Deny (default)
```

### Stateless vs Stateful

**NACL (Stateless)**:
```
Client → Server (porta 12345 → 80)
Precisa:
  Inbound rule: porta 80
  Outbound rule: porta 12345 (ephemeral)
  
Se esquecer outbound, resposta fica presa!
```

**Security Group (Stateful)**:
```
Client → Server (porta 12345 → 80)
Precisa:
  Inbound rule: porta 80
  
Outbound automático (stateful tracking)
```

### Ephemeral Ports

```
Client faz requisição na porta 54321 para servidor porta 80:

Request: 54321 → 80
Response: 80 → 54321 (precisa estar liberado em outbound!)

Faixa ephemeral do Linux: 32768-61000
Faixa ephemeral do Windows: 1024-65535

NACL precisa liberar a faixa correspondente!

Ao abrir para tudo (0.0.0.0/0):
Outbound: TCP 1024-65535 (covers ephemeral)
```

### Exemplo Prático: Multi-Tier NACL

```
Public Subnet NACL:
━━━━━━━━━━━━━━━━━

Inbound:
100 TCP 80 (HTTP)         0.0.0.0/0        Allow
110 TCP 443 (HTTPS)       0.0.0.0/0        Allow
120 TCP 22 (SSH)          10.0.0.0/16      Allow
*   *   *                 *                Deny

Outbound:
100 TCP 443               0.0.0.0/0        Allow (internet)
110 TCP 3306              10.0.0.0/16      Allow (app→db)
120 UDP 53                0.0.0.0/0        Allow (DNS)
*   *   *                 *                Deny


Private (App) Subnet NACL:
━━━━━━━━━━━━━━━━━━━━━

Inbound:
100 TCP 8080              10.0.1.0/24      Allow (web→app)
110 TCP 22                10.0.0.0/16      Allow (bastion→app)
120 TCP 1024-65535        0.0.0.0/0        Allow (ephemeral responses)
*   *   *                 *                Deny

Outbound:
100 TCP 3306              10.0.21.0/24     Allow (app→db)
110 TCP/UDP 53            0.0.0.0/0        Allow (DNS)
120 TCP 443               0.0.0.0/0        Allow (https APIs)
*   *   *                 *                Deny
```

---

## 6. Security Group (SG)

### Conceito

Security Group é **firewall stateful em nível de ENI (Elastic Network Interface)**.

### Stateful: O que significa?

```
Regra inbound: TCP porta 80 de 0.0.0.0/0 Allow

Client faz: GET / HTTP/1.1
  Source: 203.0.113.1:54321 → 10.0.1.100:80
  
Resposta que servidor tenta enviar:
  Source: 10.0.1.100:80 → 203.0.113.1:54321
  
Security Group permite automaticamente a resposta!
(Sem precisar adicionar regra de outbound)
```

### Tipos de Regras

**Inbound**:
```
Type          Protocol  Port Range  CIDR        Description
──────────────────────────────────────────────────────────
HTTP          TCP       80          0.0.0.0/0   Web traffic
SSH           TCP       22          203.0.113.1 My office
Custom TCP    TCP       8080        sg-xxxxx    App SG
All traffic   -1        -           -           (All = muito aberto)
```

**Outbound** (padrão: tudo permitido):
```
Type          Protocol  Port Range  CIDR        Description
──────────────────────────────────────────────────────────
All traffic   -1        -           0.0.0.0/0   (default)
```

### SG referenciando SG

```
Web SG:
Inbound: TCP 80/443 from 0.0.0.0/0

ALB SG:
Inbound: TCP 8080/8081 from sg-web-xxxxx (Web SG)
(Permite tráfego apenas das instâncias que têm Web SG)

Benefício: Dinâmico - nova instância com Web SG já consegue
           conectar sem mudar rules
```

### Default SG Perigoso

```
VPC padrão cria Security Group default:

Inbound:
  (none)

Outbound:
  All traffic allowed

Se você usar esse SG:
- Instâncias conseguem se comunicar (self-reference)
- Mas instâncias conseguem conectar em tudo na internet
- Risco: Dados podem vazar
```

---

## Resumo da Cascata

```
Internet → [IGW] → [NACL Inbound] → [SG Inbound] → EC2
EC2 → [SG Outbound] → [NACL Outbound] → [IGW/NAT/Local] → Destino
```

### Checklist de Troubleshooting

Se EC2 não consegue se conectar ao banco de dados em outra subnet:

- ✅ EC2 tem route até subnet do DB?
- ✅ NACL inbound do DB subnet permite tráfego?
- ✅ NACL outbound do app subnet permite tráfego?
- ✅ SG do DB permite porta 3306 de SG da app?
- ✅ RDS (se for) tem subnet group correio?
- ✅ Security Group do RDS permite tráfego?

---