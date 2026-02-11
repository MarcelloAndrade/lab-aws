# 07 - VPN e Site-to-Site Connectivity

## Tipos de Conectividade com On-Premises

```
┌──────────────────────┐        AWS
│  Data Center         │
│  (On-Premises)       │
└──────────────────────┘
        ↓
        1: AWS Site-to-Site VPN (IPsec)
        2: AWS Direct Connect (Dedicado)
        3: AWS Client VPN (Remote workers)
        4: AWS VPN CloudHub (Multi-site)
        ↓
       VPC
```

### Comparação

| Tipo | Bandwidth | Latência | Segurança | Custo |
|------|-----------|----------|-----------|-------|
| **Site-to-Site VPN** | Até 5 Gbps | 50-100ms | IPsec (bom) | $0.05/hora |
| **Direct Connect** | 50 Gbps | 10-20ms | Físico (ótimo) | $0.30/hora + priv |
| **Client VPN** | Até 5 Gbps | 50-150ms | TLS/SSL | $0.43/hora |
| **VPN CloudHub** | Até 5 Gbps | 50-100ms | IPsec (bom) | $0.05/hora |

---

## AWS Site-to-Site VPN (IPsec)

### Arquitetura Básica

```
┌──────────────────────┐           AWS
│ Your Data Center     │        VPC: 10.0.0.0/16
│ 192.168.0.0/16       │
│                      │
│ [Customer Gateway]   │   VPN Key      [VGW]
│ IP: 203.0.113.50     │   Exchange     (Virtual PG)
│ BGP ASN: 65000       ├─────────────┐
└──────────────────────┘             │
         │                           │
    Internet (encrypted)             │
         │                           │
         └──────────────────────────→
                    BGP Announces
                    DC: 192.168.0.0/16
                    VPC: 10.0.0.0/16
    
VPC Route Table:
Destination      Target
────────────────────────
10.0.0.0/16      local
192.168.0.0/16   vpn-xxxxx ← Dynamic via BGP
```

### Componentes

1. **Customer Gateway**
   ```
   Representa seu lado (on-premises)
   └─ IP público estático (seu edge device)
   └─ BGP ASN (seu número de sistema autônomo)
   
   Exemplo:
   IP: 203.0.113.50 (seu Cisco ASA / Vyos / etc)
   ASN: 65000 (for BGP)
   ```

2. **Virtual Private Gateway (VGW)**
   ```
   Representa AWS side
   └─ ASN: 64512 (AWS default, não mude)
   └─ Attached a 1 VPC
   └─ Para multi-VPC, cada uma precisa seu VGW
      (ou use Transit Gateway)
   ```

3. **VPN Connection**
   ```
   A conexão entre Customer GW e VGW
   └─ 2 Tunnels (redundância automática)
   └─ Tunnel 1: 169.254.10.1 ↔️ 169.254.10.2
   └─ Tunnel 2: 169.254.11.1 ↔️ 169.254.11.2
   
   BGP rouda tanto túneis (ECMP)
   Se túnel 1 falha, tráfego flui por túnel 2
   ```

### Setup Passo-a-Passo

#### 1. Criar Customer Gateway

```
AWS Console → VPC → Customer Gateways → Create

┌─────────────────────────────────────┐
│ Customer Gateway Details            │
├─────────────────────────────────────┤
│ Name: dc-cgw                        │
│ BGP ASN: 65000                      │
│ IP Address: 203.0.113.50            │
│ (seu edge device público IP)        │
│ Type: ipsec.1                       │
└─────────────────────────────────────┘

Resultado: cgw-xxxxx
```

#### 2. Criar Virtual Private Gateway

```
AWS Console → VPC → Virtual Private Gateways → Create

┌─────────────────────────────────────┐
│ VGW Details                         │
├─────────────────────────────────────┤
│ Name: prod-vpc-vgw                  │
│ ASN: 64512 (default)                │
│ Enable route propagation: Yes       │
└─────────────────────────────────────┘

Resultado: vgw-xxxxx

Importante: Attach VGW a VPC
VGW → Actions → Attach to VPC → prod-vpc
```

#### 3. Criar VPN Connection

```
AWS Console → VPC → VPN Connections → Create

┌──────────────────────────────────────────┐
│ VPN Connection Details                   │
├──────────────────────────────────────────┤
│ Name: dc-prod-vpn                        │
│ Customer Gateway: dc-cgw                 │
│ VGW: prod-vpc-vgw                        │
│ VPN Type: ipsec.1                        │
│ Dynamic or Static: Dynamic (BGP)         │
│ BGP Options:                             │
│   Local BGP ASN: 65000 (must match CGW)  │
│   CIDR ranges: 192.168.0.0/16            │
│ Enable acceleration: Optional            │
└──────────────────────────────────────────┘

Resultado: vpn-xxxxx
State: pending → available (≈15 minutos)
```

#### 4. Configurar On-Premises device

```
AWS Console → VPN Connection → Download Configuration

PDF com:
├─ Public IPs (VGW endpoint IPs)
├─ Pre-shared keys (para IPsec)
├─ BGP configuration
│  └─ VGW BGP IP: 169.254.10.2
│  └─ Your BGP IP: 169.254.10.1
├─ Tunnel 1 & 2 configs
└─ Example configs (Cisco, Juniper, Vyatta, etc)

Copiar para seu device:
```

#### 5. Habilitar Route Propagation

```
VPC → Route Tables → Escolher route table

Edit route propagation:
┌─────────────────────────────┐
│ Propagate routes            │
├─────────────────────────────┤
│ [✓] Virtual Private Gateway │
│     (prod-vpc-vgw)          │
└─────────────────────────────┘

Result:
Route table será automaticamente atualizado
com rotas anunciadas por BGP!

Antes:
Destination    Target
10.0.0.0/16    local

Depois (automático):
Destination    Target
10.0.0.0/16    local
192.168.0.0/16 vpn-xxxxx (aparece sozinho!)
```

---

## VPN com Roteamento Estático

### Se não puder fazer BGP

```
DC device não suporta BGP? Use static routes:

AWS Console → VPN Connection → Static Routes

┌──────────────────────────────┐
│ Static Routes Tab            │
├──────────────────────────────┤
│ CIDR: 192.168.0.0/16         │
│ Target VGW: vpn-xxxxx        │
│ State: available             │
└──────────────────────────────┘

Resultado:
Route Table será atualizado manualmente com:
Destination    Target
192.168.0.0/16 vpn-xxxxx

Mas você precisa configurar reverse em DC:
10.0.0.0/16 → via 203.0.113.X (VPN endpoint)
(AWS não anuncia automaticamente)
```

---

## AWS Direct Connect (Dedicado)

### O que é?

```
Direct Connect = Circuito dedicado físico
(não pela Internet)

┌──────────────────┐
│ Your Data Center │
│                  │
│ [Your Router]    │
└────────┬─────────┘
         │
    Dedicated Line
    (1 Gbps to 100 Gbps)
         │
     ┌───────────────────┐
     │ AWS Direct        │
     │ Connect Location  │
     │ (ex: São Paulo)   │
     └────────┬──────────┘
              │
        AWS Network
              │
         (VPC/S3/EC2)
```

### Quando Usar

```
✅ Direct Connect quando:
  - Huge data transfer (TB/month)
  - Low latency required
  - Dedicated connection (no sharing Internet)
  - Hybrid cloud 24/7 production

❌ VPN cuando é ok:
  - POC/Development
  - Intermittent connectivity
  - Small data (GB/month)
  - Cost sensitive
```

---

## AWS VPN CloudHub (Multi-Site)

### Problema: Múltiplos Data Centers

```
Sem CloudHub:
┌─────────────┐
│   DC-A      │
│ 10.20.0.0/16│
└────┬────────┘
     │        VPN
     │       ┌─────────────────────┐
     └──────→│  AWS VPC            │
     ↓       │  10.0.0.0/16        │
     ├──────→│                     │
     │      └─────────────────────┘
     │
┌─────────────┐
│   DC-B      │
│ 10.30.0.0/16│
└─────────────┘

Problema: DC-A não fala com DC-B!
         DC-A pode → VPC
         DC-B pode → VPC
         Mas DC-A ↔️ DC-B precisa IR ATRAVÉS DE VPC

Solução: CloudHub = VPC atua como hub
         DC-A → VPC → DC-B
```

### Setup CloudHub

```
1. Create múltiplos Customer Gateways
   ├─ CGW-A: 203.0.113.50 (DC-A edge)
   ├─ CGW-B: 203.0.114.50 (DC-B edge)
   └─ CGW-C: 203.0.115.50 (DC-C edge)

2. Create múltiplos VPN Connections
   ├─ VPN-A: CGW-A ↔ VGW
   ├─ VPN-B: CGW-B ↔ VGW
   └─ VPN-C: CGW-C ↔ VGW

3. Enable Route Propagation
   VPC Route Table:
   10.20.0.0/16 → vpn-A
   10.30.0.0/16 → vpn-B
   10.40.0.0/16 → vpn-C
   (via BGP route propagation)

4. DC-A fala com DC-B
   DC-A packet: src 10.20.1.5 → 10.30.1.5
   └─ Route table: 10.30.0.0/16 → vpn-B
   └─ VPN-B encapsula e envia para VPA
   └─ VPA decapsula
   └─ VPC route: 10.30.0.0/16 → vpn-B (wait, loop?)
   
   Não! AWS detecta:
   └─ Packet veio de vpn-A
   └─ Route diz vpn-B
   └─ Ok, forward para vpn-B
   └─ DC-B recebe
```

**Nota**: CloudHub funciona por roteamento inteligente.

---

## Troubleshooting VPN

### VPN Status Checks

```
AWS Console → VPN Connections → vcn-xxxxx

┌────────────────────────────────┐
│ VPN Connection Status          │
├────────────────────────────────┤
│ State: available               │
│ Tunnel 1: up ✓                 │
│ Tunnel 2: up ✓                 │
│ BGP status: established        │
└────────────────────────────────┘

✓ = Tudo bem
✗ = Problem
```

### Tunnel Down? Checklist

```
1. Conectividade para AWS endpoint?
   ping <aws-public-ip-from-vpn-connection>
   
2. Pre-shared key configurado corretamente?
   Compare PDF com device config
   
3. BGP establecido?
   show ip bgp summary (no DC router)
   
4. Sua rota padrão não sobrescreve VPN?
   Route table priority: static < BGP propagação
   
5. Firewall local bloqueando porta UDP 500/4500 (IPsec)?
   UDP port 500 (IKE)
   UDP port 4500 (IPsec NAT-T)
   
6. MTU reduzido?
   IPsec overhead: -73 bytes
   Default 1500 → efectivo 1427
   Se aplikação assume 1500, falha!
```

### VPC Route Not Updated

```
VPN está up, mas rota 192.168.0.0/16 não aparecer?

Checklist:
1. Route propagation habilitado?
   Route table → Edit propagation
   
2. BGP anúncio correto?
   On DC router: advertise 192.168.0.0/16
   
3. BGP session up?
   show ip bgp neighbors
   
4. VGW recebeu rota?
   Pode demorar até 1 minuto
   Console refresh
   
5. CIDR conflicts?
   Se VPC já tem 192.168.0.0/16
   (nunca pode tem overlapping)
```

---

## Best Practices

### ✅ SIM
- Múltiplos túnels (redundância automática)
- Usar BGP dinâmico (não static routing)
- Habilitar route propagation automática
- Monitor tunnel status via CloudWatch
- VPN accelerator para performance
- Documentar pre-shared keys (rotação regular)

### ❌ NÃO
- Assumir só 1 túnel (always 2)
- CIDR overlapping entre DC e VPC
- Deixar default route via VPN (pode criar loop)
- Esquecer IPsec ports (500, 4500 UDP)
- Ignorar MTU effects (fragmentação)
- Static routing sem necessidade (BGP melhor)

---

## Próximos Passos

- **[08 - VPC Peering e Transit Gateway](08-vpc-peering-transit-gateway.md)**: Múltiplas VPCs
- **[09 - Multi-Região](09-multi-region-hybrid.md)**: High availability global
- **[12 - IaC com Terraform](12-iac-terraform.md)**: Codificar VPN

---

**Anterior**: [06 - NAT e VPC Endpoints](06-nat-endpoints.md)  
**Próximo**: [08 - VPC Peering e Transit Gateway](08-vpc-peering-transit-gateway.md)
