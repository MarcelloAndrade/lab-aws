# 11 - Alinhamento AWS Well-Architected Framework

## Overview: 5 Pillars

```
┌────────────────────────────────────────────────┐
│         Well-Architected Framework             │
├────────────────────────────────────────────────┤
│                                                │
│ 1. Operational Excellence → Monitoring, Logs   │
│ 2. Security                 → Least privilege  │
│ 3. Reliability              → Multi-AZ, backup │
│ 4. Performance Efficiency   → Right-sized, opt │
│ 5. Cost Optimization        → Eliminate waste  │
│                                                │
└────────────────────────────────────────────────┘
```

---

## 1. Operational Excellence (VPC Perspective)

### Design para Observabilidade

```
✅ VPC Flow Logs habilitados
   └─ Capture all traffic
   └─ Armazene em CloudWatch Logs ou S3

✅ CloudWatch alarmes
   └─ Monitor rejects
   └─ Monitor NAT errors
   └─ Monitor connection limits

✅ Documentação
   └─ Network diagram (Visio/Lucidchart)
   └─ CIDR allocation spreads
   └─ Security group purposes
   └─ Route table justifications

✅ IaC (Infrastructure as Code)
   └─ Terraform para reproducibilidade
   └─ Versão controlado
   └─ Code review antes deploy

✅ Change management
   └─ Network changes via Terraform plan
   └─ Approved by team
   └─ Deployed consistent
```

### Runbooks para Operações

```
Runbook: "VPC Connectivity Lost"

1. Diagnóstico
   - Check VPC Flow Logs (rejections?)
   - Run Reachability Analyzer
   - Check Route Tables
   - Check Security Groups
   - Check NACLs

2. Check list rápido
   - Route existe?     nao-exit
   - NACL permite?     nao-exit
   - SG permite?       nao-exit
   - Instance up?      nao-exit
   - Instância status? instance-not-running
   
3. Remediação comum
   - Missing route: aws ec2 create-route ...
   - SG too restrictive: aws ec2 authorize-* ...
   - NACL blocked: similar command

4. Escalation
   - se ainda falhar → Check AWS Health Dashboard
   - se ainda falhar → AWS Support ticket
```

---

## 2. Security (VPC Perspective)

### Least Privilege: Defense in Depth

```
Aplicação
    ↓ (Least trust)
Security Group
    ├─ Only ports needed
    ├─ Only sources allowed
    └─ Example: TCP 8080 from ALB SG only
    
    ↓ (Additional check)
    
Network ACL
    ├─ Ingress: allow from ALBnet
    ├─ Egress: allow to database
    └─ Ephemeral ports for responses
    
    ↓ (Final default)
    
VPC Routing
    └─ Can discard packets if no route match

Resultado:
- Defense in depth
- Single rule misconfiguration ≠ breach
- Auditável em cada camada
```

### Encryption em Transit

```
Intra-VPC: Já criptografado por default
├─ AWS usa IEEE 802.1AE (MACsec)
├─ Hypervisor-nível encryption
└─ Você não controla (AWS manage)

Internet-bound:
├─ Use TLS/HTTPS (443)
└─ NEVER insecured HTTP for sensitive data

VPN (on-premises):
├─ IPsec (built-in)
├─ Encryption: AES-256
└─ Autenticação: HMAC-SHA256 default

VPC Endpoints (AWS services):
├─ TLS/mTLS (automatic)
├─ PrivateLink (encrypted channel)
└─ No internet exposure


✅ Implement:
  - ALB/NLB termina TLS
  - Backend uses internal certificates
  - VPC Endpoint para AWS services (não via Internet)
```

### Identity & Access Control

```
Who can build/modify VPC?

IAM Policy Example:

{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VPCManagement",
      "Effect": "Allow",
      "Action": [
        "ec2:CreateVpc",
        "ec2:CreateSubnet",
        "ec2:CreateSecurityGroup",
        "ec2:AuthorizeSecurityGroup*",
        "ec2:CreateRoute",
        "ec2:*Gateway"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "aws:RequestedRegion": "us-east-1"
        },
        "StringLike": {
          "aws:username": "network-admins"
        }
      }
    }
  ]
}

Benefits:
- Only network-admins can modify VPC
- Only in us-east-1
- Developers can view but not modify
- Audit via CloudTrail
```

---

## 3. Reliability (VPC Perspective)

### Multi-AZ Design

```
Bad (Single AZ):
AZ us-east-1a:
└─ All resources in one AZ
└─ One AZ fails = complete downtime

Good (Multi-AZ):
AZ us-east-1a:
├─ Public Subnet: 10.0.1.0/24
├─ Private Subnet: 10.0.11.0/24
└─ RDS Primary

AZ us-east-1b:
├─ Public Subnet: 10.0.2.0/24
├─ Private Subnet: 10.0.12.0/24
└─ RDS Standby

AZ us-east-1c:
├─ Public Subnet: 10.0.3.0/24
├─ Private Subnet: 10.0.13.0/24
└─ (optional, provides 3x redundancy)

Impact:
- One AZ fails: 66% capacity remains
- Services auto-failover (ALB, ASG)
- RDS automatic failover
- Requests rerouted automatically
```

### Redundancy at Every Layer

```
┌─────────────────────────────┐
│Compute (ASG across AZs)     │
├─ Min 3 instances (1 per AZ) │
├─ ASG maintains desired       │
└─ Auto-replace failures       │
└─────────────────────────────┘

┌─────────────────────────────┐
│Network (Multiple IGWs/NATs)  │
├─ 1 IGW per VPC (shared)     │
├─ NAT Gateway per AZ         │
├─ If NAT fails, others work  │
└─ Route tables point to local│
└─────────────────────────────┘

┌─────────────────────────────┐
│Database (Multi-AZ)          │
├─ RDS Standby in different   │
├─ Automatic failover         │
├─ Synchronous replication    │
└─ RPO = 0 (no data loss)     │
└─────────────────────────────┘

┌─────────────────────────────┐
│Load Balancing (ALB)         │
├─ Targets across AZs         │
├─ Health check failures      │
├─ Auto-deregister            │
└─ Traffic balanced           │
└─────────────────────────────┘
```

### Health Checks & Failover

```
Define Health Check:
┌─ Protocol: HTTP/HTTPS/TCP
├─ Path: /health
├─ Port: 8080
├─ Interval: 30 seconds
├─ Timeout: 5 seconds
├─ Healthy threshold: 2
├─ Unhealthy threshold: 3
└─ (3 failed checks = marked unhealthy)

Response:
Instance responds: HTTP 200 → healthy
Instance responds: HTTP 500 → counts as failure
Instance timeout → counts as failure
Instance down → immediate deregistration

Benefit:
- Automatic removal from rotation
- No manual intervention
- Recovers when healthy again
```

---

## 4. Performance Efficiency (VPC Perspective)

### Right-Sizing Network

```
Don't Overdo VPC:
├─ CIDR: /16 is safe default
├─ Don't do: /28 (too small) or /8 (wasteful)
└─ Plan for 2-3x growth

Don't Underestimate Subnets:
├─ AWS reserves 5 IPs per subnet
├─ /24 subnet: 256 IPs, but only 251 usable
├─ Scale subnets: multiple /24s if needed
└─ Cross-AZ: distribute load

Network Performance Optimization:
├─ Co-location: put talky components same AZ
├─ Cluster placement group: same rack (< 1ms)
├─ Enhanced networking: ENA (high throughput)
└─ Jumbo frames: MTU 9000 (more data per packet)
```

### NAT Gateway vs VPC Endpoints

```
Before optimization:
┌─ EC2 reads 1TB S3 per day
├─ Route: 0.0.0.0/0 → NAT
├─ NAT costs: $230/month
├─ Data transfer: ~$45/month
└─ Total: ~$275/month

After optimization (VPC Endpoint):
┌─ EC2 reads 1TB S3 per day
├─ Route: s3-cidr → vpce-s3 (gateway endpoint)
├─ Costs: $0 (AWS managed)
├─ Data transfer: $0 (internal)
└─ Total: $0/month

↳ Savings: $275/month or $3,300/year
↳ Plus: 10x faster (no internet round-trip)
```

---

## 5. Cost Optimization (VPC Perspective)

### Cost Drivers in VPC

```
1. NAT Gateway: $0.32/hour (~$230/month)
   Optimization:
   - Use VPC Endpoint instead (free to $7/mo)
   - Share NAT across app tiers
   - Use VPC Endpoint for AWS services

2. Data Transfer:
   - EC2 to Internet: $0.09/GB (expensive!)
   - EC2 to same AZ: $0 (free)
   - EC2 to different AZ: $0.01/GB
   - EC2 to different region: $0.02/GB
   
   Optimization:
   - Keep active data same AZ
   - Use VPC Endpoints (avoid internet)
   - CloudFront for repeated requests

3. VPC Peering (Cross-region): ~$0.02/GB
   Optimization:
   - Minimize cross-region data
   - Cache to avoid repeated transfers
   - Use DynamoDB Global Tables (cheaper for distributed)

4. Transit Gateway: $0.05/hour (~$36/month per hour) + data
   Optimization:
   - Only use if 5+ VPCs
   - <5 VPCs: use peering (cheaper)
```

### Cost Allocation Tags

```
Tag every resource for cost tracking:

VPC:
  tags = {
    Name = "prod-vpc"
    Environment = "prod"
    CostCenter = "engineering"
    Project = "core-platform"
    ManagedBy = "terraform"
  }

EC2:
  tags = {
    Name = "api-server-1"
    Environment = "prod"
    CostCenter = "engineering"
    Application = "api"
  }

CloudWatch bill:
  - Filter by CostCenter tag
  - See: engineering = $15K/month
  - allocation: 60% compute, 30% data, 10% network

Benefit:
  - Chargeback to teams
  - Identify wasteful resources
  - Cost optimization opportunities
```

---

## VPC Architecture Scorecard

### Complete VPC Review

```
Operational Excellence:
- [ ] VPC Flow Logs enabled
- [ ] CloudWatch alarms configured
- [ ] Network diagram documented
- [ ] Infrastructure as Code (Terraform)
- [ ] Runbooks for common issues
- [ ] Quarterly network audit

Security:
- [ ] No 0.0.0.0/0 for SSH/RDP (except bastion)
- [ ] VPC Endpoints for AWS services
- [ ] Encryption in transit (TLS)
- [ ] NACL + SG + Routing (defense in depth)
- [ ] No hardcoded credentials in code
- [ ] CloudTrail logging for network changes

Reliability:
- [ ] Multi-AZ design (≥2 AZs)
- [ ] NAT redundancy (1 per AZ)
- [ ] RDS Multi-AZ enabled
- [ ] Application Load Balancer health checks
- [ ] Failover tested (chaos engineering)
- [ ] Backup strategy (RDS snapshots)

Performance:
- [ ] VPC Endpoint for S3/DynamoDB
- [ ] Right-sized CIDR allocation
- [ ] Co-located database/app if needed
- [ ] Enhanced networking enabled (EC2)
- [ ] No unnecessary internet egress

Cost Optimization:
- [ ] VPC Endpoint instead of NAT (when possible)
- [ ] Tags on all resources
- [ ] Quarterly cost analysis
- [ ] Dead resources removed
- [ ] Transit Gateway only if 5+ VPCs
- [ ] Multi-region data replication justified

Score: __/21 Recommendations
```

---

## Próximos Passos

- **[12 - IaC com Terraform](12-iac-terraform.md)**: Implement with code

---

**Anterior**: [10 - Monitoramento](10-monitoramento-troubleshooting.md)  
**Próximo**: [12 - Infrastructure as Code com Terraform](12-iac-terraform.md)
