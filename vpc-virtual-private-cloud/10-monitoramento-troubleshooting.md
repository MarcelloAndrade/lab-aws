# 10 - Monitoramento e Troubleshooting de VPC

## VPC Flow Logs em Detalhe

### Enablement

```
AWS Console → VPC → Flow Logs → Create

┌────────────────────────────────────┐
│ Flow Logs Configuration            │
├────────────────────────────────────┤
│ Name: vpc-flowlogs                 │
│ Filter:                            │
│   ├─ ACCEPT (apenas sucesso)       │
│   ├─ REJECT (apenas erros)         │
│   └─ ALL (todos)                   │
│ Destination:                       │
│   ├─ CloudWatch Logs               │
│   ├─ S3                            │
│   └─ Kinesis Data Firehose         │
│ Log format: (default ok)           │
│ Traffic type: Both                 │
│ TTL: 7 days                        │
│ IAM Role: (auto-created)           │
└────────────────────────────────────┘
```

### Formato de Flow Log

```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action tcpflags log-status

2 123456789012 eni-abc12345 10.0.1.100 8.8.8.8 54321 53 17 5 256 1645251300 1645251361 ACCEPT - OK
└─ version: 2
└─ account-id: sua AWS account
└─ interface-id: eni-abc12345 (EC2 ENI)
└─ srcaddr: 10.0.1.100 (EC2 IP)
└─ dstaddr: 8.8.8.8 (DNS)
└─ srcport: 54321 (ephemeral)
└─ dstport: 53 (DNS)
└─ protocol: 17 (UDP)
└─ packets: 5
└─ bytes: 256 bytes total
└─ start: timestamp
└─ end: timestamp
└─ action: ACCEPT (ou REJECT)
└─ tcpflags: TCP flags (if TCP)
└─ log-status: OK (ou error)
```

### Análise com CloudWatch Insights

```
Usar CloudWatch Logs Insights para queries:

1. Top 10 rejected connections:
   fields srcaddr, dstaddr, dstport, action
   | filter action = "REJECT"
   | stats count() as rejections by srcaddr, dstport
   | sort rejections desc
   | limit 10

2. Tráfego por endereço de destino:
   fields bytes, dstaddr, dstport
   | filter action = "ACCEPT"
   | stats sum(bytes) as total_bytes by dstaddr, dstport
   | sort total_bytes desc

3. Tentativas de SSH rejeitadas:
   fields srcaddr, dstaddr, dstport, action
   | filter dstport = 22 and action = "REJECT"
   | stats count() as attempts by srcaddr
   | sort attempts desc

4. Comunicação entre subnets:
   fields srcaddr, dstaddr
   | filter srcaddr like /10.0.11/ and dstaddr like /10.0.21/
   | stats count() as flows by srcaddr, dstaddr
```

---

## VPC Reachability Analyzer

### Uso Prático

```
Ferramenta: Diagnosticar conectividade entre recursos

AWS Console → VPC → Reachability Analyzer → Create

┌──────────────────────────────────┐
│ Create Reachability Analysis     │
├──────────────────────────────────┤
│ Name: ec2-to-rds-check           │
│ Source: EC2 instance (app)       │
│ Destination: RDS endpoint        │
│ Destination port: 3306           │
│ Protocol: TCP                    │
│ Analyze                          │
└──────────────────────────────────┘

Resultado:
✓ Reachable
  Path explanation:
  ├─ Source EC2 → Route table
  ├─ Matched: 10.0.21.0/24 → local
  ├─ NACL: allow TCP 3306
  ├─ Security Group: allow 3306 from app-sg
  └─ Destination RDS available

✗ Not Reachable
  Breakdown:
  ├─ Route table: 10.0.21.0/24 → local ✓
  ├─ NACL inbound: TCP 3306 ✗ BLOCKED
  │  └─ Suggestion: Add rule 110 TCP 3306
  ├─ Security Group: allow 3306 ✓
  └─ Destination: available ✓
  
  → Need to update NACL inbound!
```

---

## CloudWatch Monitoring VPC

### Custom Alarms

```
Alarm 1: REJECT rate high

Metric:
┌─ Namespace: AWS/VPC
└─ Metric: FlowLogsRejectCount
   (need custom metric from Flow Logs)

Threshold:
├─ 1000 RETENTIONJs per 5 minutes
└─ State: ALARM if exceeded for 2 evaluations

Action:
├─ SNS notification
└─ Auto-scaling or manual investigation

Query for custom metric:
fields action
| filter action = "REJECT"
| stats count() as rejects
| publish "VPC/RejectCount" as 'rejects'
```

### Key Metrics to Monitor

```
1. Flow Count
   └─ High sudden spike = DDoS or leak?

2. Reject Rate
   └─ Should be low for normal ops
   └─ High = firewall blocking legitimate traffic

3. Top Talkers
   └─ Identify bandwidth-heavy communicators
   └─ Detect data exfiltration attempts

4. Protocol Distribution
   └─ Normal: HTTP/HTTPS dominant
   └─ Suspicious: FTP, telnet, unencrypted protocols

5. Port Scanning Detection
   └─ Single source → many ports
   └─ Query: Group by srcaddr, count distinct dstport
```

---

## Ferramentas de Debugging

### 1. netstat / ss (em EC2)

```bash
# Ver conexões ativas no EC2
ss -tunap | grep ESTABLISHED

# Filter para IP específico
ss -tunap | grep 10.0.21.5

# Ver listening ports
ss -tunlp

# Resolve hostname
nslookup db.internal.company.com

# If resolution fails → route53 private hosted zone issue
```

### 2. tcpdump (packet capture)

```bash
# Capture tráfego para debug
sudo tcpdump -i any -n port 3306

# Capture com context (não apenas pacote)
sudo tcpdump -i any -n -A port 443 | head -100

# Filtrar por protocolo
sudo tcpdump -i any -n 'tcp and port 22'

# Salvar para arquivo (por analysis depois)
sudo tcpdump -i any -w /tmp/dump.pcap port 3306
scp ec2:/tmp/dump.pcap .
# open em wireshark no seu laptop
```

### 3. mtr (multi-traceroute)

```
# Continuous diag de latência
mtr -c 10 10.0.21.5

# Output:
My traceroute  [v0.93]
                           Packets               Pings
Host                    Loss%   Snt   Last   Avg  Best  Wrst StDev
1. 169.254.169.254      0.0%    10    1.2   0.8  0.3  1.2  0.2
2. 10.0.21.100          0.0%    10    3.4   2.8  2.1  4.5  0.8
3. 10.0.21.5            0.0%    10    4.1   3.5  2.8  5.2  0.9

✓ All hops responding = network ok
✗ Hop timing out = TTL exhausted / firewall
```

### 4. iperf3 (bandwidth testing)

```
Server EC2 (10.0.21.5):
$ iperf3 -s

Client EC2 (10.0.11.10):
$ iperf3 -c 10.0.21.5 -t 60

Output:
Connecting to host 10.0.21.5, port 5201
[ ID] Interval           Transfer     Bitrate
[  4]   0.00-60.00  sec  70.0 GBytes  9.99 Gbits/sec

✓ Close to expected (9.5 Gbps typical for modern)
✗ Low bandwidth = network bottleneck / MTU issue
```

---

## AWS Systems Manager Session Manager

### Alternative para SSH

```
Benefits:
✅ Não precisa de Security Group rule de SSH
✅ Não precisa de SSH key (usa IAM)
✅ Audit de sessions (CloudTrail)
✅ Funciona em private subnets (sem bastion)

Requirements:
1. EC2 Instance EC2 tenha IAM role
   └─ Policy: AmazonSSMManagedInstanceCore
2. VPC Endpoint para Systems Manager
   └─ Interface endpoint com security group

Setup:
AWS Systems Manager → Session Manager → Start session
├─ Choose instance
└─ Terminal abre no browser (no SSH client needed)
```

---

## Interface Endpoint Troubleshooting

### Health Check para Interface Endpoint

```
Endpoint pode estar:
✓ Available: Normal operation
✗ Unavailable: Problem

Checklist:
1. Endpoint status:
   AWS Console → VPC → Endpoints → search endpoint
   └─ State: available?

2. ENI associated with endpoint:
   Check all ENIs:
   └─ Every ENI should be in UP state
   └─ If one DOWN = problem

3. Security Group on endpoint:
   └─ Current SG allowing inbound from your app?
   └─ Port 443 open?

4. Route 53 Private DNS:
   └─ Resolving to endpoint IPs?
   nslookup secretsmanager.us-east-1.amazonaws.com
   └─ Should return endpoint IP (10.0.x.x)
   └─ If returns public IP = Private DNS disabled
```

---

## NAT Gateway Monitoring

### Metrics

```
CloudWatch Metrics (automatically collected):

1. BytesOutToDestination
   └─ Total bytes NAT translated outbound
   └─ High = lots of internet access

2. PacketsOutToDestination
   └─ Packet count translated
   └─ Can detect DDoS (spike in packets)

3. BytesInFromDestination
   └─ Inbound traffic (responses)

4. ConnectionCount
   └─ Total active connections through NAT
   └─ Can hit limits (default: 55K per second)

5. ErrorPortAllocation
   └─ Ephemeral port exhaustion
   └─ Happens if: Too many connections from single EC2
   └─ Recommendation: Scale out (more EC2s), or use ALB connection pooling
```

### Optimization

```
If NAT hits connection limits:

Option 1: Add another NAT Gateway
├─ Route table: 0.0.0.0/0 → NAT-1
├─ Route table (different subnet): 0.0.0.0/0 → NAT-2
└─ Distribute traffic across NATs

Option 2: Connection pooling
├─ Application-level (SDK configures)
├─ Reuse TCP connections instead of creating new
└─ Huge impact on connection count

Option 3: Use VPC Endpoint instead
├─ For AWS services (S3, DynamoDB)
└─ No NAT needed, no connection limits
```

---

## Security Group Audit

### Find Overly Permissive Rules

```bash
# Query all security groups with risky rules
aws ec2 describe-security-groups \
  --filters "Name=ip-permission.cidr,Values=0.0.0.0/0" \
  --query 'SecurityGroups[*].[GroupId,GroupName,IpPermissions]'

# Find SSH open to world (0.0.0.0/0)
aws ec2 describe-security-groups \
  --filters \
    "Name=ip-permission.fromPort,Values=22" \
    "Name=ip-permission.toPort,Values=22" \
    "Name=ip-permission.cidr,Values=0.0.0.0/0"
```

### Regular SG Review

```
Process:
1. Quarterly SG audit
   ├─ List all SGs
   ├─ List all rules
   └─ Document ownership

2. For each rule:
   ├─ Is it still used?
   │  └─ Check CloudTrail for use
   ├─ Is it least privilege?
   │  └─ Can narrow CIDR?
   │  └─ Can narrow port range?
   └─ Is it documented?
      └─ Why does this rule exist?

3. Deprecate unused:
   ├─ Warn team before removal
   ├─ Check attached instances
   └─ Remove if confirmed unused
```

---

## Diagnosing Latency

### Is it Network Latency?

```
1. app logs: "DB query slow"

2. Check RDS CloudWatch:
   ├─ Network throughput normal? ✓
   ├─ CPU usage normal?           ✓
   └─ Disk I/O normal?            ✓

3. Measure network latency:
   From app server:
   mtr -c 10 database.internal
   └─ Average latency: 0.8ms ✓ (normal)

Conclusion: Probably DB query, not network

If latency is 50ms+:
1. VPC too many hops? (unlikely, usually < 1ms)
2. NAT gateway in different AZ?
   └─ Route table pointing to NAT in wrong AZ?
   └─ Fix: ensure local AZ NAT preference
3. Multi-region latency?
   └─ Expected for cross-region
   └─ Optimize with caching / CDN
```

---

## Best Practices de Monitoramento

### ✅ SIM
- Enable VPC Flow Logs (all VPCs)
- CloudWatch alarms em rejects
- Regular Reachability Analyzer tests
- Session Manager for secure access
- Monitor NAT Gateway metrics
- Quarterly security group audit
- Document network architecture (Lucidchart/Visio)

### ❌ NÃO
- Assumir "network is fast" sem measurement
- Esquecer Flow Logs por cost savings (barato vs incidents)
- Ignorar CloudWatch metrics (gold mine de insights)
- Deploy não-testado em produção
- Sem alertas (silent failures)

---

## Próximos Passos

- **[11 - Well-Architected](11-well-architected.md)**: Complete operational excellence
- **[12 - IaC com Terraform](12-iac-terraform.md)**: Automatizare setup

---

**Anterior**: [09 - Multi-Região](09-multi-region-hybrid.md)  
**Próximo**: [11 - Alinhamento Well-Architected](11-well-architected.md)
