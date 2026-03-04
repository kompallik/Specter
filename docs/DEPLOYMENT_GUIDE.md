# Specter Deployment Guide

Complete guide for deploying Specter AI Agent Cluster to AWS ECS.

---

## ✅ Prerequisites Complete

- [x] Database schema migrated
- [x] Agent JIRA tokens configured
- [x] EFS repositories cloned with worktrees
- [x] Code built (`npm run build`)

---

## 🚀 Deployment Steps

### Step 1: Create S3 Buckets

```bash
aws s3 mb s3://specter-tasks --region us-east-2
aws s3 mb s3://specter-results --region us-east-2

# Enable versioning
aws s3api put-bucket-versioning \
  --bucket specter-tasks \
  --versioning-configuration Status=Enabled \
  --region us-east-2

aws s3api put-bucket-versioning \
  --bucket specter-results \
  --versioning-configuration Status=Enabled \
  --region us-east-2
```

### Step 2: Create ECR Repositories

```bash
# Create ECR repos for each service
aws ecr create-repository --repository-name specter/orchestrator --region us-east-2
aws ecr create-repository --repository-name specter/darci-worker --region us-east-2
aws ecr create-repository --repository-name specter/scribe-worker --region us-east-2
aws ecr create-repository --repository-name specter/prism-worker --region us-east-2
aws ecr create-repository --repository-name specter/hermes-worker --region us-east-2
```

### Step 3: Build and Push Docker Images

```bash
# Get ECR login
aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 323960442001.dkr.ecr.us-east-2.amazonaws.com

# Build orchestrator
docker build -t specter/orchestrator:latest -f Dockerfile.orchestrator .
docker tag specter/orchestrator:latest 323960442001.dkr.ecr.us-east-2.amazonaws.com/specter/orchestrator:latest
docker push 323960442001.dkr.ecr.us-east-2.amazonaws.com/specter/orchestrator:latest

# Build DARCI worker
docker build -t specter/darci-worker:latest -f Dockerfile.darci .
docker tag specter/darci-worker:latest 323960442001.dkr.ecr.us-east-2.amazonaws.com/specter/darci-worker:latest
docker push 323960442001.dkr.ecr.us-east-2.amazonaws.com/specter/darci-worker:latest

# Build SCRIBE worker
docker build -t specter/scribe-worker:latest -f Dockerfile.scribe .
docker tag specter/scribe-worker:latest 323960442001.dkr.ecr.us-east-2.amazonaws.com/specter/scribe-worker:latest
docker push 323960442001.dkr.ecr.us-east-2.amazonaws.com/specter/scribe-worker:latest

# Build PRISM worker  
docker build -t specter/prism-worker:latest -f Dockerfile.prism .
docker tag specter/prism-worker:latest 323960442001.dkr.ecr.us-east-2.amazonaws.com/specter/prism-worker:latest
docker push 323960442001.dkr.ecr.us-east-2.amazonaws.com/specter/prism-worker:latest

# Build HERMES worker
docker build -t specter/hermes-worker:latest -f Dockerfile.hermes .
docker tag specter/hermes-worker:latest 323960442001.dkr.ecr.us-east-2.amazonaws.com/specter/hermes-worker:latest
docker push 323960442001.dkr.ecr.us-east-2.amazonaws.com/specter/hermes-worker:latest
```

### Step 4: Create IAM Task Execution Role

```bash
# Create role for ECS task execution
aws iam create-role \
  --role-name SpecterECSTaskRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ecs-tasks.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }' \
  --region us-east-2

# Attach policies
aws iam attach-role-policy \
  --role-name SpecterECSTaskRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

# Create inline policy for S3, RDS, Bedrock, Secrets Manager
aws iam put-role-policy \
  --role-name SpecterECSTaskRole \
  --policy-name SpecterServicePolicy \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "s3:GetObject",
          "s3:PutObject",
          "s3:ListBucket"
        ],
        "Resource": [
          "arn:aws:s3:::specter-tasks/*",
          "arn:aws:s3:::specter-results/*"
        ]
      },
      {
        "Effect": "Allow",
        "Action": [
          "bedrock:InvokeModel",
          "bedrock:InvokeModelWithResponseStream"
        ],
        "Resource": "arn:aws:bedrock:us-east-2::foundation-model/*"
      },
      {
        "Effect": "Allow",
        "Action": [
          "secretsmanager:GetSecretValue"
        ],
        "Resource": "arn:aws:secretsmanager:us-east-2:323960442001:secret:document-ai-prod/database/credentials-*"
      }
    ]
  }'
```

### Step 5: Create ECS Cluster

```bash
aws ecs create-cluster \
  --cluster-name specter-cluster \
  --region us-east-2
```

### Step 6: Register Task Definitions

See task definition files:
- `ecs/orchestrator-task-definition.json`
- `ecs/darci-worker-task-definition.json`
- `ecs/scribe-worker-task-definition.json`
- `ecs/prism-worker-task-definition.json`
- `ecs/hermes-worker-task-definition.json`

Register each:
```bash
aws ecs register-task-definition \
  --cli-input-json file://ecs/orchestrator-task-definition.json \
  --region us-east-2
```

### Step 7: Create ECS Services

```bash
# Orchestrator (1 instance)
aws ecs create-service \
  --cluster specter-cluster \
  --service-name specter-orchestrator \
  --task-definition specter-orchestrator:1 \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-0791c0e76fc4f0e81],securityGroups=[sg-0d3c5747250606f0d],assignPublicIp=DISABLED}" \
  --region us-east-2

# DARCI Worker (2 instances)
aws ecs create-service \
  --cluster specter-cluster \
  --service-name specter-darci-worker \
  --task-definition specter-darci-worker:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-0791c0e76fc4f0e81],securityGroups=[sg-0d3c5747250606f0d],assignPublicIp=DISABLED}" \
  --region us-east-2

# SCRIBE Worker (1 instance)
aws ecs create-service \
  --cluster specter-cluster \
  --service-name specter-scribe-worker \
  --task-definition specter-scribe-worker:1 \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-0791c0e76fc4f0e81],securityGroups=[sg-0d3c5747250606f0d],assignPublicIp=DISABLED}" \
  --region us-east-2

# PRISM Worker (1 instance)  
aws ecs create-service \
  --cluster specter-cluster \
  --service-name specter-prism-worker \
  --task-definition specter-prism-worker:1 \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-0791c0e76fc4f0e81],securityGroups=[sg-0d3c5747250606f0d],assignPublicIp=DISABLED}" \
  --region us-east-2

# HERMES Worker (1 instance)
aws ecs create-service \
  --cluster specter-cluster \
  --service-name specter-hermes-worker \
  --task-definition specter-hermes-worker:1 \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-0791c0e76fc4f0e81],securityGroups=[sg-0d3c5747250606f0d],assignPublicIp=DISABLED}" \
  --region us-east-2
```

---

## 🐳 Dockerfiles Needed

You need to create 5 Dockerfiles (or 1 with build args):

### Dockerfile.orchestrator
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY dist/ ./dist/
CMD ["node", "dist/orchestrator/index.js"]
```

### Dockerfile.darci
```dockerfile
FROM node:20-alpine
RUN apk add --no-cache git
WORKDIR /app
COPY package*.json ./
RUN npm ci --production
COPY dist/ ./dist/
ENV AGENT_NAME=DARCI
CMD ["node", "dist/orchestrator/workers/index.js", "DARCI"]
```

### Dockerfile.scribe, Dockerfile.prism, Dockerfile.hermes
(Same as DARCI, just change AGENT_NAME)

---

## 📋 Next Steps

1. **Create Dockerfiles** (5 files)
2. **Create ECS Task Definitions** (5 JSON files)
3. **Build and push Docker images** (~10 min)
4. **Create IAM role** (~2 min)
5. **Create ECS services** (~5 min)
6. **Test with JIRA ticket** (~10 min)

**Total Time:** ~30 minutes to full deployment

Would you like me to create the Dockerfiles and ECS task definition templates?
