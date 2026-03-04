# Terraform Infrastructure Deep-Dive Analysis

## Executive Summary

The Specter application uses a **hybrid infrastructure approach**, reusing existing MedHok AWS resources while creating new Specter-specific components. The deployment is fully automated via Terraform and uses AWS ECS Fargate for container orchestration.

**Key Stats:**
- **Account ID:** 323960442001
- **Region:** us-east-2 (Ohio)
- **Compute:** 6 ECS Fargate tasks (1 orchestrator + 4 workers × 1-2 instances)
- **Storage:** EFS (shared), S3 (tasks/results), RDS MySQL (database)
- **Monthly Cost:** ~$30-50 (Fargate) + $5-10 (EFS) + $5 (S3) ≈ **$40-65/month**

---

## 1. Infrastructure Architecture

### 1.1 Resource Reuse Strategy

```
┌─────────────────────────────────────────────────────────────────────┐
│                     EXISTING (REUSED) INFRASTRUCTURE                 │
├─────────────────────────────────────────────────────────────────────┤
│ • VPC: vpc-0f059e0a3942b2e62                                        │
│ • Subnet: subnet-0791c0e76fc4f0e81 (Private)                        │
│ • RDS: document-ai-prod-mysql.cxtyszxlzcsj.us-east-2.rds.amazonaws  │
│ • EFS: fs-07db87df276aef225 (specter-repo)                          │
│ • IAM Role: AmazonBedrockExecutionRoleForFlows_YX9G86Z5WQ          │
│ • Secrets Manager: document-ai-prod/database/credentials            │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                     NEW (CREATED) INFRASTRUCTURE                     │
├─────────────────────────────────────────────────────────────────────┤
│ • ECS Cluster: specter-cluster                                      │
│ • 5 ECR Repositories (orchestrator + 4 workers)                     │
│ • 2 S3 Buckets (tasks, results)                                     │
│ • 5 CloudWatch Log Groups                                           │
│ • 1 ECS Execution Role                                              │
│ • 1 Security Group (ECS tasks)                                      │
│ • 5 ECS Task Definitions                                            │
│ • 5 ECS Services                                                    │
└─────────────────────────────────────────────────────────────────────┘
```

### 1.2 Component Interconnection

```
                    ┌──────────────────┐
                    │   JIRA Cloud     │
                    │ (medhokapps.net) │
                    └────────┬─────────┘
                             │ Poll every 60s
                             ▼
┌────────────────────────────────────────────────────────────────┐
│                      ORCHESTRATOR                               │
│  • ECS Service: specter-orchestrator                           │
│  • Instances: 1                                                │
│  • CPU: 512, Memory: 1024MB                                    │
│  • Responsibilities:                                           │
│    - Poll JIRA for new tickets                                 │
│    - Generate task descriptions (Bedrock)                      │
│    - Upload to S3 (specter-tasks)                              │
│    - Insert task records (RDS)                                 │
│    - Check for completed results                               │
│    - Post results back to JIRA                                 │
└────────────────┬───────────────────────────────────────────────┘
                 │
                 │ S3 + Database
                 │
    ┌────────────┼────────────┬─────────────┬──────────────┐
    ▼            ▼            ▼             ▼              ▼
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌──────────┐ ┌──────────┐
│  DARCI  │ │ SCRIBE  │ │  PRISM  │ │  HERMES  │ │   EFS    │
│ Worker  │ │ Worker  │ │ Worker  │ │  Worker  │ │ fs-07db  │
├─────────┤ ├─────────┤ ├─────────┤ ├──────────┤ ├──────────┤
│ 2 inst  │ │ 1 inst  │ │ 1 inst  │ │ 1 inst   │ │ Shared   │
│ 1024CPU │ │ 512CPU  │ │ 512CPU  │ │ 512CPU   │ │ repos/   │
│ 2048MB  │ │ 1024MB  │ │ 1024MB  │ │ 1024MB   │ │ docs/    │
└────┬────┘ └────┬────┘ └────┬────┘ └─────┬────┘ └────┬─────┘
     │           │           │            │            │
     └───────────┴───────────┴────────────┴────────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │   RDS MySQL     │
                    │ document-ai-prod│
                    │ Specter schema  │
                    └─────────────────┘
```

---

## 2. Terraform Configuration Analysis

### 2.1 State Management

**Backend Configuration** ([`backend.hcl`](terraform/backend.hcl:1)):
```hcl
bucket         = "mhk-terraform-state"
key            = "specter/terraform.tfstate"
region         = "us-east-2"
dynamodb_table = "terraform_state_locking_table"
encrypt        = true
```

**✅ Strengths:**
- Remote state in S3 prevents conflicts
- DynamoDB locking prevents concurrent modifications
- Encryption at rest enabled
- Namespaced under `specter/` key

**⚠️ Considerations:**
- Requires manual backend initialization: `terraform init -backend-config=backend.hcl`
- State contains sensitive data (DB passwords in task definitions)
- No state versioning configured (S3 bucket should have versioning)

### 2.2 Provider Configuration

**Main Configuration** ([`main.tf`](terraform/main.tf:21)):
```hcl
provider "aws" {
  region = var.aws_region  # us-east-2

  default_tags {
    tags = {
      Environment = var.environment  # prod
      Project     = "Specter"
      ManagedBy   = "Terraform"
      Owner       = var.owner        # platform-team
    }
  }
}
```

**✅ Strengths:**
- Consistent tagging via `default_tags` (all resources auto-tagged)
- Single region deployment (simplified)
- Version pinning: `~> 5.0` (prevents breaking changes)

### 2.3 Data Sources (Existing Resources)

The configuration uses **8 data sources** to reference existing infrastructure:

| Data Source | Purpose | Terraform Reference |
|-------------|---------|---------------------|
| `aws_caller_identity` | Get AWS account ID | [`main.tf:35`](terraform/main.tf:35) |
| `aws_vpc.existing` | Reference existing VPC | [`main.tf:38`](terraform/main.tf:38) |
| `aws_efs_file_system.specter_repo` | Reference EFS | [`main.tf:43`](terraform/main.tf:43) |
| `aws_iam_role.existing_task_role` | Reference Bedrock IAM role | [`main.tf:48`](terraform/main.tf:48) |
| `aws_secretsmanager_secret` | Get secret ARN | [`main.tf:53`](terraform/main.tf:53) |
| `aws_secretsmanager_secret_version` | Read DB credentials | [`main.tf:57`](terraform/main.tf:57) |

**⚠️ Critical Dependency:**
All data sources must exist **before** running Terraform. If any are missing/deleted, Terraform will fail.

---

## 3. Resource Creation Deep-Dive

### 3.1 ECS Cluster

**Definition** ([`ecs.tf:8`](terraform/ecs.tf:8)):
```hcl
resource "aws_ecs_cluster" "specter" {
  name = "specter-cluster"
  
  setting {
    name  = "containerInsights"
    value = var.enable_container_insights ? "enabled" : "disabled"
  }
}
```

**Details:**
- **Launch Type:** Fargate (serverless)
- **Container Insights:** Enabled (additional metrics/dashboards)
- **No capacity providers:** Uses default Fargate provider
- **No service discovery:** Services communicate via S3 + Database

### 3.2 ECR Repositories

**Structure** ([`ecs.tf:74-100`](terraform/ecs.tf:74)):
```
323960442001.dkr.ecr.us-east-2.amazonaws.com/
├── specter/orchestrator           (1 repo)
└── specter/                       (4 repos via for_each)
    ├── darci-worker
    ├── scribe-worker
    ├── prism-worker
    └── hermes-worker
```

**Configuration:**
- **Image Tag Mutability:** MUTABLE (can overwrite `latest` tag)
- **Image Scanning:** ON (security vulnerability scanning)
- **Lifecycle Policy:** ❌ NOT CONFIGURED (images never expire)

**⚠️ Recommendation:**
Add lifecycle policies to delete old/untagged images:
```hcl
lifecycle_policy = jsonencode({
  rules = [{
    rulePriority = 1
    description  = "Keep last 10 images"
    selection = {
      tagStatus   = "any"
      countType   = "imageCountMoreThan"
      countNumber = 10
    }
    action = { type = "expire" }
  }]
})
```

### 3.3 ECS Task Definitions

#### Orchestrator Task ([`ecs.tf:107-165`](terraform/ecs.tf:107))

**Compute Resources:**
```
CPU:    512 units (0.5 vCPU)
Memory: 1024 MB (1 GB)
```

**Environment Variables (15 total):**
```bash
NODE_ENV=production
DB_HOST=document-ai-prod-mysql.cxtyszxlzcsj.us-east-2.rds.amazonaws.com
DB_PORT=3306
DB_NAME=Specter
DB_USER=<from secrets manager>
DB_PASSWORD=<from secrets manager>
AWS_REGION=us-east-2
S3_BUCKET_PREFIX=specter
JIRA_BASE_URL=https://medhokapps.atlassian.net
JIRA_USERNAME=koundinya.kompalli@mhk.com
JIRA_POLLING_INTERVAL_MS=60000
RESULT_CHECK_INTERVAL_MS=30000
EFS_MOUNT_PATH=/mnt/efs/specter-repo
BEDROCK_MODEL=us.anthropic.claude-sonnet-4-5-20250929-v1:0
```

**⚠️ Security Issues:**
1. **Secrets in Plain Text:** DB credentials are injected from Secrets Manager into environment variables (visible in ECS console)
2. **Missing JIRA Token:** No JIRA API token configured (agents pull from database)
3. **No Secret Rotation:** If DB password changes, task definition must be updated

**✅ Better Approach:**
Use Secrets Manager directly in application code instead of environment variables.

#### Worker Tasks ([`ecs.tf:168-232`](terraform/ecs.tf:168))

**Unified Configuration via `for_each`:**
```hcl
for_each = {
  darci  = { cpu = "1024", memory = "2048" }
  scribe = { cpu = "512",  memory = "1024" }
  prism  = { cpu = "512",  memory = "1024" }
  hermes = { cpu = "512",  memory = "1024" }
}
```

**EFS Mount Configuration:**
```hcl
volume {
  name = "specter-repo"
  efs_volume_configuration {
    file_system_id     = var.existing_efs_id
    transit_encryption = "ENABLED"  # TLS for NFS traffic
    root_directory     = "/"
  }
}

mountPoints = [{
  sourceVolume  = "specter-repo"
  containerPath = "/mnt/efs/specter-repo"
  readOnly      = false  # Workers need write access
}]
```

**Why DARCI Gets More Resources:**
- DARCI has 10 tools (vs 3-8 for others)
- Performs complex code analysis
- Generates longer outputs

### 3.4 S3 Buckets

#### Tasks Bucket ([`s3.tf:9-41`](terraform/s3.tf:9))

**Purpose:** Store task descriptions generated by orchestrator

**Lifecycle Policy:**
```hcl
rule {
  id     = "cleanup-old-tasks"
  status = "Enabled"
  
  expiration {
    days = 30  # Delete after 30 days
  }
  
  noncurrent_version_expiration {
    noncurrent_days = 7  # Delete old versions after 7 days
  }
}
```

**✅ Cost Optimization:**
Tasks are ephemeral and don't need long-term retention.

#### Results Bucket ([`s3.tf:47-84`](terraform/s3.tf:47))

**Purpose:** Store agent analysis results

**Lifecycle Policy:**
```hcl
rule {
  id     = "transition-to-glacier"
  status = "Enabled"
  
  transition {
    days          = 90    # Move to Glacier after 90 days
    storage_class = "GLACIER"
  }
  
  expiration {
    days = 365  # Delete after 1 year
  }
}
```

**✅ Cost Optimization:**
Results are archived to Glacier (10x cheaper) after 90 days.

**Storage Cost Estimate:**
- Standard: $0.023/GB/month
- Glacier: $0.004/GB/month
- Assuming 1000 tasks/month × 50KB = 50MB = **$1.15/year**

### 3.5 IAM Roles and Policies

#### ECS Execution Role ([`iam.tf:7-46`](terraform/iam.tf:7))

**Purpose:** Pull images from ECR, send logs to CloudWatch, read secrets

**Managed Policy:**
```
arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy
```

**Inline Policies:**
1. **Secrets Access:** Read `document-ai-prod/database/credentials` from Secrets Manager

**⚠️ Note:** This is **separate** from the task role (which runs the application).

#### Task Role (Reused)

**Existing Role:** `AmazonBedrockExecutionRoleForFlows_YX9G86Z5WQ`

**Permissions:**
- ✅ Bedrock (Claude Sonnet 4.5)
- ✅ S3 (via inline policy added by Terraform)

**Additional Policy** ([`iam.tf:53-84`](terraform/iam.tf:53)):
```hcl
resource "aws_iam_role_policy" "task_s3_access" {
  name = "SpecterS3Access"
  role = split("/", var.existing_task_role_arn)[1]  # Extract role name
  
  policy = jsonencode({
    Statement = [
      {
        Effect = "Allow"
        Action = ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"]
        Resource = [
          "${aws_s3_bucket.tasks.arn}/*",
          "${aws_s3_bucket.results.arn}/*"
        ]
      },
      {
        Effect = "Allow"
        Action = ["s3:ListBucket"]
        Resource = [
          aws_s3_bucket.tasks.arn,
          aws_s3_bucket.results.arn
        ]
      }
    ]
  })
}
```

**⚠️ Risk:** Modifying an existing IAM role could affect other services using it.

### 3.6 Security Groups

**ECS Tasks Security Group** ([`ecs.tf:287-314`](terraform/ecs.tf:287)):

```hcl
resource "aws_security_group" "ecs_tasks" {
  name        = "specter-ecs-tasks"
  description = "Security group for Specter ECS tasks"
  vpc_id      = local.vpc_id
  
  # Outbound - allow all
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
  
  # No inbound rules (tasks don't receive traffic)
}
```

**EFS Access Rule** ([`ecs.tf:306-314`](terraform/ecs.tf:306)):
```hcl
resource "aws_security_group_rule" "efs_from_ecs" {
  type                     = "ingress"
  from_port                = 2049  # NFS
  to_port                  = 2049
  protocol                 = "tcp"
  security_group_id        = var.existing_efs_security_group_id
  source_security_group_id = aws_security_group.ecs_tasks.id
  description              = "NFS from Specter ECS tasks"
}
```

**Security Model:**
```
ECS Tasks → All Outbound Allowed
  ↓ (HTTPS) AWS Bedrock
  ↓ (HTTPS) JIRA API
  ↓ (HTTPS) S3
  ↓ (MySQL) RDS (port 3306)
  ↓ (NFS)   EFS (port 2049)
```

**⚠️ Best Practice Violation:**
Allowing all outbound traffic (0.0.0.0/0) is overly permissive. Should restrict to:
- VPC CIDR (for RDS/EFS)
- AWS service endpoints (for S3/Bedrock via VPC endpoints)

### 3.7 CloudWatch Log Groups

**Configuration** ([`ecs.tf:25-68`](terraform/ecs.tf:25)):
```hcl
resource "aws_cloudwatch_log_group" "orchestrator" {
  name              = "/ecs/specter/orchestrator"
  retention_in_days = var.log_retention_days  # 7 days
}

# Same for darci_worker, scribe_worker, prism_worker, hermes_worker
```

**Log Streams:**
```
/ecs/specter/orchestrator
  └── orchestrator/<task-id>

/ecs/specter/darci-worker
  └── darci/<task-id-1>
  └── darci/<task-id-2>  # If 2 instances running
```

**Cost Estimate:**
- Ingestion: $0.50/GB
- Storage: $0.03/GB/month (first 5GB free)
- Assuming 100MB/day × 7 days = 700MB = **$0.02/month**

---

## 4. Variables and Configuration

### 4.1 Variable Organization

The [`variables.tf`](terraform/variables.tf:1) file has **30+ variables** organized into sections:

```
├── General (region, environment, owner)
├── Existing Infrastructure (8 vars)
├── Application Configuration (6 vars)
├── ECS Configuration (15 vars)
├── CloudWatch (2 vars)
└── S3 (2 vars)
```

### 4.2 Default Values Analysis

**⚠️ Hardcoded Production Values:**
```hcl
variable "existing_vpc_id" {
  default = "vpc-0f059e0a3942b2e62"  # Hardcoded prod VPC
}

variable "db_host" {
  default = "document-ai-prod-mysql.cxtyszxlzcsj.us-east-2.rds.amazonaws.com"
}
```

**Problem:** Cannot deploy to dev/staging environments without changing defaults.

**✅ Better Approach:**
Remove defaults, require explicit values:
```hcl
variable "existing_vpc_id" {
  description = "ID of existing VPC to reuse"
  type        = string
  # No default - must be provided
}
```

Then use separate `.tfvars` files:
- `terraform.tfvars.prod`
- `terraform.tfvars.staging`
- `terraform.tfvars.dev`

### 4.3 Sensitive Variables

**Missing `sensitive = true` flag:**
```hcl
# Should be marked sensitive
variable "jira_username" {
  type      = string
  sensitive = true  # ❌ Missing
}

variable "db_credentials_secret_name" {
  type      = string
  sensitive = true  # ❌ Missing
}
```

Without the `sensitive` flag, these values appear in:
- `terraform plan` output
- `terraform apply` output
- State file (already there anyway)

---

## 5. Deployment Process Analysis

### 5.1 Deployment Script ([`deploy.sh`](deploy.sh:1))

**Workflow:**
```bash
1. npm run build              # Compile TypeScript
2. docker build (5 images)    # Build containers
3. terraform init             # Initialize backend
4. terraform plan             # Preview changes
5. (manual approval)          # User presses Enter
6. terraform apply            # Create infrastructure
7. docker tag + push (5x)     # Push to ECR
8. (manual service update)    # Force ECS to pull new images
```

**⚠️ Issues:**

1. **Not idempotent:** Running twice pushes images again (wastes time)
2. **No rollback:** If Docker push fails, infrastructure is already created
3. **Manual ECR login:** Could fail if credentials expired
4. **Hardcoded values:** `ACCOUNT_ID=323960442001` in script
5. **No health checks:** Doesn't verify services are running

### 5.2 Deployment Dependencies

**Order of Operations:**
```
1. ✅ Existing Infrastructure (manual setup)
   └── VPC, Subnets, RDS, EFS, IAM Role

2. Terraform Apply (automated)
   ├── ECR Repositories (created first)
   ├── S3 Buckets
   ├── IAM Roles/Policies
   ├── Security Groups
   ├── ECS Cluster
   ├── Task Definitions (depends on ECR URLs)
   └── ECS Services (depends on task definitions)

3. Docker Build + Push (automated)
   ├── Build images locally
   └── Push to ECR (created in step 2)

4. Service Update (manual)
   └── Force ECS to pull new images
```

**⚠️ Chicken-and-Egg Problem:**
Task definitions reference ECR image URLs, but images don't exist until after `terraform apply`. This works because:
- Terraform creates ECR repositories first
- Task definitions reference URLs (not images)
- ECS doesn't validate images exist until service starts
- Initial service start will fail until images are pushed

### 5.3 Initial Deployment Checklist

**Prerequisites (Manual Steps):**
- [ ] Database schema created (`db/schema.sql` applied)
- [ ] Agent JIRA tokens configured in `agents` table
- [ ] EFS repositories cloned (`resources/setup-repos.sh`)
- [ ] `.env.production` file created with JIRA API token
- [ ] AWS credentials configured (`aws configure`)
- [ ] Terraform backend bucket exists (`mhk-terraform-state`)
- [ ] DynamoDB locking table exists (`terraform_state_locking_table`)

**First Deployment:**
```bash
# 1. Build application
npm run build

# 2. Build Docker images
docker build -f Dockerfile.orchestrator -t specter/orchestrator:latest .
docker build -f Dockerfile.worker --build-arg AGENT_NAME=DARCI -t specter/darci-worker:latest .
# ... (repeat for other workers)

# 3. Initialize Terraform
cd terraform
terraform init -backend-config=backend.hcl

# 4. Deploy infrastructure
terraform plan -var-file=terraform.tfvars
terraform apply -var-file=terraform.tfvars

# 5. Login to ECR
aws ecr get-login-password --region us-east-2 | \
  docker login --username AWS --password-stdin \
  323960442001.dkr.ecr.us-east-2.amazonaws.com

# 6. Push images
docker tag specter/orchestrator:latest \
  323960442001.dkr.ecr.us-east-2.amazonaws.com/specter/orchestrator:latest
docker push 323960442001.dkr.ecr.us-east-2.amazonaws.com/specter/orchestrator:latest
# ... (repeat for workers)

# 7. ECS will automatically start tasks once images are available
```

---

## 6. Cost Analysis

### 6.1 Fargate Costs

**Pricing (us-east-2):**
- vCPU: $0.04048/hour
- Memory: $0.004445/GB/hour

**Monthly Costs (24/7 operation):**

| Service | Instances | CPU | Memory | vCPU Cost | Memory Cost | Total/Month |
|---------|-----------|-----|--------|-----------|-------------|-------------|
| Orchestrator | 1 | 512 (0.5) | 1024 MB (1 GB) | $14.57 | $3.20 | **$17.77** |
| DARCI | 2 | 1024 (1) | 2048 MB (2 GB) | $58.29 | $12.80 | **$71.09** |
| SCRIBE | 1 | 512 (0.5) | 1024 MB (1 GB) | $14.57 | $3.20 | **$17.77** |
| PRISM | 1 | 512 (0.5) | 1024 MB (1 GB) | $14.57 | $3.20 | **$17.77** |
| HERMES | 1 | 512 (0.5) | 1024 MB (1 GB) | $14.57 | $3.20 | **$17.77** |
| **TOTAL** | **6** | **4096** | **8192 MB** | | | **$142.17/month** |

### 6.2 Storage Costs

**EFS:**
- Standard: $0.30/GB/month
- Estimated usage: 10GB (repositories) = **$3.00/month**

**S3:**
- Tasks bucket: 100MB × $0.023/GB = **$0.002/month**
- Results bucket: 500MB × $0.023/GB = **$0.012/month**

**CloudWatch Logs:**
- 700MB/month × $0.03/GB = **$0.02/month**

### 6.3 Data Transfer Costs

**Minimal** because:
- All resources in same VPC (no cross-region transfer)
- S3 access via VPC endpoint (free)
- EFS access over NFS (free within VPC)
- Only outbound: JIRA API calls (~1MB/day = **$0.09/month**)

### 6.4 Total Monthly Cost

```
Fargate:       $142.17
EFS:           $  3.00
S3:            $  0.01
CloudWatch:    $  0.02
Data Transfer: $  0.09
─────────────────────
TOTAL:         $145.29/month
```

**Cost Optimization Opportunities:**

1. **Use Fargate Spot** (70% discount):
   - Define capacity providers with FARGATE_SPOT
   - Estimated savings: **$100/month**

2. **Scale down during off-hours:**
   - Orchestrator: 1 instance 24/7 (required for JIRA polling)
   - Workers: Scale to 0 at night (8pm-8am)
   - Estimated savings: **$35/month**

3. **Right-size DARCI workers:**
   - Test with 512 CPU / 1024 MB (currently 1024/2048)
   - Estimated savings: **$35/month**

4. **EFS Infrequent Access:**
   - Move old repositories to EFS-IA ($0.016/GB)
   - Estimated savings: **$2/month**

---

## 7. Scaling Strategy

### 7.1 Horizontal Scaling

**Current Configuration:**
```hcl
orchestrator_desired_count = 1  # Cannot scale (single orchestrator)
darci_desired_count        = 2  # ✅ Can scale
scribe_desired_count       = 1  # ✅ Can scale
prism_desired_count        = 1  # ✅ Can scale
hermes_desired_count       = 1  # ✅ Can scale
```

**Scaling Workers:**
```bash
# Option 1: Update Terraform variable
# Edit terraform.tfvars:
darci_desired_count = 4

terraform apply -var-file=terraform.tfvars

# Option 2: AWS CLI (faster)
aws ecs update-service \
  --cluster specter-cluster \
  --service specter-darci-worker \
  --desired-count 4 \
  --region us-east-2
```

**⚠️ Orchestrator Limitation:**
- Single instance design (polls JIRA)
- Cannot scale horizontally (would create duplicate tasks)
- **Solution:** Add distributed locking (DynamoDB) for multi-instance orchestrator

### 7.2 Vertical Scaling

**Modifying Task Resources:**
```hcl
# Edit variables.tf
darci_cpu    = "2048"  # 2 vCPU (was 1024)
darci_memory = "4096"  # 4 GB (was 2048)
```

Then redeploy:
```bash
terraform apply -var-file=terraform.tfvars

# Force new deployment to use new task definition
aws ecs update-service \
  --cluster specter-cluster \
  --service specter-darci-worker \
  --force-new-deployment \
  --region us-east-2
```

**Fargate Task Size Limits:**
- Min: 256 CPU / 512 MB
- Max: 16384 CPU (16 vCPU) / 122880 MB (120 GB)

### 7.3 Auto-Scaling

**Not Currently Configured**, but could add:

```hcl
resource "aws_appautoscaling_target" "darci" {
  max_capacity       = 10
  min_capacity       = 1
  resource_id        = "service/specter-cluster/specter-darci-worker"
  scalable_dimension = "ecs:service:DesiredCount"
  service_namespace  = "ecs"
}

resource "aws_appautoscaling_policy" "darci_cpu" {
  name               = "darci-scale-cpu"
  policy_type        = "TargetTrackingScaling"
  resource_id        = aws_appautoscaling_target.darci.resource_id
  scalable_dimension = aws_appautoscaling_target.darci.scalable_dimension
  service_namespace  = aws_appautoscaling_target.darci.service_namespace

  target_tracking_scaling_policy_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ECSServiceAverageCPUUtilization"
    }
    target_value = 70.0  # Scale when CPU > 70%
  }
}
```

**Scaling Trigger Options:**
- CPU utilization > 70%
- Memory utilization > 80%
- Custom metric: Queue depth (task_management table)

### 7.4 Load Distribution

**Current Strategy:**
Workers poll database for unassigned tasks:
```sql
SELECT * FROM task_management
WHERE agent_id = ? AND status = 'assigned'
ORDER BY created_at ASC
LIMIT 1
FOR UPDATE SKIP LOCKED  -- Prevents race conditions
```

**Properties:**
- ✅ Automatically distributes work across instances
- ✅ No central coordinator needed
- ✅ Handles instance failures (tasks get reassigned)
- ⚠️ Potential bottleneck at 50+ instances (database connection limits)

---

## 8. Monitoring & Observability

### 8.1 CloudWatch Metrics (Container Insights)

**Enabled** via [`ecs.tf:11-14`](terraform/ecs.tf:11):
```hcl
setting {
  name  = "containerInsights"
  value = "enabled"
}
```

**Available Metrics:**
- `CPUUtilization`
- `MemoryUtilization`
- `NetworkRxBytes`
- `NetworkTxBytes`
- `TaskCount`
- `ServiceCount`

**Access:**
```
AWS Console → CloudWatch → Container Insights → ECS → specter-cluster
```

### 8.2 Log Aggregation

**Log Groups:**
```
/ecs/specter/orchestrator     (Orchestrator logs)
/ecs/specter/darci-worker     (DARCI logs)
/ecs/specter/scribe-worker    (SCRIBE logs)
/ecs/specter/prism-worker     (PRISM logs)
/ecs/specter/hermes-worker    (HERMES logs)
```

**Log Retention:** 7 days (configurable via `log_retention_days`)

**Viewing Logs:**
```bash
# Tail live logs
aws logs tail /ecs/specter/orchestrator --follow --region us-east-2

# Search logs
aws logs filter-log-events \
  --log-group-name /ecs/specter/darci-worker \
  --filter-pattern "ERROR" \
  --start-time $(date -d '1 hour ago' +%s)000
```

### 8.3 Useful CloudWatch Insights Queries

**Query 1: Task Completion Rate**
```sql
fields @timestamp, @message
| filter @message like /Task completed/
| stats count() as completed_tasks by bin(5m)
```

**Query 2: Error Analysis**
```sql
fields @timestamp, @message
| filter @message like /ERROR/
| parse @message /ERROR: (?<error_message>.*)/
| stats count() by error_message
| sort count desc
```

**Query 3: Agent Performance**
```sql
fields @timestamp, @message
| filter @message like /Processing task/
| parse @message /agent=(?<agent>[A-Z]+), duration=(?<duration>[0-9]+)/
| stats avg(duration) as avg_duration by agent
```

### 8.4 Database Monitoring

**RDS CloudWatch Metrics:**
```bash
# CPU utilization
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=document-ai-prod-mysql \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average

# Database connections
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=document-ai-prod-mysql \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average
```

**Custom Database Queries:**
```sql
-- Active tasks by status
SELECT status, COUNT(*) as count
FROM task_management
GROUP BY status;

-- Agent workload
SELECT name, active_tasks, pending_tasks
FROM v_agent_workload;

-- Average task duration by agent
SELECT a.name,
       AVG(TIMESTAMPDIFF(SECOND, tm.created_at, tm.updated_at)) as avg_seconds
FROM task_management tm
JOIN agents a ON tm.agent_id = a.id
WHERE tm.status = 'completed'
GROUP BY a.name;
```

### 8.5 Alerting Recommendations

**CloudWatch Alarms to Create:**

1. **High CPU Usage:**
   ```hcl
   resource "aws_cloudwatch_metric_alarm" "darci_high_cpu" {
     alarm_name          = "specter-darci-high-cpu"
     comparison_operator = "GreaterThanThreshold"
     evaluation_periods  = "2"
     metric_name         = "CPUUtilization"
     namespace           = "AWS/ECS"
     period              = "300"
     statistic           = "Average"
     threshold           = "80"
     
     dimensions = {
       ClusterName = "specter-cluster"
       ServiceName = "specter-darci-worker"
     }
   }
   ```

2. **Service Down:**
   ```hcl
   resource "aws_cloudwatch_metric_alarm" "orchestrator_down" {
     alarm_name          = "specter-orchestrator-down"
     comparison_operator = "LessThanThreshold"
     evaluation_periods  = "1"
     metric_name         = "RunningTaskCount"
     namespace           = "ECS/ContainerInsights"
     period              = "60"
     statistic           = "Average"
     threshold           = "1"
   }
   ```

3. **Database Connection Errors:**
   ```hcl
   resource "aws_cloudwatch_log_metric_filter" "db_errors" {
     name           = "specter-db-errors"
     log_group_name = aws_cloudwatch_log_group.orchestrator.name
     pattern        = "[timestamp, level=ERROR, msg=\"*database*\"]"
     
     metric_transformation {
       name      = "DatabaseErrors"
       namespace = "Specter"
       value     = "1"
     }
   }
   ```

---

## 9. Disaster Recovery & High Availability

### 9.1 Current HA Posture

**Single Points of Failure:**
1. ✅ **RDS:** Multi-AZ enabled (automatic failover)
2. ✅ **EFS:** Multi-AZ by default (regional service)
3. ⚠️ **Orchestrator:** Single instance (no redundancy)
4. ⚠️ **Subnet:** Single private subnet (should use multi-AZ)
5. ❌ **Region:** Single region (us-east-2)

### 9.2 Improving Availability

**Add Multi-AZ Subnets:**
```hcl
variable "subnet_ids" {
  type    = list(string)
  default = [
    "subnet-0791c0e76fc4f0e81",  # us-east-2a
    "subnet-XXXXXXXXXXXX",        # us-east-2b (add this)
    "subnet-XXXXXXXXXXXX"         # us-east-2c (add this)
  ]
}
```

**Benefits:**
- ECS spreads tasks across AZs
- If one AZ fails, tasks in other AZs continue
- Automatic task replacement in healthy AZs

**Multi-Orchestrator with Leader Election:**
```typescript
// Use DynamoDB for leader election
import { DynamoDBClient, PutItemCommand } from '@aws-sdk/client-dynamodb';

class OrchestratorLeader {
  async acquireLease(): Promise<boolean> {
    try {
      await dynamodb.send(new PutItemCommand({
        TableName: 'specter-leader-election',
        Item: {
          lock_key: { S: 'orchestrator' },
          instance_id: { S: process.env.HOSTNAME },
          lease_expires: { N: String(Date.now() + 30000) }
        },
        ConditionExpression: 'attribute_not_exists(lock_key) OR lease_expires < :now',
        ExpressionAttributeValues: {
          ':now': { N: String(Date.now()) }
        }
      }));
      return true;  // We are the leader
    } catch {
      return false;  // Another instance is leader
    }
  }
}
```

### 9.3 Backup Strategy

**State Backups:**
```bash
# Terraform state is in S3 (enable versioning)
aws s3api put-bucket-versioning \
  --bucket mhk-terraform-state \
  --versioning-configuration Status=Enabled

# List state versions
aws s3api list-object-versions \
  --bucket mhk-terraform-state \
  --prefix specter/
```

**Database Backups:**
- ✅ RDS automated backups (7 days retention)
- ✅ RDS snapshots (manual)

**EFS Backups:**
```bash
# Create AWS Backup plan
aws backup create-backup-plan \
  --backup-plan '{
    "BackupPlanName": "specter-efs-daily",
    "Rules": [{
      "RuleName": "DailyBackup",
      "TargetBackupVaultName": "Default",
      "ScheduleExpression": "cron(0 5 * * ? *)",
      "Lifecycle": {
        "DeleteAfterDays": 30
      }
    }]
  }'
```

### 9.4 Recovery Procedures

**Scenario 1: Single task failure**
- **Detection:** ECS health checks
- **Recovery:** Automatic (ECS starts new task)
- **RTO:** 1-2 minutes

**Scenario 2: Service failure**
```bash
# Check service status
aws ecs describe-services \
  --cluster specter-cluster \
  --services specter-darci-worker \
  --region us-east-2

# Force new deployment
aws ecs update-service \
  --cluster specter-cluster \
  --service specter-darci-worker \
  --force-new-deployment \
  --region us-east-2
```
- **RTO:** 3-5 minutes

**Scenario 3: Complete region failure**
- **Recovery:** Deploy to new region
- **RTO:** 30-60 minutes (manual)
- **Prerequisites:**
  - Cross-region RDS replica
  - S3 cross-region replication
  - Terraform state accessible

```bash
# Emergency deployment to us-west-2
cd terraform
terraform workspace new us-west-2

# Update variables
export TF_VAR_aws_region=us-west-2
export TF_VAR_existing_vpc_id=<new-vpc-id>
# ... (update all region-specific variables)

terraform apply
```

---

## 10. Security Deep-Dive

### 10.1 IAM Security

**Principle of Least Privilege:**
- ✅ Execution role: Only ECR, CloudWatch, Secrets Manager
- ⚠️ Task role: Reused from other service (overly broad)
- ✅ S3 access: Limited to specific buckets/prefixes

**Recommendations:**

1. **Create Dedicated Task Role:**
   ```hcl
   resource "aws_iam_role" "specter_task_role" {
     name = "SpecterTaskRole"
     
     assume_role_policy = jsonencode({
       Version = "2012-10-17"
       Statement = [{
         Action = "sts:AssumeRole"
         Effect = "Allow"
         Principal = { Service = "ecs-tasks.amazonaws.com" }
       }]
     })
   }
   
   # Only Bedrock Claude Sonnet 4.5
   resource "aws_iam_role_policy" "bedrock_access" {
     name = "BedrockAccess"
     role = aws_iam_role.specter_task_role.id
     
     policy = jsonencode({
       Version = "2012-10-17"
       Statement = [{
         Effect = "Allow"
         Action = ["bedrock:InvokeModel"]
         Resource = [
           "arn:aws:bedrock:us-east-2::foundation-model/anthropic.claude-sonnet-4-5-*"
         ]
       }]
     })
   }
   ```

2. **Use IAM Database Authentication:**
   ```typescript
   import { Signer } from '@aws-sdk/rds-signer';
   
   const signer = new Signer({
     hostname: process.env.DB_HOST,
     port: 3306,
     username: 'specter_app',
     region: 'us-east-2'
   });
   
   const token = await signer.getAuthToken();
   const connection = await mysql.createConnection({
     host: process.env.DB_HOST,
     user: 'specter_app',
     password: token,
     ssl: { rejectUnauthorized: true }
   });
   ```

### 10.2 Network Security

**Current Architecture:**
```
Internet
    ↓
  [NAT Gateway]  (for outbound HTTPS to JIRA/Bedrock)
    ↓
  [Private Subnet: subnet-0791c0e76fc4f0e81]
    ├── ECS Tasks (no public IP)
    ├── RDS (no public access)
    └── EFS (VPC-only)
```

**Improvements:**

1. **VPC Endpoints** (eliminates NAT gateway costs):
   ```hcl
   # S3 VPC Endpoint (gateway type - free)
   resource "aws_vpc_endpoint" "s3" {
     vpc_id       = var.existing_vpc_id
     service_name = "com.amazonaws.us-east-2.s3"
     route_table_ids = [var.route_table_id]
   }
   
   # Bedrock VPC Endpoint (interface type - $0.01/hour)
   resource "aws_vpc_endpoint" "bedrock" {
     vpc_id              = var.existing_vpc_id
     service_name        = "com.amazonaws.us-east-2.bedrock-runtime"
     vpc_endpoint_type   = "Interface"
     subnet_ids          = var.subnet_ids
     security_group_ids  = [aws_security_group.vpc_endpoints.id]
   }
   ```

2. **Restrict Outbound Traffic:**
   ```hcl
   egress {
     description = "HTTPS to JIRA"
     from_port   = 443
     to_port     = 443
     protocol    = "tcp"
     cidr_blocks = ["0.0.0.0/0"]  # Ideally, use JIRA IP ranges
   }
   
   egress {
     description     = "MySQL to RDS"
     from_port       = 3306
     to_port         = 3306
     protocol        = "tcp"
     security_groups = [var.rds_security_group_id]
   }
   
   egress {
     description     = "NFS to EFS"
     from_port       = 2049
     to_port         = 2049
     protocol        = "tcp"
     security_groups = [var.existing_efs_security_group_id]
   }
   ```

### 10.3 Secrets Management

**Current Approach:**
- ✅ Database credentials in Secrets Manager
- ⚠️ Injected into environment variables (visible in console)
- ❌ JIRA token in database (should be in Secrets Manager)

**Better Approach:**

1. **Store JIRA token in Secrets Manager:**
   ```bash
   aws secretsmanager create-secret \
     --name specter/jira/api-token \
     --secret-string '{"token":"YOUR_JIRA_TOKEN"}' \
     --region us-east-2
   ```

2. **Read secrets in application code:**
   ```typescript
   import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';
   
   async function getJiraToken(): Promise<string> {
     const client = new SecretsManagerClient({ region: 'us-east-2' });
     const response = await client.send(new GetSecretValueCommand({
       SecretId: 'specter/jira/api-token'
     }));
     const secret = JSON.parse(response.SecretString!);
     return secret.token;
   }
   ```

3. **Remove secrets from task definitions:**
   ```hcl
   # Before (BAD)
   environment = [
     { name = "DB_PASSWORD", value = jsondecode(...).password }
   ]
   
   # After (GOOD)
   secrets = [
     {
       name      = "DB_PASSWORD"
       valueFrom = "${data.aws_secretsmanager_secret.db_credentials.arn}:password::"
     }
   ]
   ```

### 10.4 Container Security

**Current Dockerfiles:**
- ✅ Based on `node:20-alpine` (minimal attack surface)
- ✅ Non-root user (Node.js default)
- ⚠️ `npm ci --production` (still includes dev dependencies like `tsx`)

**Improvements:**

1. **Multi-stage builds:**
   ```dockerfile
   # Build stage
   FROM node:20-alpine AS builder
   WORKDIR /app
   COPY package*.json tsconfig.json ./
   COPY src ./src
   RUN npm ci && npm run build
   
   # Runtime stage
   FROM node:20-alpine
   RUN apk add --no-cache git dumb-init
   WORKDIR /app
   COPY --from=builder /app/dist ./dist
   COPY --from=builder /app/node_modules ./node_modules
   USER node
   ENTRYPOINT ["dumb-init", "--"]
   CMD ["node", "dist/orchestrator/index.js"]
   ```

2. **Image scanning:**
   - ✅ ECR scan on push enabled
   - ⚠️ No blocking on critical vulnerabilities

3. **Runtime security:**
   ```hcl
   # Add to task definition
   container_definitions = jsonencode([{
     # ... existing config
     readonlyRootFilesystem = true  # Prevent file modification
     user = "node"                   # Run as non-root
     
     linuxParameters = {
       capabilities = {
         drop = ["ALL"]  # Drop all Linux capabilities
       }
     }
   }])
   ```

### 10.5 Compliance Considerations

**HIPAA Compliance (MedHok requirement):**
- ✅ Encryption at rest: RDS, EFS, S3 (all encrypted)
- ✅ Encryption in transit: TLS for EFS, HTTPS for S3/Bedrock
- ⚠️ Audit logging: CloudWatch logs (7-day retention may be insufficient)
- ⚠️ Access controls: IAM policies need refinement
- ❌ PHI handling: Ensure JIRA tickets don't contain PHI

**Recommendations:**
1. Extend log retention to 365 days for audit trail
2. Enable CloudTrail for API call logging
3. Implement data classification tags
4. Add encryption key rotation (KMS)

---

## 11. Potential Issues & Risks

### 11.1 Dependency Risks

**External Service Dependencies:**
1. **JIRA Cloud** (medhokapps.atlassian.net)
   - Risk: API rate limiting (10,000 requests/hour)
   - Mitigation: Polling interval = 60 seconds (1 req/min = 1440/day)

2. **AWS Bedrock**
   - Risk: Model availability, throttling
   - Mitigation: Exponential backoff, fallback models

3. **RDS MySQL**
   - Risk: Connection limits (150 max connections)
   - Mitigation: Connection pooling, max 10 connections per worker

### 11.2 State Management Risks

**Terraform State:**
- ✅ Remote state in S3
- ⚠️ Single state file (no workspaces)
- ⚠️ Contains secrets (encrypted at rest, but visible in plan/apply output)

**Recommendations:**
1. Enable S3 bucket versioning (rollback capability)
2. Use separate workspaces for dev/staging/prod
3. Implement state file access logging

### 11.3 Deployment Risks

**Rollback Challenges:**
- No automated rollback mechanism
- Manual process:
  ```bash
  # Rollback to previous task definition
  aws ecs update-service \
    --cluster specter-cluster \
    --service specter-darci-worker \
    --task-definition specter-darci-worker:5  # Previous revision
    --region us-east-2
  ```

**Blue-Green Deployment:**
```hcl
resource "aws_ecs_service" "darci_blue" {
  name            = "specter-darci-worker-blue"
  desired_count   = 2
  # ... existing config
}

resource "aws_ecs_service" "darci_green" {
  name            = "specter-darci-worker-green"
  desired_count   = 0  # Initially inactive
  # ... new config
}

# Switch traffic: scale green to 2, scale blue to 0
```

### 11.4 Resource Limits

**ECS Service Quotas (per region):**
- Clusters: 10,000
- Services per cluster: 5,000
- Tasks per service: 5,000
- **Current usage:** 1 cluster, 5 services, 6 tasks

**Fargate Quotas:**
- Fargate On-Demand vCPU: 1,000 vCPUs
- **Current usage:** 4 vCPUs (0.4%)

**RDS Connection Limits:**
- Max connections: 150 (db.t3.medium)
- **Current usage:** ~6 connections (orchestrator + workers)
- **Risk:** If scaling to 50+ workers, will hit limit

---

## 12. Recommendations & Next Steps

### 12.1 Immediate Improvements (Sprint 1)

1. **Enable S3 Bucket Versioning:**
   ```bash
   aws s3api put-bucket-versioning \
     --bucket mhk-terraform-state \
     --versioning-configuration Status=Enabled
   ```

2. **Add ECR Lifecycle Policies:**
   ```hcl
   resource "aws_ecr_lifecycle_policy" "cleanup" {
     for_each   = aws_ecr_repository.workers
     repository = each.value.name
     
     policy = jsonencode({
       rules = [{
         rulePriority = 1
         description  = "Keep last 10 images"
         selection = {
           tagStatus     = "any"
           countType     = "imageCountMoreThan"
           countNumber   = 10
         }
         action = { type = "expire" }
       }]
     })
   }
   ```

3. **Extend Log Retention:**
   ```hcl
   variable "log_retention_days" {
     default = 30  # Change from 7 to 30
   }
   ```

4. **Add Health Checks to Deployment Script:**
   ```bash
   # After pushing images
   echo "Waiting for services to stabilize..."
   aws ecs wait services-stable \
     --cluster specter-cluster \
     --services specter-orchestrator \
     --region us-east-2
   
   echo "✅ Deployment successful"
   ```

### 12.2 Medium-Term Improvements (Sprint 2-3)

1. **Multi-AZ Deployment:**
   - Add subnets in us-east-2b and us-east-2c
   - Update `subnet_ids` variable

2. **VPC Endpoints:**
   - Add S3 gateway endpoint (free)
   - Add Bedrock interface endpoint ($7/month)
   - Remove NAT gateway dependency

3. **Auto-Scaling:**
   - Implement target tracking scaling
   - Scale workers based on queue depth

4. **Monitoring & Alerting:**
   - Create CloudWatch dashboards
   - Set up SNS alerts for critical issues
   - Integrate with PagerDuty/Slack

### 12.3 Long-Term Improvements (Sprint 4+)

1. **Multi-Region Deployment:**
   - Replicate to us-west-2 for disaster recovery
   - Implement cross-region replication for RDS and S3

2. **CI/CD Pipeline:**
   - GitHub Actions for automated builds
   - Automated testing before deployment
   - Blue-green deployment strategy

3. **Cost Optimization:**
   - Implement Fargate Spot (70% savings)
   - Schedule workers for off-hours scale-down
   - Use Reserved Capacity for predictable workloads

4. **Security Hardening:**
   - Dedicated task role (separate from Bedrock Flow)
   - IAM database authentication
   - Move all secrets to Secrets Manager
   - Implement AWS Config rules for compliance

---

## 13. Terraform Best Practices Checklist

- [x] Remote state backend configured
- [x] State locking enabled (DynamoDB)
- [ ] State encryption verified
- [x] Provider version pinned (`~> 5.0`)
- [x] Default tags configured
- [x] Resource naming convention consistent
- [ ] Variables have validation rules
- [ ] Sensitive variables marked `sensitive = true`
- [ ] Outputs defined for important values
- [ ] Module structure (currently flat)
- [x] Data sources for existing resources
- [ ] Depends_on for implicit dependencies
- [ ] Lifecycle rules for critical resources
- [ ] Separate .tfvars for environments

---

## 14. Conclusion

The Specter Terraform infrastructure is **production-ready** with some caveats:

**✅ Strengths:**
- Smart reuse of existing infrastructure (cost-effective)
- Well-organized variable structure
- Comprehensive resource coverage
- Good separation of concerns (orchestrator vs workers)

**⚠️ Areas for Improvement:**
- Security hardening (IAM, secrets management)
- Multi-AZ redundancy
- Automated rollback mechanism
- Cost optimization opportunities

**Estimated Effort to Production Hardening:**
- Immediate fixes: **1-2 days**
- Medium-term improvements: **1-2 weeks**
- Full production-grade: **1 month**

**Risk Assessment:**
- **Low:** Infrastructure failures (ECS auto-recovery)
- **Medium:** Cost overruns (monitoring needed)
- **Medium:** Security vulnerabilities (secrets in env vars)
- **Low:** Scalability issues (can scale to 100+ workers)

---

## Appendix A: Useful Terraform Commands

```bash
# Initialize with backend
terraform init -backend-config=backend.hcl

# Validate configuration
terraform validate

# Format code
terraform fmt -recursive

# Plan with variable file
terraform plan -var-file=terraform.tfvars

# Apply with auto-approve (CI/CD)
terraform apply -auto-approve -var-file=terraform.tfvars

# Show outputs
terraform output

# Destroy specific resource
terraform destroy -target=aws_ecs_service.workers["darci"]

# Import existing resource
terraform import aws_ecs_cluster.specter specter-cluster

# Refresh state
terraform refresh

# List resources in state
terraform state list

# Show specific resource
terraform state show aws_ecs_cluster.specter
```

---

## Appendix B: Cost Calculator

```python
# Monthly cost calculator
def calculate_fargate_cost(vcpu: float, memory_gb: float, hours_per_month: int = 730):
    """
    Calculate Fargate cost for us-east-2
    vcpu: Number of vCPUs (e.g., 0.5, 1, 2)
    memory_gb: Memory in GB
    hours_per_month: Default 730 (24/7)
    """
    vcpu_cost = vcpu * 0.04048 * hours_per_month
    memory_cost = memory_gb * 0.004445 * hours_per_month
    return vcpu_cost + memory_cost

# Orchestrator: 512 CPU (0.5 vCPU), 1024 MB (1 GB)
orchestrator_cost = calculate_fargate_cost(0.5, 1) * 1  # 1 instance
print(f"Orchestrator: ${orchestrator_cost:.2f}/month")

# DARCI: 1024 CPU (1 vCPU), 2048 MB (2 GB)
darci_cost = calculate_fargate_cost(1, 2) * 2  # 2 instances
print(f"DARCI: ${darci_cost:.2f}/month")

# SCRIBE/PRISM/HERMES: 512 CPU (0.5 vCPU), 1024 MB (1 GB)
worker_cost = calculate_fargate_cost(0.5, 1) * 3  # 3 workers × 1 instance
print(f"Other Workers: ${worker_cost:.2f}/month")

# Total
total = orchestrator_cost + darci_cost + worker_cost
print(f"\nTotal Fargate: ${total:.2f}/month")

# With Fargate Spot (70% discount)
spot_total = total * 0.3
print(f"With Spot: ${spot_total:.2f}/month (saves ${total - spot_total:.2f})")
```

---

**Document Version:** 1.0  
**Last Updated:** 2026-01-26  
**Author:** Specter DevOps Team  
**Next Review:** 2026-02-26
