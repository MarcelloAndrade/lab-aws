# 06 - NAT Gateway e VPC Endpoints

## NAT Gateway vs NAT Instance

### NAT Gateway (Gerenciado AWS)

**Pros:**
- ✅ AWS gerencia infraestrutura
- ✅ Alta disponibilidade (redundância automática)
- ✅ Escalável automaticamente (10 Gbps+)
- ✅ Sem Security Groups para gerenciar
- ✅ Sem OS patches

**Cons:**
- ❌ Caro (~$32/mês por NAT Gateway + dados)
- ❌ Sem controle fino (não consegue inspecionar tráfego)
- ❌ Um por AZ recomendado (para HA)

### NAT Instance (Self-Managed com EC2)

**Pros:**
- ✅ Barato (free tier t2.micro ou t2.small)
- ✅ Controle total (pode instalar software, logar tráfego)
- ✅ Sistema completo (não só NAT)

**Cons:**
- ❌ Você gerencia updates, patches, segurança
- ❌ Sem HA automática (você monitora saúde)
- ❌ Deve desabilitar source/destination check
- ❌ Pode ficar lento sob carga
- ❌ Paga por data transfer igual

### Recomendação

```
Produção:     Use NAT Gateway
POC/Dev:      NAT Instance (mais barato)
Compliance:   NAT Instance (logar tráfego fino)
Alta carga:   NAT Gateway (escala automática)
```

---

## Arquitetura de NAT Gateway

```
┌─────────────────────────────┐
│ Public Subnet: 10.0.1.0/24  │
│                             │
│ NAT Gateway                 │
│ ├─ ENI: 10.0.1.200          │
│ └─ EIP: 203.0.113.100       │
│                             │
└─────────────────────────────┘
            ↑
    Gateway Route
            ↑
┌────────────────────────────────┐
│ Private Subnet: 10.0.11.0/24  │
│                                │
│ EC2 App Server                │
│ └─ IP: 10.0.11.5              │
│    └─ Quer sair para Internet │
│    └─ Faz request para 8.8.8.8│
│                                │
└────────────────────────────────┘
```

### Fluxo de Pacote

1. **EC2 em subnet privada origina pacote**
   ```
   From: 10.0.11.5:54321
   To: 8.8.8.8:53 (DNS)
   ```

2. **Route table lookup em private-app-rt**
   ```
   Destino: 8.8.8.8/32 matches 0.0.0.0/0?
   Sim → next hop: nat-xxxxx
   ```

3. **Packet chega no NAT Gateway**
   ```
   Nota: NAT está em different subnet (public)
   Mas internamente AWS roteia via internal path
   Sem passar pelo IGW primeiro
   ```

4. **NAT Gateway traduz**
   ```
   From: 10.0.11.5:54321 → 203.0.113.100:60000 (aleatório)
   To: 8.8.8.8:53 (mesmo)
   ```

5. **NAT envia para IGW**
   ```
   IGW vê: EIP 203.0.113.100 está attached
   IGW deixa passar (é válido elastic IP)
   Pacote sai para Internet
   ```

6. **Resposta volta**
   ```
   8.8.8.8 → 203.0.113.100:60000
   IGW recebe e traduz
   NAT recebe, faz reverse NAT
   Pacote: 8.8.8.8 → 10.0.11.5:54321
   Entrega para EC2
   ```

---

## VPC Endpoints: Sem Sair da AWS Network

### Problema: Dados saindo via Internet

```
EC2 privado quer acessar S3:

Sem VPC Endpoint:
EC2 (10.0.11.5) → NAT Gateway (203.0.113.100)
                 → IGW
                 → Internet
                 → S3 (52.x.x.x)
Custo: $0.045 per GB (per vir de/carro)
Segurança: Dados saem da AWS network!
Latência: > 100ms (vai para Internet)

Com VPC Endpoint:
EC2 (10.0.11.5) → VPC Endpoint (interno)
                 → S3 (via AWS network)
Custo: FREE (Gateway) ou $7/mês (Interface)
Segurança: Inside AWS network
Latência: < 10ms (same region)
```

### Tipos de VPC Endpoints

#### Gateway Endpoints (S3, DynamoDB)

```
Arquitetura:

┌──────────────────────────────┐
│ Private Subnet               │
│ EC2 (10.0.11.5)              │
│  └─ boto3.s3                 │
│      └─ s3.us-east-1.amazonaws.com
│      └─ Route Table lookup    │
│      └─ Matches s3 prefix    │
│      └─ Next hop: vpce-s3    │
└──────────────────────────────┘
         ↓
┌──────────────────────────────┐
│ Gateway Endpoint: S3         │
│ (AWS managed, no cost)       │
└──────────────────────────────┘
         ↓
┌──────────────────────────────┐
│ AWS S3 Service               │
│ (same region only)           │
└──────────────────────────────┘

Config:
Route Table:
  Destination: *s3.us-east-1.amazonaws.com
  Target: vpce-s3-xxxxxx (Gateway endpoint)

Ou mais específico:
  Destination: 52.92.0.0/20 (S3 CIDR)
  Target: vpce-s3-xxxxxx
```

**Limitação**: Gateway endpoints são só com CIDR-based routing (não nome).

#### Interface Endpoints (SQS, SNS, Secrets Manager, etc)

```
Arquitetura:

┌──────────────────────────────┐
│ Private Subnet               │
│ EC2 (10.0.11.5)              │
│  └─ boto3.secretsmanager     │
│      └─ secretsmanager.us-east-1.vpce.amazonaws.com
│      └─ DNS lookup           │
│      └─ Resolve para ENI     │
│      └─ 10.0.11.100 (endpoint ENI)
└──────────────────────────────┘
         ↓
┌──────────────────────────────┐
│ Interface Endpoint: VPCE     │
│ (AWS managed ENI)            │
│ - IP: 10.0.11.100            │
│ - Security Group: vpce-sg    │
│ - DNS: auto                  │
└──────────────────────────────┘
         ↓
┌──────────────────────────────┐
│ AWS Service                  │
│ (via AWS network)            │
└──────────────────────────────┘

Config:
1. Criar Interface Endpoint
   └─ Service: com.amazonaws.us-east-1.secretsmanager
   └─ Subnets: private-1a, private-1b
   └─ SG: vpce-sg (permite 443 from app-sg)

2. DNS Resolution
   └─ Private DNS enabled: Yes
   └─ Automático: secretsmanager.us-east-1.amazonaws.com
      resolva para endpoint IP

3. Security Group do Endpoint
   └─ Inbound: 443 from app-sg
```

---

## Gateway Endpoint: S3 & DynamoDB

### S3 Gateway Endpoint

```
VPC ohne endpoint:

EC2 (10.0.11.5)
├─ aws s3 ls
├─ Faz request para s3.us-east-1.amazonaws.com
├─ Route table: 0.0.0.0/0 → NAT
├─ NAT traduz (203.0.113.100)
└─ Internet call para S3 IP (52.92.x.x)

VPC com endpoint:

EC2 (10.0.11.5)
├─ aws s3 ls
├─ Faz request para s3.us-east-1.amazonaws.com
├─ Route table lookup
│  └─ Matches 52.92.0.0/20 (S3 CIDR)?
│  └─ Sim → next hop: vpce-s3-xxxxxx
├─ Direto para S3 (no Internet)
└─ Free! ✓
```

### DynamoDB Gateway Endpoint

```
Idem ao S3, mas:

Route Table:
Destination: dynamodb.us-east-1.amazonaws.com
(ou CIDR)
Target: vpce-dynamodb-xxxxx
```

### Policy em Gateway Endpoint (Optional)

```
Você pode adicionar policy para restringir:

{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::sensitive-bucket/*"
    },
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:DeleteObject",
      "Resource": "arn:aws:s3:::sensitive-bucket/*"
    }
  ]
}

Benefício: Bloqueia delete mesmo se EC2 tem IAM permission
```

---

## Interface Endpoint: Casos Comuns

### 1. Secretsmanager Interface Endpoint

```
Use case:
EC2 privado quer ler secrets para database password

Config:
1. Criar interface endpoint
  └─ Service: secretsmanager
  └─ Private DNS enabled: Yes

2. EC2 código:
   import boto3
   sm = boto3.client('secretsmanager', region_name='us-east-1')
   secret = sm.get_secret_value(SecretId='prod/db/password')
   # Automaticamente usa endpoint
   # Nenhum NAT/IGW necessário

3. Security:
   └─ EC2's SG outbound: 443 to vpce-sg
   └─ Endpoint's SG inbound: 443 from app-sg
   └─ IAM role permite: secretsmanager:GetSecretValue
```

### 2. S3 Interface Endpoint (alternativa)

```
Por que usar em vez de Gateway Endpoint?
└─ Precisa fazer requests signed (SigV4)
└─ Quer acesso cross-region (Interface consegue)
└─ Quer Private Link (que é de rede, não rotas)

Gateway:
└─ Mais simples
└─ Route table apenas

Interface:
└─ Mais controle
└─ Posso ter múltiplas por VPC
```

### 3. CloudWatch Logs Endpoint

```
EC2 quer enviar logs:
/var/log/app.log → CloudWatch Logs

Sem Endpoint:
EC2 → NAT → IGW → internet → CloudWatch

Com Endpoint:
EC2 → Endpoint ENI → CloudWatch (AWS network)

Config:
1. Interface Endpoint com service: logs
2. EC2's role: logs:PutLogEvents permission
3. Endpoint enables PrivateLink (automatic)
```

---

## Custos: NAT Gateway vs Endpoint

```
Scenario: EC2 em private subnet lê 1 TB/mês de S3

NAT Gateway: 
  Hours: $0.32/hour = $230/month
  Data: 1 TB = $45 (out) + $0 (in S3 free)
  Total: ~$275/month

Gateway Endpoint:
  Hours: $0 (gerenciado AWS)
  Data: $0 (no transfer charge)
  Total: $0/month

↳ Economia: $275/month!

Interface Endpoint:
  Hours: $7.20/month
  Data: $0.01/GB = $10/month
  Total: ~$17/month

↳ Economia: $260/month vs NAT!
```

---

## VPC Endpoint Policies

### Restrictar por IP (EC2)

```
Quer: Apenas EC2 na subnet 10.0.11.0/24 acessa secrets

Policy:
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "secretsmanager:GetSecretValue",
      "Condition": {
        "StringEquals": {
          "aws:SourceVpc": "vpc-xxxxx"
        }
      }
    },
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "*",
      "NotResources": "arn:aws:secretsmanager:*:*:secret:prod/*"
    }
  ]
}
```

---

## Troubleshooting VPC Endpoints

### Gateway Endpoint não funciona

```
EC2 tenta: aws s3 ls
Parede: "An error occurred (AccessDenied) when calling..."

Checklist:
1. Endpoint criado?
   aws ec2 describe-vpc-endpoints | grep s3
   
2. Route table tem rota?
   Destination: *s3* Target: vpce-xxxxx
   
3. EC2 tem IAM permission?
   iam:GetRole, check inline/attached policies
   
4. Security Group do endpoint?
   (Gateway não tem SG, mas check NACL inbound)
```

### Interface Endpoint não funciona

```
EC2 tenta: boto3.client('secretsmanager')
Erro: "RemoteDisconnected" ou timeout

Checklist:
1. Endpoint criado e in subnet?
   aws ec2 describe-vpc-endpoints
   Check "Subnets" field
   
2. Private DNS enabled?
   "PrivateDnsEnabled": true
   
3. Endpoint SG permite port 443?
   Inbound: TCP 443 from app-sg
   
4. NACL permite?
   Inbound: 443 from EC2 subnet
   Outbound: ephemeral (1024-65535) responses
   
5. EC2 SG permite outbound?
   outbound: 443 to vpce-sg
```

---

## Best Practices

### ✅ SIM
- Use Gateway Endpoints para S3 e DynamoDB (sempre)
- Use Interface Endpoints para Secrets Manager, SSM Parameter Store
- Enable Private DNS em interface endpoints
- Documentar quais services usam endpoints
- Monitor endpoint usage via CloudWatch
- Múltiplos AZs para interface endpoints (HA)

### ❌ NÃO
- Esquecer que gateway endpoints não precisam Security Group
- Tentar usar cross-region gateway endpoints (não funciona)
- Interface endpoints sem Private DNS (confuso com DNS)
- Deixar policies de endpoint muito abertas
- Não avisar developers que tráfego não sai da AWS

---

## Próximos Passos

- **[07 - VPN e Site-to-Site](07-vpn-connectivity.md)**: Conectar on-premises
- **[05 - Roteamento Avançado](05-roteamento-avancado.md)**: BGP e roteamento dinâmico para NAT
- **[12 - IaC com Terraform](12-iac-terraform.md)**: Codificar endpoints

---

**Anterior**: [05 - Roteamento Avançado](05-roteamento-avancado.md)  
**Próximo**: [07 - VPN e Site-to-Site](07-vpn-connectivity.md)
