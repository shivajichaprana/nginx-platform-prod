# Nginx Platform - Production Deployments

**Production-only deployment repository** for manual canary releases with human approval.

## ⚠️ Important

- **NO image builds** - Only deploys pre-built images from ECR
- **NO automated deployments** - All deployments require manual approval
- **Immutable artifacts only** - Uses specific image tags from dev pipeline

## Quick Start

### Deploy to Production

1. Get validated image tag from dev (e.g., `abc1234`)
2. Go to **Actions** → **Production Canary Deployment**
3. Enter image tag
4. Approve deployment
5. Gradually shift traffic using **Traffic Shift** workflow

[📖 Full Deployment Runbook](./docs/DEPLOYMENT.md)

## Architecture

```
Dev Pipeline (nginx-platform repo)
    ↓
Build & Push to ECR (tag: abc1234)
    ↓
Manual Testing in Dev
    ↓
Prod Deployment (this repo)
    ↓
Deploy to Green (0% traffic)
    ↓
Shift Traffic: 5% → 25% → 50% → 100%
    ↓
Monitor & Rollback if needed
```

## Workflows

### 1. Production Canary Deployment
- **Trigger**: Manual (workflow_dispatch)
- **Inputs**: image_tag
- **Approval**: Required (production environment)
- **Actions**: Deploy new task definition to ECS

### 2. Traffic Shift
- **Trigger**: Manual (workflow_dispatch)
- **Inputs**: blue_weight, green_weight
- **Approval**: Required (production environment)
- **Actions**: Modify ALB listener weights

## AWS Resources

- **Cluster**: nginx-fargate-demo-cluster
- **Service**: nginx-fargate-demo-prod-service
- **ALB**: Port 8080 (dev uses port 80)
- **Target Groups**: 
  - nginx-fargate-demo-prod-blue
  - nginx-fargate-demo-prod-green
- **IAM Role**: github-actions-nginx-platform-prod

## Setup

### 1. Create GitHub Repository

```bash
gh repo create shivajichaprana/nginx-platform-prod --public
cd nginx-platform-prod
git init
git add .
git commit -m "Initial production deployment setup"
git branch -M main
git remote add origin git@github.com:shivajichaprana/nginx-platform-prod.git
git push -u origin main
```

### 2. Configure GitHub Environment

1. Go to **Settings** → **Environments**
2. Create environment: `production`
3. Enable **Required reviewers**
4. Add reviewer(s)
5. Set **Wait timer**: 0 minutes (optional: add delay)

### 3. Deploy Infrastructure

```bash
cd ../ecs-automation-project/infrastructure
terraform apply
```

This creates:
- Production ECS service
- Blue/green target groups
- ALB listener on port 8080
- Production IAM OIDC role

### 4. First Deployment

```bash
# Get latest validated image from dev
aws ecr describe-images --repository-name nginx-demo --query 'sort_by(imageDetails,&imagePushedAt)[-1].imageTags[0]' --output text

# Deploy via GitHub Actions (manual approval required)
```

## Rollback

**Instant rollback** by shifting traffic back to blue:

```
Blue: 100%, Green: 0%
```

No redeployment needed. Rollback completes in < 5 seconds.

## Monitoring

```bash
# CloudWatch logs
aws logs tail /ecs/nginx-fargate-demo-prod --follow

# Target health
aws elbv2 describe-target-health --target-group-arn <TG_ARN>

# Service status
aws ecs describe-services --cluster nginx-fargate-demo-cluster --services nginx-fargate-demo-prod-service
```

## Security

- ✅ OIDC authentication (no stored credentials)
- ✅ Least privilege IAM (read-only ECR, prod service only)
- ✅ Environment protection (required reviewers)
- ✅ Immutable artifacts (specific tags only)
- ✅ Audit trail (GitHub Actions logs)

## Cost

Additional cost over dev environment: ~$30/month
- 2 additional ECS tasks (minimum)
- Additional target group
- CloudWatch logs

## Support

- **Runbook**: [docs/DEPLOYMENT.md](./docs/DEPLOYMENT.md)
- **Infrastructure**: [ecs-automation-project/infrastructure](../ecs-automation-project/infrastructure)
- **Dev Pipeline**: [nginx-platform](https://github.com/shivajichaprana/nginx-platform)
