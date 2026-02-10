# 🚀 CI/CD com ECS

## GitHub Actions + OIDC (Zero Credentials)

```yaml
name: Deploy to ECS

on:
  push:
    branches:
      - main
    paths:
      - 'src/**'
      - 'Dockerfile'
      - '.github/workflows/deploy-ecs.yml'

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: minha-aplicacao
  ECS_SERVICE: minha-aplicacao-service
  ECS_CLUSTER: producao-cluster
  ECS_TASK_DEFINITION: minha-aplicacao

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write  # OIDC token

    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      # OIDC Federation (zero aws credentials!)
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          role-to-assume: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/GitHubActionsRole
          aws-region: ${{ env.AWS_REGION }}

      # Login ECR
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v1

      # Build & Test
      - name: Build Docker image
        id: image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/${{ env.ECR_REPOSITORY }}:$IMAGE_TAG .
          docker tag $ECR_REGISTRY/${{ env.ECR_REPOSITORY }}:$IMAGE_TAG $ECR_REGISTRY/${{ env.ECR_REPOSITORY }}:latest
          echo "image=$ECR_REGISTRY/${{ env.ECR_REPOSITORY }}:$IMAGE_TAG" >> $GITHUB_OUTPUT

      # Push to ECR
      - name: Push image to ECR
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
        run: |
          docker push ${{ steps.image.outputs.image }}
          docker push $ECR_REGISTRY/${{ env.ECR_REPOSITORY }}:latest

      # Update Task Definition
      - name: Update ECS task definition
        id: task-def
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: .aws/task-definition.json
          container-name: ${{ env.ECR_REPOSITORY }}
          image: ${{ steps.image.outputs.image }}

      # Deploy to ECS
      - name: Deploy to ECS service
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.task-def.outputs.task-definition }}
          service: ${{ env.ECS_SERVICE }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
          force-new-deployment: true

      # Notify Slack on success
      - name: Notify Slack
        if: success()
        uses: slackapi/slack-github-action@v1
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "✅ ECS Deployment Success",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*ECS Deploy Successful*\nService: ${{ env.ECS_SERVICE }}\nImage: ${{ steps.image.outputs.image }}\nCommit: <https://github.com/${{ github.repository }}/commit/${{ github.sha }}|${{ github.sha }}>"
                  }
                }
              ]
            }

      # Notify Slack on failure
      - name: Notify Slack on failure
        if: failure()
        uses: slackapi/slack-github-action@v1
        with:
          webhook-url: ${{ secrets.SLACK_WEBHOOK }}
          payload: |
            {
              "text": "❌ ECS Deployment Failed",
              "blocks": [
                {
                  "type": "section",
                  "text": {
                    "type": "mrkdwn",
                    "text": "*ECS Deploy Failed*\nService: ${{ env.ECS_SERVICE }}\nCommit: <https://github.com/${{ github.repository }}/commit/${{ github.sha }}|${{ github.sha }}>\nCheck workflow: <https://github.com/${{ github.repository }}/actions/runs/${{ github.run_id }}|here>"
                  }
                }
              ]
            }
```

## IAM Role para GitHub Actions

```hcl
# OIDC Provider
resource "aws_iam_openid_connect_provider" "github" {
  url             = "https://token.actions.githubusercontent.com"
  client_id_list  = ["sts.amazonaws.com"]
  thumbprint_list = ["6938fd4d98bab03faadb97b34396831e3780aea1"]  # GitHub's thumbprint
}

# IAM Role para GitHub
resource "aws_iam_role" "github_actions" {
  name = "GitHubActionsRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Federated = aws_iam_openid_connect_provider.github.arn
      }
      Action = "sts:AssumeRoleWithWebIdentity"
      Condition = {
        StringEquals = {
          "token.actions.githubusercontent.com:aud" = "sts.amazonaws.com"
        }
        StringLike = {
          "token.actions.githubusercontent.com:sub" = "repo:seu-github-org/seu-repo:*"
        }
      }
    }]
  })
}

# Permissões necessárias para deploy
resource "aws_iam_role_policy" "github_actions_ecs" {
  name = "GitHubActionsECSPolicy"
  role = aws_iam_role.github_actions.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "ECRAccess"
        Effect = "Allow"
        Action = [
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:PutImage",
          "ecr:InitiateLayerUpload",
          "ecr:UploadLayerPart",
          "ecr:CompleteLayerUpload"
        ]
        Resource = "*"
      },
      {
        Sid    = "ECSDeployment"
        Effect = "Allow"
        Action = [
          "ecs:DescribeServices",
          "ecs:DescribeTaskDefinition",
          "ecs:DescribeTasks",
          "ecs:ListTasks",
          "ecs:UpdateService",
          "ecs:RegisterTaskDefinition"
        ]
        Resource = [
          aws_ecs_service.app.arn,
          aws_ecs_task_definition.app_production.arn
        ]
      },
      {
        Sid    = "PassRole"
        Effect = "Allow"
        Action = [
          "iam:PassRole"
        ]
        Resource = [
          aws_iam_role.ecs_task_role.arn,
          aws_iam_role.ecs_task_execution_role.arn
        ]
      }
    ]
  })
}
```

## Pipeline Layout

```
┌──────────────────────────────┐
│ 1. Checkout                  │
│    (código do repositório)   │
└────────────┬─────────────────┘
             │
             ▼
┌──────────────────────────────┐
│ 2. Build Docker Image        │
│    (tests included)          │
└────────────┬─────────────────┘
             │
             ▼
┌──────────────────────────────┐
│ 3. Push to ECR               │
│    (sha tag + latest)        │
└────────────┬─────────────────┘
             │
             ▼
┌──────────────────────────────┐
│ 4. Render Task Definition    │
│    (substitui image URI)     │
└────────────┬─────────────────┘
             │
             ▼
┌──────────────────────────────┐
│ 5. Deploy to ECS             │
│    (Service atualiza)        │
└────────────┬─────────────────┘
             │
             ▼
┌──────────────────────────────┐
│ 6. Wait for Stability        │
│    (healthchecks passam)     │
└────────────┬─────────────────┘
             │
             ▼
┌──────────────────────────────┐
│ 7. Notify (Slack/Teams)      │
│    (sucesso ou falha)        │
└──────────────────────────────┘
```

## Boas Práticas CI/CD

| Prática | Benefício |
|---------|-----------|
| Usar OIDC, não credenciais | Zero secrets storage |
| Build sempre antes de deploy | Falha rápido em erros |
| Tag images com SHA | Rastreabilidade |
| Wait for stability | Detecta problemas cedo |
| Rollback automático em erro | HA garantido |
| Notify statusno Slack/Teams | Visibilidade |
| Separate build e deploy steps | Paralelizável |
