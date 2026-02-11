# 12 - Infrastructure as Code com Terraform

## Estrutura de Projeto Recomendada

```
vpc-terraform/
├── main.tf                 # VPC + Gateways
├── subnets.tf             # Subnets(public/private)
├── route_tables.tf        # Route tables + routes
├── security_groups.tf     # Security groups + rules
├── nat.tf                 # NAT Gateways
├── vpc_endpoints.tf       # VPC Endpoints
├── variables.tf           # Input variables
├── outputs.tf             # Output values
├── terraform.tfvars       # Variable values
├── terraform.tfvars.example # Example values
└── .gitignore             # Exclude sensitive files
```

---

## 1. Main VPC Configuration

```hcl
# main.tf

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Environment  = var.environment
      ManagedBy    = "Terraform"
      CostCenter   = var.cost_center
      CreatedAt    = timestamp()
    }
  }
}

# Main VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "${var.environment}-vpc"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.environment}-igw"
  }
}

# VPC Flow Logs
resource "aws_flow_log" "main" {
  iam_role_arn    = aws_iam_role.flow_logs.arn
  log_destination = aws_cloudwatch_log_group.flow_logs.arn
  traffic_type    = "ALL"
  vpc_id          = aws_vpc.main.id

  tags = {
    Name = "${var.environment}-vpc-flowlogs"
  }
}

resource "aws_cloudwatch_log_group" "flow_logs" {
  name              = "/aws/vpc/flowlogs/${var.environment}"
  retention_in_days = 7

  tags = {
    Name = "${var.environment}-vpc-flowlogs"
  }
}

resource "aws_iam_role" "flow_logs" {
  name = "${var.environment}-vpc-flowlogs-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "vpc-flow-logs.amazonaws.com"
        }
      }
    ]
  })
}

resource "aws_iam_role_policy" "flow_logs" {
  name = "${var.environment}-vpc-flowlogs-policy"
  role = aws_iam_role.flow_logs.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents",
          "logs:DescribeLogGroups",
          "logs:DescribeLogStreams"
        ]
        Effect   = "Allow"
        Resource = "${aws_cloudwatch_log_group.flow_logs.arn}:*"
      }
    ]
  })
}
```

---

## 2. Subnets Configuration

```hcl
# subnets.tf

# Data source: Available AZs
data "aws_availability_zones" "available" {
  state = "available"
}

# Public Subnets (Web Tier)
resource "aws_subnet" "public" {
  count                   = var.number_of_azs
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "${var.environment}-public-subnet-${data.aws_availability_zones.available.names[count.index]}"
    Tier = "Public"
  }
}

# Private Subnets (App Tier)
resource "aws_subnet" "private_app" {
  count             = var.number_of_azs
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, 10 + count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "${var.environment}-private-app-subnet-${data.aws_availability_zones.available.names[count.index]}"
    Tier = "Application"
  }
}

# Private Subnets (Data Tier)
resource "aws_subnet" "private_data" {
  count             = var.number_of_azs
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, 20 + count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "${var.environment}-private-data-subnet-${data.aws_availability_zones.available.names[count.index]}"
    Tier = "Database"
  }
}

# RDS Subnet Group (Multi-AZ requirement)
resource "aws_db_subnet_group" "main" {
  name       = "${var.environment}-db-subnet-group"
  subnet_ids = aws_subnet.private_data[*].id

  tags = {
    Name = "${var.environment}-db-subnet-group"
  }
}
```

---

## 3. Route Tables Configuration

```hcl
# route_tables.tf

# Public Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.environment}-public-rt"
  }
}

# Default route to IGW
resource "aws_route" "public_default" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.main.id
}

# Associate public subnets
resource "aws_route_table_association" "public" {
  count          = var.number_of_azs
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Private (App) Route Table
resource "aws_route_table" "private_app" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.environment}-private-app-rt"
  }
}

# Default route to NAT
resource "aws_route" "private_app_default" {
  route_table_id     = aws_route_table.private_app.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id     = aws_nat_gateway.main[0].id
}

# Associate app subnets
resource "aws_route_table_association" "private_app" {
  count          = var.number_of_azs
  subnet_id      = aws_subnet.private_app[count.index].id
  route_table_id = aws_route_table.private_app.id
}

# Private (Data) Route Table
resource "aws_route_table" "private_data" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.environment}-private-data-rt"
  }
}

# Data tier can communicate to app, but outbound via NAT
resource "aws_route" "private_data_default" {
  route_table_id         = aws_route_table.private_data.id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.main[0].id
}

# Associate data subnets
resource "aws_route_table_association" "private_data" {
  count          = var.number_of_azs
  subnet_id      = aws_subnet.private_data[count.index].id
  route_table_id = aws_route_table.private_data.id
}
```

---

## 4. Security Groups Configuration

```hcl
# security_groups.tf

# Web Tier Security Group (ALB)
resource "aws_security_group" "alb" {
  name_prefix = "${var.environment}-alb-"
  vpc_id      = aws_vpc.main.id

  tags = {
    Name = "${var.environment}-alb-sg"
  }
}

resource "aws_vpc_security_group_ingress_rule" "alb_http" {
  security_group_id = aws_security_group.alb.id

  from_port   = 80
  to_port     = 80
  ip_protocol = "tcp"
  cidr_ipv4   = "0.0.0.0/0"

  description = "Allow HTTP from Internet"
  tags = {
    Name = "allow-http"
  }
}

resource "aws_vpc_security_group_ingress_rule" "alb_https" {
  security_group_id = aws_security_group.alb.id

  from_port   = 443
  to_port     = 443
  ip_protocol = "tcp"
  cidr_ipv4   = "0.0.0.0/0"

  description = "Allow HTTPS from Internet"
  tags = {
    Name = "allow-https"
  }
}

# Application Tier Security Group
resource "aws_security_group" "app" {
  name_prefix = "${var.environment}-app-"
  vpc_id      = aws_vpc.main.id

  tags = {
    Name = "${var.environment}-app-sg"
  }
}

resource "aws_vpc_security_group_ingress_rule" "app_from_alb" {
  security_group_id = aws_security_group.app.id

  from_port                    = 8080
  to_port                      = 8080
  ip_protocol                  = "tcp"
  referenced_security_group_id = aws_security_group.alb.id

  description = "Allow from ALB"
  tags = {
    Name = "allow-from-alb"
  }
}

# Database Security Group
resource "aws_security_group" "database" {
  name_prefix = "${var.environment}-database-"
  vpc_id      = aws_vpc.main.id

  tags = {
    Name = "${var.environment}-database-sg"
  }
}

resource "aws_vpc_security_group_ingress_rule" "db_from_app" {
  security_group_id = aws_security_group.database.id

  from_port                    = 3306
  to_port                      = 3306
  ip_protocol                  = "tcp"
  referenced_security_group_id = aws_security_group.app.id

  description = "MySQL from App tier"
  tags = {
    Name = "allow-mysql-from-app"
  }
}

# Egress (usually default: allow all out)
resource "aws_vpc_security_group_egress_rule" "all" {
  for_each = {
    alb      = aws_security_group.alb.id
    app      = aws_security_group.app.id
    database = aws_security_group.database.id
  }

  security_group_id = each.value

  from_port   = 0
  to_port     = 65535
  ip_protocol = "-1"
  cidr_ipv4   = "0.0.0.0/0"

  description = "Allow all outbound"
}
```

---

## 5. NAT Gateway Configuration

```hcl
# nat.tf

# Elastic IPs for NAT Gateway
resource "aws_eip" "nat" {
  count  = var.number_of_azs
  domain = "vpc"

  depends_on = [aws_internet_gateway.main]

  tags = {
    Name = "${var.environment}-nat-eip-${data.aws_availability_zones.available.names[count.index]}"
  }
}

# NAT Gateways (one per AZ for HA)
resource "aws_nat_gateway" "main" {
  count         = var.number_of_azs
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  depends_on = [aws_internet_gateway.main]

  tags = {
    Name = "${var.environment}-nat-gw-${data.aws_availability_zones.available.names[count.index]}"
  }
}
```

---

## 6. VPC Endpoints Configuration

```hcl
# vpc_endpoints.tf

# S3 Gateway Endpoint
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.${var.aws_region}.s3"
  route_table_ids = [
    aws_route_table.public.id,
    aws_route_table.private_app.id,
    aws_route_table.private_data.id
  ]

  tags = {
    Name = "${var.environment}-s3-endpoint"
  }
}

# S3 Endpoint Policy (restrict to specific bucket)
resource "aws_vpc_endpoint_policy" "s3" {
  vpc_endpoint_id = aws_vpc_endpoint.s3.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = "*"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:ListBucket"
        ]
        Resource = [
          "arn:aws:s3:::${var.app_bucket}",
          "arn:aws:s3:::${var.app_bucket}/*"
        ]
      },
      {
        Effect = "Deny"
        Principal = "*"
        Action = "s3:*"
        Resource = "*"
        Condition = {
          StringNotEquals = {
            "aws:SourceVpc" = aws_vpc.main.id
          }
        }
      }
    ]
  })
}

# Secrets Manager Interface Endpoint
resource "aws_vpc_endpoint" "secretsmanager" {
  vpc_id              = aws_vpc.main.id
  service_name        = "com.amazonaws.${var.aws_region}.secretsmanager"
  vpc_endpoint_type   = "Interface"
  subnet_ids          = aws_subnet.private_app[*].id
  security_group_ids  = [aws_security_group.vpc_endpoint.id]
  private_dns_enabled = true

  tags = {
    Name = "${var.environment}-secretsmanager-endpoint"
  }
}

# Security Group for Interface Endpoints
resource "aws_security_group" "vpc_endpoint" {
  name_prefix = "${var.environment}-vpce-"
  vpc_id      = aws_vpc.main.id

  tags = {
    Name = "${var.environment}-vpce-sg"
  }
}

resource "aws_vpc_security_group_ingress_rule" "vpc_endpoint" {
  security_group_id = aws_security_group.vpc_endpoint.id

  from_port                    = 443
  to_port                      = 443
  ip_protocol                  = "tcp"
  referenced_security_group_id = aws_security_group.app.id

  description = "Allow from App tier"
}
```

---

## 7. Variables Configuration

```hcl
# variables.tf

variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be dev, staging, or prod."
  }
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "number_of_azs" {
  description = "Number of availability zones to use"
  type        = number
  default     = 2
  validation {
    condition     = var.number_of_azs > 0 && var.number_of_azs <= 3
    error_message = "Number of AZs must be 1-3."
  }
}

variable "cost_center" {
  description = "Cost center for billing tags"
  type        = string
}

variable "app_bucket" {
  description = "S3 bucket name for app data"
  type        = string
}
```

---

## 8. Outputs Configuration

```hcl
# outputs.tf

output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "VPC CIDR block"
  value       = aws_vpc.main.cidr_block
}

output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_app_subnet_ids" {
  description = "Private app tier subnet IDs"
  value       = aws_subnet.private_app[*].id
}

output "private_data_subnet_ids" {
  description = "Private data tier subnet IDs"
  value       = aws_subnet.private_data[*].id
}

output "nat_gateway_eips" {
  description = "NAT Gateway Elastic IPs"
  value       = aws_eip.nat[*].public_ip
}

output "security_group_ids" {
  description = "Security group IDs"
  value = {
    alb      = aws_security_group.alb.id
    app      = aws_security_group.app.id
    database = aws_security_group.database.id
  }
}

output "s3_endpoint_id" {
  description = "S3 VPC Endpoint ID"
  value       = aws_vpc_endpoint.s3.id
}

output "rds_subnet_group_name" {
  description = "RDS DB Subnet Group name"
  value       = aws_db_subnet_group.main.name
}
```

---

## 9. Terraform Values File

```hcl
# terraform.tfvars

aws_region   = "us-east-1"
environment  = "prod"
vpc_cidr     = "10.0.0.0/16"
number_of_azs = 3
cost_center  = "engineering"
app_bucket   = "prod-app-data"
```

---

## 10. Usage & Deployment

```bash
# Initialize Terraform (download providers)
terraform init

# Validate configuration syntax
terraform validate

# Format configuration (auto-fix styling)
terraform fmt -recursive

# Plan deployment (preview changes)
terraform plan -out=tfplan

# Review plan output
cat tfplan | grep '# aws_' | head -20

# Apply changes
terraform apply tfplan

# Get outputs
terraform output

# Query specific output
terraform output vpc_id

# Destroy (careful in prod!)
terraform destroy -auto-approve
```

---

## 11. Best Practices

### ✅ SIM
- Modular structure (separate files by component)
- Input variables for reusability
- Output all important IDs
- Comments explain "why" not "what"
- Terraform validation na build
- Separate dev/staging/prod tfvars
- Version control push all files except *.tfstate

### ❌ NÃO
- Everything in main.tf (hard to manage)
- Hardcoded values (use variables)
- Secrets in tfvars (use AWS Secrets Manager)
- Push .tfstate to git (store remotely with TF Cloud)
- assume_role in code (use AWS credentials provider)
- Manual changes after terraform apply (defeats purpose)

---

## 12. Managing State Remotely

```hcl
# Recommended: Terraform Cloud or S3 + DynamoDB

# S3 + DynamoDB backend
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "vpc/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}

# Terraform Cloud
terraform {
  cloud {
    organization = "my-org"
    workspaces {
      name = "vpc-${var.environment}"
    }
  }
}
```

---

## Próximos Passos

Seu ambiente VPC está pronto! Próximas etapas:
1. Deploy EC2 instances (criar arquivo: ec2.tf)
2. Setup RDS databases (criar arquivo: rds.tf)
3. Configure ALB (criar arquivo: alb.tf)
4. Setup monitoring/alarms (criar arquivo: monitoring.tf)
5. Implement CI/CD (GitHub Actions + Terraform)

---

**Anterior**: [11 - Well-Architected](11-well-architected.md)  
**Final do Módulo**: Parabéns por completar o estudo de VPC!

---

## Referências

- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest)
- [Terraform Best Practices](https://www.terraform.io/docs/language)
- [AWS VPC Terraform Guide](https://docs.aws.amazon.com/vpc/latest/userguide/)
