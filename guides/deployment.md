# Production Deployment Guide

This guide covers deploying the ChainLearn platform to production. The infrastructure is managed via Terraform and deployed through GitHub Actions CI/CD pipelines.

## Architecture

Production runs on AWS with the following topology:

```
                    +------------------+
                    |   CloudFront CDN |
                    |   (frontend)     |
                    +--------+---------+
                             |
                    +--------v---------+
                    |   ALB            |
                    |   (load balancer)|
                    +--------+---------+
                             |
              +--------------+--------------+
              |              |              |
     +--------v---+  +------v-----+  +-----v------+
     | API (ECS)  |  | AI (ECS)   |  | Indexer    |
     | 2-4 tasks  |  | 1-2 tasks  |  | 1 task     |
     +--------+---+  +------+-----+  +-----+------+
              |              |              |
     +--------v--------------v--------------v------+
     |              Amazon RDS (PostgreSQL)         |
     |              Amazon ElastiCache (Redis)      |
     +---------------------------------------------+
```

## Environments

| Environment | URL | Purpose |
|-------------|-----|---------|
| Local | localhost:* | Development |
| Staging | staging.chainlearn.dev | Pre-production testing |
| Production | chainlearn.dev | Live platform |

## Prerequisites

- AWS CLI configured with appropriate IAM roles
- Terraform 1.7+
- Docker
- Access to the `chainlearn-infra` repository
- Stellar mainnet account with funded XLM for contract operations

## Infrastructure Provisioning

### Initial Setup

```bash
cd chainlearn-infra/terraform

# Initialize Terraform
terraform init

# Review the plan
terraform plan -var-file=environments/staging.tfvars

# Apply (staging first)
terraform apply -var-file=environments/staging.tfvars
```

### Terraform Modules

| Module | Resources |
|--------|-----------|
| `vpc` | VPC, subnets, NAT gateways |
| `ecs` | ECS cluster, services, task definitions |
| `rds` | PostgreSQL instance, read replicas |
| `elasticache` | Redis cluster |
| `cloudfront` | CDN distribution for frontend |
| `alb` | Application Load Balancer |
| `secrets` | AWS Secrets Manager for credentials |

## CI/CD Pipeline

Deployment is automated via GitHub Actions. Each repo has its own pipeline.

### Pipeline Stages

```
Push to main -> Build -> Test -> Build Docker Image -> Push to ECR -> Deploy to Staging -> Manual Approval -> Deploy to Production
```

### Workflow Files

**API (`chainlearn-api/.github/workflows/deploy.yml`):**

```yaml
name: Deploy API
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
      - run: npm ci
      - run: npm test

  build-and-deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1
      - uses: aws-actions/amazon-ecr-login@v2
      - run: |
          docker build -t $ECR_REGISTRY/chainlearn-api:$GITHUB_SHA .
          docker push $ECR_REGISTRY/chainlearn-api:$GITHUB_SHA
      - run: |
          aws ecs update-service \
            --cluster chainlearn \
            --service chainlearn-api \
            --force-new-deployment
```

### Deploying Individual Services

Each service can be deployed independently:

```bash
# Build and push Docker image
docker build -t chainlearn-api .
docker tag chainlearn-api:latest $ECR_REGISTRY/chainlearn-api:latest
docker push $ECR_REGISTRY/chainlearn-api:latest

# Trigger ECS deployment
aws ecs update-service \
  --cluster chainlearn \
  --service chainlearn-api \
  --force-new-deployment
```

## Database Migrations

Migrations run as part of the API deployment:

```bash
# In the ECS task entrypoint:
npx prisma migrate deploy
```

**Important:** Always test migrations on staging first. Back up production data before running destructive migrations.

### Manual Migration

```bash
# Connect to production database (via bastion host)
aws ssm start-session --target bastion-host-id

# Run migration
cd /app && npx prisma migrate deploy
```

## Smart Contract Deployment

Contracts are deployed separately from the application services. See [Contract Deployment](../contracts/deployment.md) for the full guide.

### Post-Deployment Steps

After deploying contracts to mainnet:

1. Update AWS Secrets Manager with new contract addresses
2. Restart API and Indexer services to pick up new addresses
3. Verify contracts on Stellar Expert
4. Update the `chainlearn-infra` repo with the addresses

## Environment Variables

### AWS Secrets Manager

Store sensitive values in Secrets Manager, not in environment files:

```bash
# Store database URL
aws secretsmanager create-secret \
  --name chainlearn/production/database-url \
  --secret-string "postgresql://user:pass@rds-host:5432/chainlearn"

# Store JWT secret
aws secretsmanager create-secret \
  --name chainlearn/production/jwt-secret \
  --secret-string "$(openssl rand -hex 32)"

# Store Cohere API key
aws secretsmanager create-secret \
  --name chainlearn/production/cohere-api-key \
  --secret-string "your-cohere-key"

# Store Stellar admin secret key
aws secretsmanager create-secret \
  --name chainlearn/production/stellar-admin-key \
  --secret-string "SDNM..."
```

### ECS Task Definition

```json
{
  "secrets": [
    {
      "name": "DATABASE_URL",
      "valueFrom": "arn:aws:secretsmanager:us-east-1:123456:secret:chainlearn/production/database-url"
    },
    {
      "name": "JWT_SECRET",
      "valueFrom": "arn:aws:secretsmanager:us-east-1:123456:secret:chainlearn/production/jwt-secret"
    }
  ]
}
```

## Monitoring and Alerting

### CloudWatch Alarms

| Alarm | Threshold | Action |
|-------|-----------|--------|
| API 5xx error rate | > 1% | PagerDuty alert |
| API latency p99 | > 2s | Slack notification |
| CPU utilization | > 80% | Auto-scale ECS tasks |
| Database connections | > 80% of max | PagerDuty alert |
| Disk usage | > 85% | Slack notification |
| Indexer lag | > 60 seconds | PagerDuty alert |

### Health Checks

Each service exposes a health endpoint:

```bash
# API health
curl https://api.chainlearn.dev/health
# {"status":"ok","version":"1.2.3","uptime":12345}

# AI service health
curl https://ai.chainlearn.dev/health
# {"status":"ok","model":"command-r-plus"}

# Indexer health
curl https://indexer.chainlearn.dev/health
# {"status":"ok","last_ledger":12345678,"lag_seconds":5}
```

### Logs

```bash
# View API logs
aws logs tail /ecs/chainlearn-api --follow

# View AI service logs
aws logs tail /ecs/chainlearn-ai --follow

# Search for errors
aws logs filter-log-events \
  --log-group-name /ecs/chainlearn-api \
  --filter-pattern "ERROR"
```

## Scaling

### Auto-Scaling Configuration

```hcl
# terraform/modules/ecs/autoscaling.tf
resource "aws_appautoscaling_policy" "api_cpu" {
  name               = "api-cpu-scaling"
  service_namespace  = aws_appautoscaling_target.api.service_namespace
  scalable_dimension = aws_appautoscaling_target.api.scalable_dimension
  resource_id        = aws_appautoscaling_target.api.resource_id

  policy_type = "TargetTrackingScaling"
  target_tracking_scaling_policy_configuration {
    target_value = 70.0
    scale_in_cooldown  = 300
    scale_out_cooldown = 60
  }
}
```

| Service | Min Tasks | Max Tasks | Scale Trigger |
|---------|-----------|-----------|---------------|
| API | 2 | 8 | CPU > 70% |
| AI | 1 | 4 | CPU > 60% |
| Indexer | 1 | 1 | Manual only |

## Rollback

### Application Rollback

```bash
# Roll back to a previous task definition
aws ecs update-service \
  --cluster chainlearn \
  --service chainlearn-api \
  --task-definition chainlearn-api:123  # Previous revision
```

### Database Rollback

Database rollbacks require manual intervention:

1. Stop the API service
2. Restore from the latest automated snapshot
3. Re-run migrations to the desired version
4. Restart the API service

## Disaster Recovery

### Backups

- **PostgreSQL:** Automated daily snapshots with 30-day retention
- **Redis:** Not persisted (cache only, can be rebuilt)
- **Smart Contracts:** Immutable on-chain, no backup needed
- **Frontend:** Deployed from Git, can be redeployed from any commit

### Recovery Time Objectives

| Component | RTO | RPO |
|-----------|-----|-----|
| Frontend | 5 minutes | 0 (Git) |
| API | 15 minutes | 0 (stateless) |
| Database | 30 minutes | 24 hours (last snapshot) |
| Indexer | 15 minutes | 0 (rebuilds from chain) |
| Smart Contracts | N/A | N/A (immutable) |
