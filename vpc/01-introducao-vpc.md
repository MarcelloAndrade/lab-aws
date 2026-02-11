# 01 - Introdução à VPC (Virtual Private Cloud)

## O que é VPC?

VPC é sua **rede privada isolada dentro da AWS**. É o primeiro passo de isolamento de segurança - como se você tivesse aluguel de um data center privado, mas virtualizado.

### Analogia do Mundo Real

```
AWS Region = Região Geográfica
  └─ VPC = Seu Prédio Privado
      └─ Subnet = Andar do prédio
          └─ EC2, RDS, etc = Escritórios naquele andar
```

---

## Por Que Usar VPC?

### 1. **Isolamento**
- Sua VPC é isolada logicamente das outras
- Apenas você controla a comunicação de entrada/saída
- AWS não pode ver seu tráfego

### 2. **Controle Total de Rede**
- Você define blocos CIDR
- Você define subnets
- Você define rotas
- Você define regras de firewall

### 3. **Segurança em Camadas**
```
┌─────────────────────────────────┐
│  Aplicação (API, App Logic)     │  Camada 5
├─────────────────────────────────┤
│  Security Group (Stateful)      │  Camada 4 (Instance-level)
├─────────────────────────────────┤
│  Network ACL (Stateless)        │  Camada 3 (Subnet-level)
├─────────────────────────────────┤
│  VPC Routing                    │  Camada 2
├─────────────────────────────────┤
│  AWS Physical Infrastructure    │  Camada 1
└─────────────────────────────────┘
```

### 4. **Conectividade Flexível**
- On-premises via VPN
- Outras VPCs via Peering ou Transit Gateway
- Internet via Internet Gateway
- AWS Services via VPC Endpoints

### 5. **Compliance & Regulatória**
- HIPAA: VPCs isoladas por conta/região
- PCI DSS: Networking controlado
- SOC 2: Auditoria e logging via VPC Flow Logs

---

## Arquitetura Básica de VPC

```
┌──────────────────────────────────────────┐
│     AWS Region (us-east-1)              │
│                                          │
│  ┌──────────────────────────────────┐   │
│  │  VPC: 10.0.0.0/16               │   │
│  │                                  │   │
│  │  ┌──────────────┐ Availability   │   │
│  │  │  Availability│ Zone: us-east-1a
│  │  │  Zone: a     │ ┌────────────┐│   │
│  │  │ ┌──────────┐ │ │ Public     ││   │
│  │  │ │Public S. │ │ │ Subnet     ││   │
│  │  │ │10.0.1/24 │─┼─│10.0.1.0/24││   │
│  │  │ └──────────┘ │ │(NAT, IGW)  ││   │
│  │  │             − │ └────────────┘│   │
│  │  │ ┌──────────┐ │ ┌────────────┐│   │
│  │  │ │Private S.│ │ │ Private    ││   │
│  │  │ │10.0.2/24 │ │ │ Subnet     ││   │
│  │  │ └──────────┘ │ │10.0.2.0/24 ││   │
│  │  │             − │ │(RDS, Cache)││   │
│  │  │  ┌────────┐  │ └────────────┘│   │
│  │  │  │RDS DB  │  │                │   │
│  │  │  │(Multi  │  │                │   │
│  │  │  │ AZ)    │  │                │   │
│  │  └──┴────────┴──┘                │   │
│  │                                  │   │
│  │  ┌──────────────┐ ┌────────────┐│   │
│  │  │  Availability│ │ Public     ││   │
│  │  │  Zone: us-   │ │ Subnet     ││   │
│  │  │  east-1b     │ │10.0.3.0/24 ││   │
│  │  │ ┌──────────┐ │ └────────────┘│   │
│  │  │ │Public S. │ │  ┌────────────┐  │
│  │  │ │10.0.3/24 │─┼──│ ALB        │  │
│  │  │ └──────────┘ │  │            │  │
│  │  │             − │  └────────────┘  │
│  │  │ ┌──────────┐ │                   │
│  │  │ │Private S.│ │                   │
│  │  │ │10.0.4/24 │ │                   │
│  │  │ └──────────┘ │                   │
│  │  └──────────────┘                   │
│  │                                  │   │
│  │  [Internet Gateway]              │   │
│  │        ↓                         │   │
│  │   ┌─────────┐                    │   │
│  │   │ Internet│ ←─ Rotas          │   │
│  │   └─────────┘                    │   │
│  │                                  │   │
│  └──────────────────────────────────┘   │
│         ↓            ↓      ↓            │
└────────────────────────────────────────── 
        Internet     VPN    AWS
                           Services
```

---

## Componentes Essenciais

### 1. **VPC (o principal bloco)**
- Bloco CIDR: Ex: `10.0.0.0/16` (65,536 endereços)
- Isolada logicamente
- Pode ser deletada, mas com cuidado (dados não são deletados)

### 2. **Subnets**
- Subdivisão da VPC
- Sempre associadas a 1 Availability Zone (AZ)
- Bloco CIDR menor (ex: `10.0.1.0/24` - 256 endereços)
- **Public**: tem rota para IGW
- **Private**: não tem rota para IGW

### 3. **Internet Gateway (IGW)**
- Permite comunicação entre VPC e Internet
- Escalável, redundante, high-available
- Precisa de:
  - Attachment à VPC
  - Rota na Route Table apontando para IGW

### 4. **Route Table (Tabela de Rotas)**
- Define para onde o tráfego vai
- Criada por padrão (local route)
- Pode associar múltiplas subnets a 1 Route Table
- Avaliada em ordem de especificidade (CIDR mais longo ganha)

### 5. **Network ACL**
- Firewall em nível de subnet
- Stateless (default deny all, depois liberar)
- Ordem importa (processado em ordem)

### 6. **Security Group**
- Firewall em nível de instance
- Stateful (responses automaticamente aceitas)
- Usa nomes ou IDs de outros SGs

---

## Ciclo de Vida de um Pacote em VPC

```
┌─────────────────────────────────────────┐
│  1. Instância origina pacote            │
│     - IP origem: 10.0.1.X               │
│     - IP destino: 8.8.8.8               │
└─────────────────────────────────────────┘
                ↓
┌─────────────────────────────────────────┐
│  2. Security Group avalia               │
│     - Outbound rule existe?             │
│     - Sim → Permite                     │
└─────────────────────────────────────────┘
                ↓
┌─────────────────────────────────────────┐
│  3. Network ACL avalia                  │
│     - Outbound rule existe?             │
│     - Sim → Permite                     │
└─────────────────────────────────────────┘
                ↓
┌─────────────────────────────────────────┐
│  4. Route Table busca rota              │
│     - Destino 8.8.8.8 → 0.0.0.0/0       │
│     - Próximo passo: IGW                │
└─────────────────────────────────────────┘
                ↓
┌─────────────────────────────────────────┐
│  5. IGW translada IP                    │
│     - Origem: 10.0.1.X → EIP/NAT        │
│     - Envia para Internet                │
└─────────────────────────────────────────┘
                ↓
┌─────────────────────────────────────────┐
│  6. Resposta volta de 8.8.8.8           │
│     - Destino: EIP                      │
│     - IGW translada de volta             │
│     - Chega Instância                   │
└─────────────────────────────────────────┘
```

---

## Conceitos Importantes: CIDR

### O que é CIDR?

CIDR (Classless Inter-Domain Routing) = notação compacta para ranges de IP.

```
10.0.0.0/16
└─ /16 = primeiros 16 bits são "rede", resto é "host"
   = 2^(32-16) = 2^16 = 65,536 endereços

10.0.1.0/24
└─ /24 = primeiros 24 bits são "rede"
   = 2^(32-24) = 2^8 = 256 endereços
```

### Qual tamanho de VPC?

**Regra**: Sempre comece maior, nunca é possível aumentar CIDR de VPC!

```
/16 = 65,536 endereços     ← Padrão recomendado para maioria
/20 = 4,096 endereços      ← Para POC/dev pequeno
/28 = 16 endereços         ← Nunca faça isso em produção
```

**Por que?**
- Subnets dentro da VPC sempre são maiores (ex `/24`)
- VPC Peering conecta VPCs - não podem ter CIDRs sobrepostos
- Documentar permite crescimento futuro

---

## VPC vs Conta AWS vs Região

| Aspecto | VPC | Conta AWS | Região |
|--------|-----|-----------|--------|
| Isolamento | Lógico (rede) | Administrativo (billing) | Geográfico |
| Acesso | Via credentials de EC2 | Via IAM | Via coordenadas físicas |
| Dados | Dentro da VPC | Múltiplas VPCs | Múltiplas contas |
| Custo | Baseado em XData transfer | Baseado em uso total | Varia por região |

---

## Modelo de Responsabilidade Compartilhada

```
┌────────────────────────────────┐
│ AWS gerencia                   │
├────────────────────────────────┤
│ - Infraestrutura física        │
│ - Hyper-visor/isolamento       │
│ - Disponibilidade (99.99%)     │
└────────────────────────────────┘
              ↑
        Boundary
              ↓
┌────────────────────────────────┐
│ Você gerencia                  │
├────────────────────────────────┤
│ - CIDR e subnets              │
│ - Security Groups & NACLs     │
│ - Routes e gateways           │
│ - Firewall de aplicação       │
│ - Logging e monitoring        │
└────────────────────────────────┘
```

---

## Erros Comuns ao Começar com VPC

### ❌ Erro 1: CIDR muito pequeno
```
VPC: 10.0.0.0/28 (16 endereços)
Problema: AWS reserva 5 endereços
  - 10.0.0.0: Network ID
  - 10.0.0.1: AWS Router
  - 10.0.0.2: AWS DNS
  - 10.0.0.3: Future use
  - 10.0.0.15: Broadcast
Sobram apenas 11 para instâncias!
```

**Solução**: `/20` mínimo, `/16` padrão

### ❌ Erro 2: Tudo em uma subnet
```
EC2 [Web]
EC2 [Database]
Problema: Sem separação de camadas
         Falha em AZ = perda total
```

**Solução**: Mínimo 2 AZs, ao menos 2 subnets por AZ

### ❌ Erro 3: Security Group aberto demais
```
Inbound: 0.0.0.0/0 porta 22 (SSH)
Problema: Qualquer um na internet pode tentar conectar
```

**Solução**: SSH apenas de IP office/bastion

### ❌ Erro 4: NAT Gateway em subnet privada
```
Problema lógico: Packet privado não consegue sair sem IGW
```

**Solução**: NAT Gateway está em subnet PUBLIC (tem IGW)

### ❌ Erro 5: Sem monitoramento
```
Problema: Não sabe o que acontece com o tráfego
```

**Solução**: VPC Flow Logs para tudo

---

## Resumo

| Conceito | Resumo |
|----------|--------|
| **VPC** | Rede privada isolada na AWS |
| **Subnet** | Divisão lógica de VPC em 1 AZ |
| **IGW** | Porta de entrada/saída para Internet |
| **Route Table** | Define para onde tráfego vai |
| **Security Group** | Firewall stateful em nível de instance |
| **NACL** | Firewall stateless em nível de subnet |
| **CIDR** | Notação para ranges de IP |

**Lembre-se**: VPC é o foundation de segurança na AWS. Dominar VPC é dominar 80% de segurança de nuvem.

---