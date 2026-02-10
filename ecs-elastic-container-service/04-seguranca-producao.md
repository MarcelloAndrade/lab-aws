# 🔐 Segurança em Produção

Segurança em ECS é uma cadeia: um elo fraco quebra tudo.

## 1. IAM Roles - Zero Credentials

```hcl
# ============================================
# EXECUTION ROLE (para ECS Agent, não container)
# ============================================

resource "aws_iam_role" "ecs_task_execution_role" {
  name = "ecsTaskExecutionRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
    }]
  })
}

# AWS managed policy (ECR pull, CloudWatch push)
resource "aws_iam_role_policy_attachment" "ecs_task_execution_policy" {
  role       = aws_iam_role.ecs_task_execution_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}

# Permissão adicional: ler secrets
resource "aws_iam_role_policy" "ecs_execution_secrets" {
  name = "ecsExecutionSecretsPolicy"
  role = aws_iam_role.ecs_task_execution_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "secretsmanager:GetSecretValue",
          "kms:Decrypt"
        ]
        Resource = [
          aws_secretsmanager_secret.db_password.arn,
          aws_secretsmanager_secret.api_key.arn,
          aws_kms_key.secrets.arn
        ]
      }
    ]
  })
}

# ============================================
# TASK ROLE (para container app)
# ============================================

resource "aws_iam_role" "ecs_task_role" {
  name = "ecsTaskRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Action = "sts:AssumeRole"
      Effect = "Allow"
      Principal = {
        Service = "ecs-tasks.amazonaws.com"
      }
    }]
  })
}

# Permissões específicas do app (LEAST PRIVILEGE)
resource "aws_iam_role_policy" "ecs_task_policy" {
  name = "ecsTaskPolicy"
  role = aws_iam_role.ecs_task_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "S3Access"
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:PutObject",
          "s3:DeleteObject"
        ]
        Resource = "${aws_s3_bucket.app_data.arn}/uploads/*"
      },
      {
        Sid    = "DynamoDBAccess"
        Effect = "Allow"
        Action = [
          "dynamodb:Query",
          "dynamodb:GetItem",
          "dynamodb:PutItem"
        ]
        Resource = aws_dynamodb_table.app_table.arn
      },
      {
        Sid    = "RDSAccess"
        Effect = "Allow"
        Action = [
          "rds:DescribeDatabases"
        ]
        Resource = "*"
      },
      {
        Sid    = "CloudWatchLogs"
        Effect = "Allow"
        Action = [
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "${aws_cloudwatch_log_group.ecs_app.arn}:*"
      },
      {
        Sid    = "XRayWrite"
        Effect = "Allow"
        Action = [
          "xray:PutTraceSegments",
          "xray:PutTelemetryRecords"
        ]
        Resource = "*"
      }
    ]
  })
}

# Contas do IAM NÃO devem ser usadas em containers
# Se precisar de credenciais de terceiros, use Secrets Manager
```

## 2. Secrets Management

```hcl
resource "aws_secretsmanager_secret" "db_password" {
  name                    = "prod/rds/password"
  recovery_window_in_days = 7  # Grace period antes de delete

  tags = {
    Application = "minha-aplicacao"
    Rotation    = "manual"  # ou automática com Lambda
  }
}

resource "aws_secretsmanager_secret_version" "db_password" {
  secret_id = aws_secretsmanager_secret.db_password.id
  
  # NÃO hardcode! Usar variável do Terraform
  secret_string = jsonencode({
    username = "postgres"
    password = var.db_password  # De tfvars ou TFC
    host     = aws_db_instance.postgres.address
    port     = 5432
    dbname   = "prodapp"
  })
}

# Encryption com CMK customizada
resource "aws_kms_key" "secrets" {
  description             = "KMS key for ECS secrets"
  deletion_window_in_days = 10
  enable_key_rotation     = true

  tags = {
    Application = "minha-aplicacao"
  }
}

resource "aws_kms_alias" "secrets" {
  name          = "alias/ecs-secrets"
  target_key_id = aws_kms_key.secrets.key_id
}
```

## 3. Network Isolation

```hcl
# VPC com subnets privadas
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "app-vpc"
  }
}

# Subnets PRIVADAS para tasks
resource "aws_subnet" "private_1" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.1.0/24"
  availability_zone = data.aws_availability_zones.available.names[0]

  tags = {
    Name = "private-1"
  }
}

resource "aws_subnet" "private_2" {
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.2.0/24"
  availability_zone = data.aws_availability_zones.available.names[1]

  tags = {
    Name = "private-2"
  }
}

# NAT Gateway (para tasks acessar internet)
resource "aws_eip" "nat" {
  domain = "vpc"
  depends_on = [aws_internet_gateway.main]

  tags = {
    Name = "nat-eip"
  }
}

resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public_1.id

  tags = {
    Name = "nat-gw"
  }

  depends_on = [aws_internet_gateway.main]
}

# Route tables para privadas (via NAT)
resource "aws_route_table" "private" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main.id
  }

  tags = {
    Name = "private-rt"
  }
}

resource "aws_route_table_association" "private_1" {
  subnet_id      = aws_subnet.private_1.id
  route_table_id = aws_route_table.private.id
}

resource "aws_route_table_association" "private_2" {
  subnet_id      = aws_subnet.private_2.id
  route_table_id = aws_route_table.private.id
}

# Security Group: Inbound restraints
resource "aws_security_group" "ecs_tasks" {
  name        = "ecs-tasks-sg"
  description = "Restrict inbound to ALB only"
  vpc_id      = aws_vpc.main.id

  # ✅ ONLY from ALB
  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  # ✅ Outbound para dependências
  egress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.rds.id]
    description     = "To RDS"
  }

  egress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
    description = "To external APIs"
  }

  tags = {
    Name = "ecs-tasks-sg"
  }
}
```

## Checklist de Segurança

- [ ] Task Definition com versão específica de imagem (não `latest`)
- [ ] Secrets Management via Secrets Manager (não environment variables)
- [ ] Execution Role com permissões mínimas (ECR, CloudWatch)
- [ ] Task Role com least privilege (apenas recursos necessários)
- [ ] VPC privado para tasks (subnets privadas)
- [ ] NAT Gateway para acesso à internet
- [ ] Security Groups restritivos (inbound apenas ALB, egress target)
- [ ] CloudWatch Logs com retention apropriado
- [ ] Health checks configurados
- [ ] KMS encryption para secrets
- [ ] Network Policies (se usar service mesh)
- [ ] Container image scanning no ECR
- [ ] Audit logging no CloudTrail
