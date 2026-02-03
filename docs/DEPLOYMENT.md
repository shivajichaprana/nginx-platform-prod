# Production Canary Deployment Runbook

## Overview

This repository handles **production deployments only**. It does NOT build Docker images. It deploys pre-built, immutable images from ECR that were validated in the dev environment.

## Architecture

- **ECS Service**: `nginx-fargate-demo-prod-service` (2-8 tasks)
- **Target Groups**: Blue (prod-blue) and Green (prod-green)
- **ALB**: Port 8080 with weighted routing
- **Initial State**: 100% Blue, 0% Green

## Deployment Process

### 1. Deploy New Version

1. Go to **Actions** → **Production Canary Deployment**
2. Click **Run workflow**
3. Enter:
   - **image_tag**: The ECR tag from dev (e.g., `abc1234` or `v1.2.3`)
4. **Approve** the deployment (requires reviewer approval)
5. Wait for deployment to complete
6. New tasks are running in blue target group

### 2. Verify Health

```bash
# Check ECS tasks
aws ecs list-tasks --cluster nginx-fargate-demo-cluster --service-name nginx-fargate-demo-prod-service

# Check CloudWatch logs
aws logs tail /ecs/nginx-fargate-demo-prod --follow

# Check target health
aws elbv2 describe-target-health --target-group-arn <BLUE_TG_ARN>
```

### 3. Gradual Traffic Shift (Canary)

Go to **Actions** → **Traffic Shift (Canary)** and run multiple times:

#### Step 1: 5% Canary
- Blue: 95
- Green: 5
- **Monitor for 10 minutes**

#### Step 2: 25% Canary
- Blue: 75
- Green: 25
- **Monitor for 10 minutes**

#### Step 3: 50% Split
- Blue: 50
- Green: 50
- **Monitor for 15 minutes**

#### Step 4: Full Cutover
- Blue: 0
- Green: 100
- **Monitor for 30 minutes**

### 4. Monitoring During Canary

```bash
# CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name TargetResponseTime \
  --dimensions Name=TargetGroup,Value=<TG_ARN> \
  --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Average

# Error rate
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_Target_5XX_Count \
  --dimensions Name=TargetGroup,Value=<TG_ARN> \
  --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Sum
```

### 5. Rollback (Instant)

If issues detected, **immediately** shift traffic back:

Go to **Actions** → **Traffic Shift (Canary)**:
- Blue: 100
- Green: 0

**Rollback completes in seconds** (no redeployment needed).

## Safety Guardrails

✅ **No image builds** - Only deploys pre-built images  
✅ **Manual approval** - Production environment requires reviewer  
✅ **Immutable artifacts** - Uses specific ECR tags, never `latest`  
✅ **Zero-downtime** - New tasks start before traffic shifts  
✅ **Instant rollback** - ALB weight change only (< 5 seconds)  
✅ **Least privilege** - Prod role cannot push to ECR or modify dev  
✅ **Session stickiness** - Users stay on same version during canary  

## IAM Role

**Role**: `github-actions-nginx-platform-prod`

**Permissions**:
- ECR: Read-only (pull images, describe)
- ECS: Update prod service, register task definitions
- ALB: Modify listener weights, describe target health
- CloudWatch: Read logs

**Trust Policy**: Only `shivajichaprana/nginx-platform-prod` repo, `main` branch, `production` environment

## Emergency Procedures

### Complete Rollback
1. Traffic shift: Blue 100%, Green 0
2. Update service to previous task definition revision

### Scale Up
```bash
aws ecs update-service \
  --cluster nginx-fargate-demo-cluster \
  --service nginx-fargate-demo-prod-service \
  --desired-count 8
```

### Check Service Events
```bash
aws ecs describe-services \
  --cluster nginx-fargate-demo-cluster \
  --services nginx-fargate-demo-prod-service \
  --query 'services[0].events[0:10]'
```

## Pre-Deployment Checklist

- [ ] Image tag exists in ECR
- [ ] Image was tested in dev environment
- [ ] Change ticket approved
- [ ] Rollback plan confirmed
- [ ] On-call engineer available
- [ ] Monitoring dashboards open

## Post-Deployment

- [ ] Verify 0 errors in CloudWatch
- [ ] Check ALB target health (all healthy)
- [ ] Verify response times < baseline
- [ ] Update deployment log
- [ ] Notify team in Slack

## Useful Commands

```bash
# Get current traffic weights
aws elbv2 describe-listeners --listener-arns <LISTENER_ARN>

# List available ECR images
aws ecr describe-images --repository-name nginx-demo --query 'sort_by(imageDetails,&imagePushedAt)[-10:].[imageTags[0],imagePushedAt]' --output table

# Current task definition
aws ecs describe-services --cluster nginx-fargate-demo-cluster --services nginx-fargate-demo-prod-service --query 'services[0].taskDefinition'
```
