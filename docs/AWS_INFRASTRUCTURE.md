# AWS Infrastructure Setup

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              AWS VPC                                         │
│                                                                             │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │                         Private Subnets                               │  │
│  │                                                                       │  │
│  │    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐            │  │
│  │    │     ECS     │    │     RDS     │    │     EFS     │            │  │
│  │    │   Cluster   │    │   (MySQL)   │    │  (Storage)  │            │  │
│  │    │             │    │             │    │             │            │  │
│  │    │ ┌─────────┐ │    │             │    │ /mnt/efs/   │            │  │
│  │    │ │Orchstr  │ │    │             │    │ ├─ repos/   │            │  │
│  │    │ │DARCI    │◄├────┤             │    │ └─ docs/    │            │  │
│  │    │ │SCRIBE   │ │    │             │    │             │            │  │
│  │    │ │PRISM    │ │    │             │    │             │            │  │
│  │    │ │HERMES   │ │    │             │    │             │            │  │
│  │    │ └────┬────┘ │    │             │    │      ▲      │            │  │
│  │    │      │      │    │             │    │      │      │            │  │
│  │    │      └──────┼────┼─────────────┼────┼──────┘      │            │  │
│  │    │    mount    │    │             │    │   mount     │            │  │
│  │    └─────────────┘    └─────────────┘    └─────────────┘            │  │
│  │                              │                  │                   │  │
│  └──────────────────────────────┼──────────────────┼───────────────────┘  │
│                                 │                  │                      │
│  ┌──────────────────────────────┼──────────────────┼───────────────────┐  │
│  │                       Public Subnet             │                   │  │
│  │                                                 │                   │  │
│  │    ┌─────────────────────────────────────────────────────────────┐  │  │
│  │    │                     BASTION HOST                            │  │  │
│  │    │                                                             │  │  │
│  │    │  ┌─────────────┐     ┌──────────────────────────────────┐  │  │  │
│  │    │  │ SSH Access  │     │         EFS Mount                │  │  │  │
│  │    │  │ (port 22)   │     │                                  │  │  │  │
│  │    │  │             │     │  /mnt/efs/                       │  │  │  │
│  │    │  │ • mysql CLI │     │  ├── repos/                      │  │  │  │
│  │    │  │ • Browse EFS│     │  │   ├── project-a/              │  │  │  │
│  │    │  │ • VS Code   │     │  │   └── project-b/              │  │  │  │
│  │    │  │   Remote    │     │  └── documents/                  │  │  │  │
│  │    │  └─────────────┘     │      ├── runbooks/               │  │  │  │
│  │    │                      │      └── confluence/             │  │  │  │
│  │    │                      └──────────────────────────────────┘  │  │  │
│  │    │                                                             │  │  │
│  │    └─────────────────────────────────────────────────────────────┘  │  │
│  │                                                                      │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## 1. Create EFS File System

```bash
# Create EFS file system
aws efs create-file-system \
    --performance-mode generalPurpose \
    --throughput-mode bursting \
    --encrypted \
    --tags Key=Name,Value=specter-efs \
    --region us-east-1

# Note the FileSystemId (e.g., fs-12345678)
```

## 2. Create Security Group for EFS

```bash
# Create security group for EFS
aws ec2 create-security-group \
    --group-name specter-efs-sg \
    --description "Security group for Specter EFS" \
    --vpc-id vpc-XXXXXXXX

# Allow NFS (port 2049) from:
# - ECS tasks security group
# - Bastion security group

aws ec2 authorize-security-group-ingress \
    --group-id sg-efs-XXXXXXXX \
    --protocol tcp \
    --port 2049 \
    --source-group sg-ecs-XXXXXXXX

aws ec2 authorize-security-group-ingress \
    --group-id sg-efs-XXXXXXXX \
    --protocol tcp \
    --port 2049 \
    --source-group sg-bastion-XXXXXXXX
```

## 3. Create EFS Mount Targets

Create mount targets in each subnet where ECS or bastion runs:

```bash
# Private subnet (for ECS)
aws efs create-mount-target \
    --file-system-id fs-12345678 \
    --subnet-id subnet-private-XXXXXXXX \
    --security-groups sg-efs-XXXXXXXX

# Public subnet (for Bastion) - if bastion is in public subnet
aws efs create-mount-target \
    --file-system-id fs-12345678 \
    --subnet-id subnet-public-XXXXXXXX \
    --security-groups sg-efs-XXXXXXXX
```

## 4. Mount EFS on Bastion Host

SSH into your bastion and run:

```bash
# Install NFS client (Amazon Linux 2)
sudo yum install -y amazon-efs-utils

# Or for Ubuntu
sudo apt-get install -y nfs-common amazon-efs-utils

# Create mount point
sudo mkdir -p /mnt/efs

# Mount EFS (replace fs-12345678 with your EFS ID)
sudo mount -t efs -o tls fs-12345678:/ /mnt/efs

# Verify mount
df -h /mnt/efs
ls -la /mnt/efs
```

## 5. Make Mount Persistent (Bastion)

Add to `/etc/fstab`:

```bash
# Add this line to /etc/fstab
fs-12345678:/ /mnt/efs efs _netdev,tls 0 0

# Or using DNS name
fs-12345678.efs.us-east-1.amazonaws.com:/ /mnt/efs nfs4 nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport,_netdev 0 0
```

## 6. Create Directory Structure on EFS

```bash
# SSH to bastion, then:
sudo mkdir -p /mnt/efs/repos
sudo mkdir -p /mnt/efs/documents/runbooks
sudo mkdir -p /mnt/efs/documents/confluence-exports

# Set permissions (optional - for ECS tasks)
sudo chmod 755 /mnt/efs
sudo chmod 755 /mnt/efs/repos
sudo chmod 755 /mnt/efs/documents
```

## 7. ECS Task Definition with EFS

```json
{
  "family": "specter-darci",
  "taskRoleArn": "arn:aws:iam::ACCOUNT:role/specter-task-role",
  "executionRoleArn": "arn:aws:iam::ACCOUNT:role/specter-execution-role",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "1024",
  "memory": "2048",
  "volumes": [
    {
      "name": "specter-efs",
      "efsVolumeConfiguration": {
        "fileSystemId": "fs-12345678",
        "rootDirectory": "/",
        "transitEncryption": "ENABLED"
      }
    }
  ],
  "containerDefinitions": [
    {
      "name": "darci-worker",
      "image": "YOUR_ECR_REPO:latest",
      "command": ["node", "dist/orchestrator/workers/index.js", "DARCI"],
      "essential": true,
      "mountPoints": [
        {
          "sourceVolume": "specter-efs",
          "containerPath": "/mnt/efs",
          "readOnly": false
        }
      ],
      "environment": [
        { "name": "EFS_MOUNT_PATH", "value": "/mnt/efs" },
        { "name": "REPOSITORY_PATH", "value": "/mnt/efs/repos" },
        { "name": "DOCUMENTS_PATH", "value": "/mnt/efs/documents" }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/specter-darci",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "darci"
        }
      }
    }
  ]
}
```

## 8. Using VS Code Remote SSH

You can also use VS Code Remote SSH to browse EFS visually:

1. Install "Remote - SSH" extension in VS Code
2. Connect to bastion: `ssh -i your-key.pem ec2-user@bastion-ip`
3. Open folder: `/mnt/efs`
4. Browse repos and documents visually!

## Summary

| Component | Access to EFS |
|-----------|---------------|
| Bastion Host | ✅ NFS mount `/mnt/efs` |
| ECS - Orchestrator | ❌ Not needed |
| ECS - DARCI Worker | ✅ EFS volume mount |
| ECS - SCRIBE Worker | ✅ EFS volume mount |
| ECS - PRISM Worker | ✅ EFS volume mount |
| ECS - HERMES Worker | ✅ EFS volume mount |

All agents and your bastion see the **exact same files** in real-time!
