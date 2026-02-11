# VPC - Virtual Private Cloud

### Fundamentos
- **[01 - Introdução à VPC](01-introducao-vpc.md)** - Conceitos fundamentais, o que é VPC, por que usar
- **[02 - Componentes Fundacionais](02-componentes-fundacionais.md)** - Subnets, Route Tables, Internet Gateway, NAT

### Arquitetura & Design
- **[03 - Arquitetura e Design Patterns](03-arquitetura-design.md)** - Multi-tier, DMZ, padrões de conectividade
- **[04 - Segurança em VPC](04-seguranca-nlb-acl.md)** - Security Groups, NACLs, melhores práticas

### Conectividade Avançada
- **[05 - Roteamento Avançado](05-roteamento-avancado.md)** - Métricas, prioridades, roteamento condicional
- **[06 - NAT e VPC Endpoints](06-nat-endpoints.md)** - NAT Gateway/Instance, VPC Endpoints (Gateway, Interface)
- **[07 - VPN e Site-to-Site](07-vpn-connectivity.md)** - Customer Gateway, VPN Connection, Direct Connect

### Conectividade Intra-AWS
- **[08 - VPC Peering e Transit Gateway](08-vpc-peering-transit-gateway.md)** - Peering, TGW, roteamento centralizado
- **[09 - Arquiteturas Multi-Região e Híbridas](09-multi-region-hybrid.md)** - Failover, distribuição global, hub-and-spoke

### Operação & Otimização
- **[10 - Monitoramento e Troubleshooting](10-monitoramento-troubleshooting.md)** - VPC Flow Logs, ferramentas de diagnóstico
- **[11 - Alinhamento Well-Architected](11-well-architected.md)** - Segurança, confiabilidade, performance, custos
- **[12 - Infrastructure as Code com Terraform](12-iac-terraform.md)** - Implementação prática com Terraform

---

## 🏗️ Estrutura Conceitual

```
┌─────────────────────────────────────┐
│        AWS Region (us-east-1)       │
├─────────────────────────────────────┤
│                                     │
│  ┌──────────────────────────────┐   │
│  │  VPC (10.0.0.0/16)          │   │
│  │                              │   │
│  │  ┌──────────────┐            │   │
│  │  │ Public Subnet│  (GW)      │   │
│  │  │ 10.0.1.0/24 │────────→IGW│───┼─→ Internet
│  │  │              │            │   │
│  │  └──────────────┘            │   │
│  │          ↓ Route             │   │
│  │  ┌──────────────┐            │   │
│  │  │Private Subnet│            │   │
│  │  │ 10.0.2.0/24 │            │   │
│  │  │              │───NAT──→   │   │
│  │  └──────────────┘       (EIP)│   │
│  │                              │   │
│  └──────────────────────────────┘   │
│                                     │
└─────────────────────────────────────┘
```

---

## 🔑 Conceitos-Chave a Entender

### 1. **Isolamento e Segurança**
VPC é sua rede privada na AWS. Não é apenas um container - é um controle de borda completo com camadas múltiplas.

### 2. **Roteamento**
Tudo em sua VPC passa por tabelas de roteamento. Entender roteamento é entender VPC.

### 3. **Conectividade**
- Local: Comunicação dentro da VPC (subnets)
- Regional: Com outras VPCs na mesma região (Peering)
- Multi-região: Com VPCs em outras regiões
- Híbrida: Com on-premises (VPN, Direct Connect)

### 4. **Camadas de Segurança**
Múltiplas camadas garantem defesa em profundidade:
- **Network ACLs** (Subnet-level, stateless)
- **Security Groups** (Instance-level, stateful)
- **IP Routing** (Egress filtering via roteamento)

### 5. **Custo e Performance**
Sistema de preços baseado em:
- Dados transferidos entre VPCs
- NAT Gateway (por hora + dados)
- VPC Endpoints
- Data transfer out

---

## 💡 Filosofia de Design

### Design para Falha
- Nunca assuma que um componente está sempre disponível
- Distribua resources across AZs
- Implemente failover automático

### Least Privilege
- Menor subnet que funciona
- Menor Security Group que funciona
- Bloqueia tudo por padrão, abre conforme necessário

### Simplicidade Operacional
- Menos componentes = menos falhas
- Use managed services quando possível
- Automatize via IaC

### Custo-Consciente
- NAT Gateway é caro em escala
- VPC Endpoints para AWS services economizam
- Planeje banda de rede antecipadamente

---

## 🔗 Recursos Externos

- [AWS VPC Documentation](https://docs.aws.amazon.com/vpc/)
- [VPC User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/)
- [AWS Well-Architected Framework - Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/welcome.html)
- [AWS VPC Best Practices](https://aws.amazon.com/blogs/networking-and-content-delivery/)

---