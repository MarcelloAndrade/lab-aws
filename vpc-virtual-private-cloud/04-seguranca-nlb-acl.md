# 04 - Segurança em VPC

## Estratégia de Segurança em Camadas

```
Aplicação (Validação de input, autenticação, autorização)
         ↓
Security Group (Instance-level firewall, stateful)
         ↓
Network ACL (Subnet-level firewall, stateless)
         ↓
Roteamento VPC (Pode descartar pacotes indesejados)
         ↓
AWS Physical Security (Data center, fysisk isolation)
```

**Princípio**: Defense in Depth. Não confie que apenas 1 camada vai proteger.

---

## 1. Security Groups em Profundidade

### Regras Inbound (Entrada)

```
Security Group: web-sg
─────────────────────────────────────────────

Type            Protocol  Port    Source
────────────────────────────────────────
HTTP            TCP       80      0.0.0.0/0
HTTPS           TCP       443     0.0.0.0/0
SSH             TCP       22      10.0.0.0/16 (seu VPC)
Custom TCP      TCP       8080    sg-xxxx (app-sg only)
Custom ICMP     ICMP      -1      10.0.0.0/16 (ping)
```

### Regras Outbound (Saída)

```
Type                Protocol  Port    Destination
─────────────────────────────────────────────
All traffic         -1        -       0.0.0.0/0
(permite tudo por padrão)

OU mais restritivo:

HTTPS               TCP       443     0.0.0.0/0 (APIs externas)
MySQL/Aurora        TCP       3306    sg-db (only to DB)
DNS                 UDP       53      0.0.0.0/0
NTP                 UDP       123     0.0.0.0/0
```

### Padrão: Menor SG Possível

```
❌ RUIM:
Inbound: All traffic from 0.0.0.0/0
(Qualquer um no mundo consegue conectar qualquer porta)

✅ BOM:
Inbound: 
  - TCP 80 from 0.0.0.0/0 (HTTP)
  - TCP 443 from 0.0.0.0/0 (HTTPS)
  - TCP 22 from 203.0.113.0/32 (seu office)
(Apenas o necessário)
```

### Cross-SG References

```
Web Load Balancer:
├─ SG: alb-sg
│  ├─ Inbound: TCP 80/443 from 0.0.0.0/0
│  └─ Outbound: TCP 8080 to sg-app

Application Servers:
├─ SG: app-sg
│  ├─ Inbound: TCP 8080 from sg-alb
│  └─ Outbound: TCP 3306 to sg-rds

Database:
├─ SG: rds-sg
│  ├─ Inbound: TCP 3306 from sg-app
│  └─ Outbound: (all or specific)

Benefício: Se TLS termination no ALB muda, apenas
           mude destination em alb-sg outbound
           Não precisa mexer em app-sg inbound
```

### Common Mistakes

```
❌ Deixar SSH aberto para 0.0.0.0/0 (qualquer um tenta brute force)
✅ Use office IP, Bastion Host, ou Systems Manager Session Manager

❌ Múltiplos SGs com mesmo propósito (confusão)
✅ Consolidar em 1 SG por function

❌ Esquecer de outbound rules (aplicação fica lenta)
✅ Pelo menos HTTPS para APIs, DNS para lookups

❌ SG muito grande (256 rules = lento)
✅ Manter < 100 rules, usar referencias cruzadas
```

---

## 2. Network ACLs em Profundidade

### Stateless: Precisa ir-e-voltar

```
Problema clássico:

NACL Regra de Inbound:
Rule 100: Allow TCP 80 (HTTP) from 0.0.0.0/0

Cliente faz: GET / HTTP/1.1
  Request:  SRC_IP:54321 → SERVER_IP:80

Servidor responde:
  Response: SERVER_IP:80 → SRC_IP:54321

⚠️ PROBLEMA: Response é uma OUTBOUND rule!
   Precisa ter outbound rule permitindo!
```

### Ephemeral Port Ranges

```
Linux:     1024-65535
Windows:   1024-65535
macOS:     49152-65535
AWS SDK:   1024-65535   (mais permissivo)

NACL Segura para resposta:
Outbound Rule: Allow TCP 1024-65535 to 0.0.0.0/0
(cobre a maioria dos casos)
```

### Exemplo NACL Segura para Public Subnet

```
NACL: public-web-nacl

INBOUND RULES (ordem importa):
─────────────────────────────────────────────────
#    Protocol  Port      Source      Action
100  TCP       80        0.0.0.0/0   Allow (HTTP)
110  TCP       443       0.0.0.0/0   Allow (HTTPS)
120  TCP       22        10.0.0.0/16 Allow (SSH from VPC)
130  TCP       1024-65535 0.0.0.0/0  Allow (ephemeral responses)
*    All       All       All         Deny  (default)


OUTBOUND RULES:
─────────────────────────────────────────────────
#    Protocol  Port      Destination Action
100  TCP       80        0.0.0.0/0   Allow (outbound HTTP)
110  TCP       443       0.0.0.0/0   Allow (outbound HTTPS)
120  TCP       1024-65535 0.0.0.0/0  Allow (responses)
130  UDP       53        0.0.0.0/0   Allow (DNS)
140  TCP       3306      10.0.21.0/24 Allow (to database)
*    All       All       All         Deny  (default)
```

### Por que número de rule importa

```
REQUEST chega: SRC:54321 → DST:80

Avaliação:
Rule 50:  TCP 1-1000?   Não, 54321 > 1000
Rule 100: TCP 80?       Sim, e 54321 é ephemeral
          Action: Allow → Pacote passa

Rule 200: TCP 80?       Não será avaliada (já foi permitida)
```

**Ordem crescente**: Primeira match ganha. Números menores tem precedência.

### NACL vs Security Group

| Aspecto | NACL | SG |
|--------|------|-----|
| Nível | Subnet | Instance (ENI) |
| Stateless | ✓ Sim | ✗ Stateful |
| Ephemeral | Precisa abrir | Automático |
| Performance | Igual | Igual |
| Debug | Difícil (stateless) | Fácil (stateful) |
| Default | Permite tudo | Nega tudo |

**Prática**: Use NACL para catcher de último recurso, confie mais em SGs

---

## 3. VPC Flow Logs (Observabilidade)

### O que são?

Flow Logs capturam metadados de pacotes:

```
START END    SRCADDR         DSTADDR         SRCPORT  DSTPORT  PROTOCOL  PACKETS  BYTES  ACTION
645251300    10.0.1.100      172.15.0.50     54321    443      6         10       1234   ACCEPT
645251400    10.0.1.100      8.8.8.8         54322    53       17        2        256    ACCEPT
645251500    203.0.113.50    10.0.1.100      22       22       6         1        0      REJECT
```

**Nota**: Não captura payload (conteúdo), apenas metadata.

### Tipos de Flow Logs

```
Nível VPC:    Todos pacotes da VPC
Nível Subnet: Todos pacotes da subnet
Nível ENI:    Apenas dessa interface de rede
```

### Por que usar?

1. **Troubleshooting**: "Por que app não consegue conectar ao DB?"
2. **Security**: "Quem tentou conectar na porta 22?"
3. **Compliance**: "Preciso de auditoria de tráfego"
4. **Performance**: "Que aplicação usa muita banda?"

### Habilitando Flow Logs

```
VPC → Flow Logs → Create
├─ Resource: VPC
├─ Traffic Type: Both (ou Reject apenas)
├─ Destination: CloudWatch Logs (ou S3)
├─ Log group: /aws/vpc/flowlogs
├─ IAM role: PermitirEscreverLogsCloudWatch
└─ Fields: Default ou custom
```

### Análise de Flow Logs

```
Query útil (CloudWatch Insights):

# Rejections por SG
fields scraddr, dstaddr, dstport, action
| filter action = "REJECT"
| stats count() as rejections by dstport

# Top talkers
fields bytes
| stats sum(bytes) as traffic by srcaddr, dstaddr
| sort traffic desc
| limit 10

# Tráfego para Internet
fields srcaddr, dstaddr, dstport
| filter dstport = 443 and action = "ACCEPT"
| stats count() as connections by dstaddr
```

---

## 4. Política de Privacidade de IP (AWS não acessa dados)

```
Seu tráfego VPC:
└─ Criptografado entre instâncias via internal network
└─ Isolado de outras VPCs (hypervisor isolation)
└─ AWS não pode ler payload (apenas metadata em Flow Logs)

Se quer extra segurança:
└─ Encrypt IP payload (VPN, TLS/mTLS)
└─ Então mesmo Flow Logs não revela o que está sendo feito
```

---

## 5. Best Practices de Segurança

### Access Control

```
✅ Use Security Groups, não confie em NACL
✅ Use referencias cruzadas entre SGs
✅ Menor privilege possível (default deny)
✅ Documenter o porquê de cada rule

❌0.0.0.0/0 para SSH
❌ Tudo aberto "temporariamente"
❌ SG com 1000 rules (consolidar)
```

### Monitoramento

```
✅ VPC Flow Logs habilitados
✅ CloudWatch alarms para rejections
✅ Regular audit de SG rules
✅ Alert se SG negar tráfego legítimo

❌ Sem logging de tráfego
❌ Sem alertas
❌ SG rules decomposto (ninguém sabe porquê)
```

### Compliance

```
✅ Documentar security controls
✅ Manter auditoria de mudanças (CloudTrail)
✅ Encrypt data in transit (TLS)
✅ Encript data at rest (S3, RDS encryption)

❌ Dados sensíveis sobre VPC public
❌ Sem auditoria
❌ HTTP (não-criptografado)
```

### Incident Response

```
Se suspeita de breach:

1. Isolar instância
   └─ Remover de Load Balancer
   └─ Deixar enquanto faz forensics

2. Coletar evidências
   └─ VPC Flow Logs (últimas 24h)
   └─ CloudTrail events (mudanças em SG)
   └─ OS logs (SSH attempts, etc)

3. Análise post mortem
   └─ Como o attacker entrou?
   └─ Que SG combinar foi permissivo?
   └─ Como prevenir futuramente?
```

---

## 6. Estabilidade de Rede

### Keep-Alive e Connection Drops

```
Problema: Conexão perde depois de X minutos

NAT Gateway timeout padrão: 350 segundos
Load Balancer (ALB/NLB): 60 segundos (idle timeout)

Solução:
- Use keep-alive em aplicação (TCP flags)
- Ajustar idle timeout se necessário
- Usar longest-lived connections (proxies, brokers)
```

### MTU (Maximum Transmission Unit)

```
Default VPC: 1500 bytes (Ethernet standard)

Se usar VPN ou Transit Gateway:
└─ MTU reduz (precisa espaço para overhead)
└─ Pode causar fragmentação/drops

Solução: Path MTU discovery ou ajustar MTU
```

---

## 7. Casos de Uso Avançados

### Network Segmentation com NACL

```
Quarentena de subnet suspeita:
NACL inbound: Deny all
NACL outbound: Deny all

Mas deixa acesso de forensics:
└─ ALlow TCP 22 from 10.0.1.100 (seu forensics box)

Agora instância isolada mas você pode investigar
```

### Rate Limiting com NACL

```
Problema: DDoS tentando abrir muitas conexões

Solução (não ideal, mas possível):
└─ Contar connections em Flow Logs
└─ Bloquear IP com muitos establishs via NACL
   └─ Ephemeral rule #999: Deny from X.X.X.X

Melhor solução: AWS Shield Standard (automatic)
```

---

## Resumo de Checklist de Segurança de VPC

- ✅ Todas as subnets privadas exceto públicas
- ✅ NAT Gateway em subnets públicas
- ✅ Security Groups default deny, depois permite por função
- ✅ Cross-SG references usadas (não hardcoded IPs)
- ✅ NACL com regras de ephemeral port
- ✅ VPC Flow Logs habilitados
- ✅ CloudWatch alarms para eventos suspeitos
- ✅ SSH não aberto para Internet (use Session Manager ou Bastion)
- ✅ Dados sensíveis em subnets privadas
- ✅ Criptografia em transit (TLS) e at rest
- ✅ Regular audit de regras
- ✅ Documentação de porquê cada rule existe

---

## Próximos Passos

- **[06 - NAT e VPC Endpoints](06-nat-endpoints.md)**: Otimizar conectividade mantendo segurança
- **[07 - VPN e Site-to-Site](07-vpn-connectivity.md)**: Conectar on-premises
- **[10 - Monitoramento e Troubleshooting](10-monitoramento-troubleshooting.md)**: Ferramentas de debug

---

**Anterior**: [03 - Arquitetura e Design](03-arquitetura-design.md)  
**Próximo**: [05 - Roteamento Avançado](05-roteamento-avancado.md)
