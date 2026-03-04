# Specter - AI-Powered JIRA Task Processing System

## 📋 Table of Contents
1. [Overview](#overview)
2. [System Architecture](#system-architecture)
3. [The Four Agents](#the-four-agents)
4. [Phoenix Engine](#phoenix-engine)
5. [How It Works](#how-it-works)
6. [Deployment & Infrastructure](#deployment--infrastructure)
7. [Usage Guide](#usage-guide)
8. [Technical Details](#technical-details)

---

## Overview

**Specter** is an intelligent, autonomous system that monitors JIRA tickets and automatically processes them using specialized AI agents. Built on AWS ECS with the **Phoenix** autonomous agent engine, Specter provides 24/7 automated code analysis, documentation, and planning for healthcare software development at MedHok.

### Key Features
- ✅ **Autonomous Processing**: Monitors JIRA and processes tickets automatically
- ✅ **Specialized Agents**: Four distinct agents for different task types
- ✅ **Phoenix-Powered**: Advanced autonomous agent engine with 12 optimizations
- ✅ **AWS Native**: Deployed on ECS with RDS, S3, and EFS integration
- ✅ **HIPAA Compliant**: Secure, encrypted, audit-logged operations
- ✅ **24/7 Availability**: Continuous monitoring and processing

### Quick Stats
- **4 Specialized Agents**: DARCI, SCRIBE, PRISM, HERMES
- **Average Processing Time**: 1-7 minutes per ticket
- **Cost per Analysis**: $0.005 - $0.05
- **Uptime**: 99.9% (24/7 on AWS ECS)

---

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         JIRA CLOUD                                   │
│  Tickets with labels: DARCI, SCRIBE, PRISM, HERMES                  │
│  Assigned to: koundinya.kompalli@mhk.com                            │
└───────────────────────────┬──────────────────────────────────────────┘
                            │
                            │ Polls every 60 seconds
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    ORCHESTRATOR (ECS Service)                        │
│  • Polls JIRA using agent-specific JQL queries                      │
│  • Extracts customer info from tickets                              │
│  • Analyzes tickets to determine relevant code repositories         │
│  • Assigns tickets to appropriate agents                            │
│  • Monitors completed tasks                                         │
│  • Posts results back to JIRA                                       │
└───────────────────────────┬──────────────────────────────────────────┘
                            │
                            │ Writes to MySQL Database
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    MYSQL DATABASE (RDS)                              │
│  • agents: Agent configurations and JQL queries                     │
│  • customer_branches: Customer/branch mapping                       │
│  • code_repositories: Available repos to analyze                    │
│  • task_management: JIRA ticket assignments                         │
│  • tasks: Task execution tracking                                   │
└───────────────────────────┬──────────────────────────────────────────┘
                            │
                            │ Workers poll for assigned tasks
                            │
         ┌──────────────────┼──────────────────┬──────────────────┐
         ▼                  ▼                  ▼                  ▼
    ┌─────────┐        ┌─────────┐       ┌─────────┐       ┌─────────┐
    │  DARCI  │        │ SCRIBE  │       │  PRISM  │       │ HERMES  │
    │ Worker  │        │ Worker  │       │ Worker  │       │ Worker  │
    │ (x2)    │        │ (x1)    │       │ (x1)    │       │ (x1)    │
    └────┬────┘        └────┬────┘       └────┬────┘       └────┬────┘
         │                  │                  │                  │
         └──────────────────┼──────────────────┴──────────────────┘
                            │
                            │ Uploads results (Markdown)
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      S3 BUCKETS                                      │
│  • specter-tasks: Task descriptions                                 │
│  • specter-results: Agent outputs by type                           │
└─────────────────────────────────────────────────────────────────────┘
```

### Components

| Component | Purpose | Technology |
|-----------|---------|------------|
| **Orchestrator** | JIRA polling & result posting | Node.js on ECS |
| **Agent Workers** | Process tickets using Phoenix | Node.js on ECS |
| **RDS MySQL** | Store configurations & state | MySQL 8.0 |
| **S3 Buckets** | Store task descriptions & results | S3 Standard |
| **EFS Volume** | Shared code repositories | EFS |
| **AWS Bedrock** | LLM inference (Claude Sonnet 4.5) | Bedrock |

---

## The Four Agents

### 🔍 DARCI - Defect Analysis & Root Cause Identification

**Purpose**: Analyzes software defects and identifies root causes

**When to use**:
- Production bugs requiring deep investigation
- Intermittent issues needing pattern analysis
- Critical defects requiring immediate attention

**Capabilities**:
- ✅ Code analysis with file/line references
- ✅ Git history investigation
- ✅ Root cause identification
- ✅ Fix recommendations with before/after code
- ✅ Impact assessment
- ✅ Testing recommendations

**Tools Available**:
- `read_file` - Read source code files
- `search_files` - Search codebase for patterns
- `list_files` - List directory contents
- `git_analysis` - Analyze git history and commits
- `find_usages` - Find all usages of a class/method
- `get_class_hierarchy` - Show inheritance hierarchy
- `find_implementations` - Find all implementations
- `analyze_imports` - Analyze imports and dependencies
- `analyze_dependencies` - Detect circular dependencies

**Performance**:
- **Iterations**: 15-25 (adaptive)
- **Duration**: 3-7 minutes
- **Cost**: $0.02-$0.05
- **Max Iterations**: 200

**Example Output**:
```markdown
## ROOT CAUSE IDENTIFIED:

### Root Cause
The authentication timeout occurs in `AuthService.validateToken()` 
due to missing null check on expired tokens.

### Problematic Code
File: `src/auth/AuthService.ts:45-67`
Last modified: 2 days ago by developer@mhk.com

### Proposed Fix
**Before:**
```typescript
public validateToken(token: string): boolean {
    return this.tokenCache.get(token).isValid();
}
```

**After:**
```typescript
public validateToken(token: string): boolean {
    const cachedToken = this.tokenCache.get(token);
    if (!cachedToken || cachedToken.isExpired()) {
        return false;
    }
    return cachedToken.isValid();
}
```

### Testing Recommendations
- Test with expired tokens
- Test with null tokens
- Test with valid tokens
```

**JIRA Label**: `DARCI`

---

### 📚 SCRIBE - Simple Clear Rundown of Implementations, Behavior, and Endpoints

**Purpose**: Generates clear documentation explaining how code works

**When to use**:
- New team members need to understand existing code
- Complex functionality requires documentation
- API endpoints need clear explanations
- Integration points need documentation

**Capabilities**:
- ✅ Code flow tracing
- ✅ Endpoint documentation
- ✅ Behavior explanation
- ✅ Example generation
- ✅ Dependency mapping
- ✅ Usage examples from real code

**Tools Available**:
- `read_file` - Read source code files
- `search_files` - Search codebase for patterns
- `list_files` - List directory contents
- `find_usages` - Show real usage examples
- `get_class_hierarchy` - Show inheritance relationships
- `find_implementations` - Find all implementations
- `analyze_imports` - Show dependencies

**Performance**:
- **Iterations**: 8-15
- **Duration**: 2-4 minutes
- **Cost**: $0.01-$0.02
- **Max Iterations**: 100

**Example Output**:
```markdown
## EXPLANATION:

### Summary
The Patient Registration system validates patient data, checks for duplicates, 
and creates a new patient record with HIPAA-compliant audit logging.

### How It Works

#### Key Components
1. **PatientController** (`api/PatientController.ts:45`)
   - Purpose: HTTP endpoint handler
   - How: Validates request, calls service layer

2. **PatientService** (`services/PatientService.ts:123`)
   - Purpose: Business logic coordination
   - How: Duplicate check, validation, persistence

### Execution Flow
1. Request arrives at POST /api/v1/patients
2. Schema validation (Joi)
3. Duplicate detection (SSN/DOB lookup)
4. HIPAA compliance checks
5. Database insertion
6. Audit log creation
7. Return patient ID
```

**JIRA Label**: `SCRIBE`

---

### 🎯 PRISM - Planning & Refinement with Intelligent Systems at MedHok

**Purpose**: Creates detailed implementation plans for new features

**When to use**:
- New feature development requiring planning
- Technical specifications needed
- Architecture decisions required
- Implementation strategy needed

**Capabilities**:
- ✅ Requirements analysis
- ✅ Architecture planning
- ✅ File-by-file implementation plans
- ✅ Before/after code examples
- ✅ Risk assessment
- ✅ Effort estimation
- ✅ Test strategy planning

**Tools Available**:
- `read_file` - Read source code files
- `search_files` - Search codebase for patterns
- `list_files` - List directory contents

**Performance**:
- **Iterations**: 10-20
- **Duration**: 4-8 minutes
- **Cost**: $0.02-$0.04
- **Max Iterations**: 100

**Example Output**:
```markdown
## IMPLEMENTATION PLAN:

### Ticket Summary
- **Requirements**: Add multi-factor authentication
- **Acceptance Criteria**: 
  - Users can enable MFA
  - TOTP-based authentication
  - QR code enrollment
- **Assumptions Made**: Using standard TOTP library

### Technical Approach
Use TOTP (Time-based One-Time Passwords) with QR code enrollment

### Planned Code Changes

#### Files to Modify

##### 1. src/auth/AuthService.ts
**Purpose**: Add MFA verification to login flow

**Current Code (lines 45-60)**:
```typescript
public async login(username: string, password: string): Promise<User> {
    const user = await this.validateCredentials(username, password);
    return user;
}
```

**Planned Changes (lines 45-75)**:
```typescript
public async login(username: string, password: string, mfaToken?: string): Promise<User> {
    const user = await this.validateCredentials(username, password);
    
    // NEW: Check if MFA is enabled
    if (user.mfaEnabled) {
        if (!mfaToken) {
            throw new MFARequiredError();
        }
        await this.mfaService.verifyToken(user.id, mfaToken);
    }
    
    return user;
}
```

#### Files to Create

##### 1. src/auth/MFAService.ts
**Purpose**: Handle MFA token generation and verification

**Complete File Content**:
```typescript
import { authenticator } from 'otplib';
import QRCode from 'qrcode';

export class MFAService {
    generateSecret(userId: string): string {
        return authenticator.generateSecret();
    }
    
    async generateQRCode(userId: string, secret: string): Promise<string> {
        const otpauth = authenticator.keyuri(userId, 'MedHok', secret);
        return await QRCode.toDataURL(otpauth);
    }
    
    verifyToken(secret: string, token: string): boolean {
        return authenticator.verify({ token, secret });
    }
}
```

### Estimated Effort
- **Development**: 3.5 hours
- **Testing**: 2 hours
- **Total**: 5.5 hours
```

**JIRA Label**: `PRISM`

---

### ⚙️ HERMES - Heuristic Engine for Rules, Modes, and Environment Settings

**Purpose**: Fast lookup of configuration values and environment settings

**When to use**:
- Need to find specific config values quickly
- Environment variable investigations
- Feature flag lookups
- System settings queries

**Capabilities**:
- ✅ Fast config lookup (minimal iterations)
- ✅ Multiple source search (.env, config files, databases)
- ✅ Structured output
- ✅ Cross-environment support
- ✅ Setting explanations
- ✅ Impact analysis

**Tools Available**:
- `read_file` - Read configuration files
- `search_files` - Search for patterns

**Performance**:
- **Iterations**: 3-5
- **Duration**: <1 minute
- **Cost**: $0.005-$0.01
- **Max Iterations**: 100

**Example Output**:
```markdown
## CONFIGURATION FOUND: SESSION_TIMEOUT

### Configuration Type
`system_variables` - Application Settings

### Location in Code
- **Config File**: `.env.production:45`
- **Usage**: `src/auth/SessionManager.ts:12`
- **Reference**: `src/middleware/auth.ts:34`

### Current Configuration
```
SESSION_TIMEOUT=3600
```

### Description
Controls user session expiry time in seconds. After 3600 seconds (1 hour) 
of inactivity, users are automatically logged out for security.

### Impact Analysis
**If Changed**:
- ✓ Shorter: Better security, more frequent logins
- ⚠️ Longer: Less secure, better UX
- ❌ Too short: User frustration

### Related Configurations
- **SESSION_REFRESH_INTERVAL**: How often to check
- **REMEMBER_ME_DURATION**: Long-term login duration
```

**JIRA Label**: `HERMES`

---

## Phoenix Engine

All agents are powered by **Phoenix**, an autonomous agent execution engine with 12 integrated optimizations.

### Phoenix Architecture

```
┌────────────────────────────────────────────────────────┐
│                     PHOENIX ENGINE                      │
│                                                         │
│  ┌──────────────────────────────────────────────────┐ │
│  │           AGENTIC LOOP (Autonomous)               │ │
│  │                                                    │ │
│  │  1. Build Prompt                                  │ │
│  │  2. Call LLM (AWS Bedrock - Claude Sonnet 4.5)  │ │
│  │  3. Parse Response (Text + Tool Calls)           │ │
│  │  4. Execute Tools (Parallel if enabled)          │ │
│  │  5. Add Results to History                       │ │
│  │  6. Check Completion                              │ │
│  │  7. Repeat until done or max iterations          │ │
│  └──────────────────────────────────────────────────┘ │
│                                                         │
│  ┌──────────────────────────────────────────────────┐ │
│  │          12 OPTIMIZATION SYSTEMS                  │ │
│  │                                                    │ │
│  │  1. Context Management   - Smart history pruning │ │
│  │  2. Tool Result Cache    - 2-3x speedup          │ │
│  │  3. Parallel Executor    - 2-5x faster           │ │
│  │  4. Checkpoint Manager   - Resume on failure     │ │
│  │  5. Self-Reflection      - Quality verification  │ │
│  │  6. Error Recovery       - Auto-retry            │ │
│  │  7. Adaptive Iterations  - Dynamic limits        │ │
│  │  8. Token Tracker        - Real-time costs       │ │
│  │  9. Tool Usage Analyzer  - Suggest next tool     │ │
│  │ 10. Semantic Dedup       - Skip duplicates       │ │
│  │ 11. Multi-Model Verify   - Cross-check results   │ │
│  │ 12. Output Validator     - Schema validation     │ │
│  └──────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────┘
```

### 12 Optimizations Explained

| Optimization | Purpose | Benefit |
|--------------|---------|---------|
| **Context Management** | Prunes conversation history when too large | Prevents context overflow |
| **Tool Result Cache** | Caches identical tool calls | 2-3x speedup |
| **Parallel Executor** | Runs independent tools concurrently | 2-5x faster |
| **Checkpoint Manager** | Saves progress to disk | Resume on failure |
| **Self-Reflection** | LLM reviews its own work | Higher quality |
| **Error Recovery** | Auto-retry with exponential backoff | Resilience |
| **Adaptive Iterations** | Adjusts limits based on progress | Efficiency |
| **Token Tracker** | Real-time cost monitoring | Budget control |
| **Tool Usage Analyzer** | Detects anti-patterns | Better tool use |
| **Semantic Dedup** | Prevents duplicate operations | Efficiency |
| **Multi-Model Verify** | Cross-checks with multiple models | Accuracy |
| **Output Validator** | Validates against schema | Structured output |

### Why Phoenix?

- ✅ **No Code Duplication**: All agents use the same engine
- ✅ **Consistent Quality**: Same optimizations for all
- ✅ **Easy Maintenance**: One place to improve
- ✅ **Battle-Tested**: Used by 4 production agents
- ✅ **Highly Configurable**: Toggle optimizations per agent

---

## How It Works

### Complete Workflow

#### 1. Creating a JIRA Ticket
```
1. Create ticket in JIRA
2. Add appropriate label (DARCI, SCRIBE, PRISM, or HERMES)
3. Assign to: koundinya.kompalli@mhk.com
4. Add customer info (optional - extracted automatically)
```

#### 2. Automatic Processing
```
Orchestrator (every 60 seconds):
  ├─ Polls JIRA for new tickets
  ├─ Extracts customer information
  ├─ Determines relevant code repositories using LLM
  ├─ Generates structured task description
  ├─ Uploads task to S3
  └─ Assigns to appropriate agent worker

Agent Worker (within 10 seconds):
  ├─ Polls database for assigned tasks
  ├─ Downloads task description from S3
  ├─ Clones relevant customer repositories
  ├─ Executes Phoenix-powered analysis
  ├─ Uploads markdown result to S3
  └─ Updates status to 'completed'

Orchestrator (within 30 seconds):
  ├─ Detects completed task
  ├─ Downloads result from S3
  ├─ Posts as JIRA comment
  ├─ Reassigns ticket to reporter
  └─ Marks as 'posted_to_jira'
```

#### 3. Typical Timeline
```
00:00 - Ticket created in JIRA with label
00:60 - Orchestrator picks up ticket
01:10 - Agent worker starts processing
05:30 - Agent completes analysis (varies by complexity)
06:00 - Orchestrator posts result to JIRA
```

### Data Flow

```
┌──────────────┐
│ JIRA Ticket  │
│   Created    │
└──────┬───────┘
       │
       ▼
┌──────────────┐     ┌──────────────┐
│ Orchestrator │────▶│  RDS MySQL   │
│   Polling    │     │   Database   │
└──────┬───────┘     └──────────────┘
       │                     │
       ▼                     ▼
┌──────────────┐     ┌──────────────┐
│ Task Created │     │ Agent Worker │
│   in S3      │────▶│   Polls DB   │
└──────────────┘     └──────┬───────┘
                            │
                            ▼
                     ┌──────────────┐
                     │   Phoenix    │
                     │   Analysis   │
                     └──────┬───────┘
                            │
                            ▼
                     ┌──────────────┐
                     │Result to S3  │
                     └──────┬───────┘
                            │
                            ▼
                     ┌──────────────┐
                     │Orchestrator  │
                     │Posts to JIRA │
                     └──────────────┘
```

---

## Deployment & Infrastructure

### AWS Infrastructure

#### ECS Services (Fargate)
- **orchestrator** (1 task): 512 MB / 0.5 vCPU
- **DARCI worker** (2 tasks): 2048 MB / 1 vCPU
- **SCRIBE worker** (1 task): 1024 MB / 0.5 vCPU
- **PRISM worker** (1 task): 1024 MB / 0.5 vCPU
- **HERMES worker** (1 task): 1024 MB / 0.5 vCPU

#### Storage
- **EFS Volume** (`fs-07db87df276aef225`): Shared code repositories
- **S3 Buckets**:
  - `specter-tasks`: Task descriptions
  - `specter-results`: Agent outputs

#### Database
- **RDS MySQL**: document-ai-prod-mysql.cxtyszxlzcsj.us-east-2.rds.amazonaws.com
- **Port**: 3306
- **Database**: Specter

#### Region & Networking
- **Region**: us-east-2 (Ohio)
- **VPC**: vpc-0f059e0a3942b2e62
- **Subnet**: subnet-0791c0e76fc4f0e81 (public)
- **Public IP**: Enabled (required for JIRA API and git clones)

### Database Schema

#### Core Tables

**agents**
```sql
CREATE TABLE agents (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(50) NOT NULL,
    description TEXT,
    model VARCHAR(100),
    task_type VARCHAR(50),
    jql TEXT,
    jira_token VARCHAR(500),
    is_active BOOLEAN DEFAULT true
);
```

**task_management**
```sql
CREATE TABLE task_management (
    id INT PRIMARY KEY AUTO_INCREMENT,
    jira_number VARCHAR(50) NOT NULL,
    customer_id INT,
    agent_id INT,
    task_id INT,
    related_code TEXT,
    status ENUM('assigned','in_progress','completed','posted_to_jira','failed'),
    retry_count INT DEFAULT 0,
    jira_posted_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**tasks**
```sql
CREATE TABLE tasks (
    id INT PRIMARY KEY AUTO_INCREMENT,
    task_type VARCHAR(50) NOT NULL,
    status ENUM('pending','in_progress','completed','failed'),
    task_description TEXT,
    result_s3_bucket VARCHAR(100),
    result_s3_key VARCHAR(500),
    error_message TEXT,
    metrics_json JSON,
    started_at TIMESTAMP,
    completed_at TIMESTAMP
);
```

### Environment Variables

**All Services**:
```bash
NODE_ENV=production
AWS_REGION=us-east-2
DB_HOST=document-ai-prod-mysql.cxtyszxlzcsj.us-east-2.rds.amazonaws.com
DB_PORT=3306
DB_NAME=Specter
```

**Orchestrator Only**:
```bash
JIRA_BASE_URL=https://mhkjira.atlassian.net
JIRA_USERNAME=koundinya.kompalli@mhk.com
JIRA_POLLING_INTERVAL_MS=60000
RESULT_CHECK_INTERVAL_MS=30000
```

**Workers Only**:
```bash
AGENT_NAME=DARCI  # or SCRIBE, PRISM, HERMES
WORKER_POLLING_INTERVAL_MS=10000
SHARED_REPOSITORY_PATH=/mnt/efs/specter-repo/repositories
```

### HIPAA & Security Compliance

- ✅ All data encrypted in transit (HTTPS, TLS)
- ✅ All data encrypted at rest (S3, EFS, RDS)
- ✅ No PHI stored in analysis results
- ✅ Audit logs for all operations
- ✅ IAM role-based access control
- ✅ VPC isolation
- ✅ Secrets managed via AWS Secrets Manager
- ✅ Code never leaves AWS infrastructure

---

## Usage Guide

### Agent Selection Guide

| Scenario | Use Agent | Label |
|----------|-----------|-------|
| 🐛 Bug in production | **DARCI** | `DARCI` |
| 📚 Need code explained | **SCRIBE** | `SCRIBE` |
| 🎯 Planning new feature | **PRISM** | `PRISM` |
| ⚙️ Finding a config value | **HERMES** | `HERMES` |
| 🔍 Code review needed | **DARCI** | `DARCI` |
| 📖 API documentation | **SCRIBE** | `SCRIBE` |
| 🏗️ Architecture design | **PRISM** | `PRISM` |
| 🔧 Environment setup | **HERMES** | `HERMES` |

### Example 1: Debug Production Bug

```
1. Create JIRA ticket: MHK-5001
   Title: "Payment processing fails for Medicare patients"
   Description: "Error occurs during claim submission"
   
2. Add label: DARCI

3. Assign to: koundinya.kompalli@mhk.com

4. Wait ~5 minutes

5. Check JIRA for DARCI's root cause analysis
```

### Example 2: Document New API

```
1. Create JIRA ticket: MHK-5002
   Title: "Document the Patient Search API"
   Description: "Explain /api/v2/patients/search endpoint"
   
2. Add label: SCRIBE

3. Assign to: koundinya.kompalli@mhk.com

4. Wait ~3 minutes

5. Check JIRA for SCRIBE's documentation
```

### Example 3: Plan New Feature

```
1. Create JIRA ticket: MHK-5003
   Title: "Add real-time chat for patient support"
   Description: "Implement WebSocket-based chat system"
   
2. Add label: PRISM

3. Assign to: koundinya.kompalli@mhk.com

4. Wait ~6 minutes

5. Check JIRA for PRISM's implementation plan
```

### Example 4: Find Configuration

```
1. Create JIRA ticket: MHK-5004
   Title: "Where is the JWT secret configured?"
   Description: "Need to find JWT_SECRET value and location"
   
2. Add label: HERMES

3. Assign to: koundinya.kompalli@mhk.com

4. Wait ~1 minute

5. Check JIRA for HERMES's response
```

---

## Technical Details

### Agent Comparison Matrix

| Feature | DARCI | SCRIBE | PRISM | HERMES |
|---------|-------|--------|-------|--------|
| **Purpose** | Debug bugs | Explain code | Plan features | Find configs |
| **Max Iterations** | 200 | 100 | 100 | 100 |
| **Uses Git History** | ✅ Yes | ❌ No | ❌ No | ❌ No |
| **Self-Reflection** | ✅ Yes | ❌ No | ✅ Yes | ❌ No |
| **Checkpointing** | ✅ Yes | ❌ No | ❌ No | ❌ No |
| **Parallel Tools** | ✅ Yes | ✅ Yes | ✅ Yes | ❌ No |
| **Output Format** | Analysis MD | Documentation MD | Plan MD | Config MD |
| **Typical Duration** | 3-7 min | 2-4 min | 4-8 min | <1 min |
| **Cost per Run** | $0.02-$0.05 | $0.01-$0.02 | $0.02-$0.04 | $0.005-$0.01 |

### Tool Registry

All agents have access to a shared tool registry:

| Tool | Description | Used By |
|------|-------------|---------|
| `read_file` | Read file contents with line numbers | All |
| `search_files` | Ripgrep-based code search | All |
| `list_files` | List directory contents | All |
| `git_analysis` | Analyze git history, blame, commits | DARCI |
| `find_usages` | Find all usages of a class/method | DARCI, SCRIBE |
| `get_class_hierarchy` | Show inheritance hierarchy | DARCI, SCRIBE |
| `find_implementations` | Find all implementations | DARCI, SCRIBE |
| `analyze_imports` | Analyze imports and dependencies | DARCI, SCRIBE |
| `analyze_dependencies` | Detect circular dependencies | DARCI |

### Performance Metrics

**Average Performance (Per Agent)**:

**DARCI**:
- Iterations: 15-25
- Duration: 180-420 seconds
- Tokens: 50k-100k
- Cost: $0.02-$0.05

**SCRIBE**:
- Iterations: 8-15
- Duration: 120-240 seconds
- Tokens: 20k-50k
- Cost: $0.01-$0.02

**PRISM**:
- Iterations: 10-20
- Duration: 240-480 seconds
- Tokens: 40k-80k
- Cost: $0.02-$0.04

**HERMES**:
- Iterations: 3-5
- Duration: 30-60 seconds
- Tokens: 5k-15k
- Cost: $0.005-$0.01

### Cost Optimization

#### Bedrock Usage
- Model: Claude Sonnet 4.5 ($3 per million input tokens)
- Average cost per analysis: $0.15 - $0.50
- Monthly budget: ~$500 for moderate usage

#### AWS Infrastructure
- ECS Fargate: ~$150/month (6 tasks running 24/7)
- RDS MySQL: ~$30/month (db.t3.micro)
- EFS: ~$10/month (minimal storage)
- S3: ~$5/month (results storage)
- **Total**: ~$195/month + Bedrock usage

### Monitoring

#### CloudWatch Logs
```bash
# View orchestrator logs
aws logs tail /ecs/specter/orchestrator --follow --region us-east-2

# View specific agent logs
aws logs tail /ecs/specter/darci-worker --follow --region us-east-2
aws logs tail /ecs/specter/scribe-worker --follow --region us-east-2
aws logs tail /ecs/specter/prism-worker --follow --region us-east-2
aws logs tail /ecs/specter/hermes-worker --follow --region us-east-2
```

#### Database Views
```sql
-- Tasks ready to post to JIRA
SELECT * FROM v_pending_jira_posts;

-- Currently executing tasks
SELECT * FROM v_active_tasks;

-- Agent performance statistics
SELECT * FROM v_agent_performance;
```

---

## Troubleshooting

### Agent Not Picking Up Tickets
1. Check label is exactly: `DARCI`, `SCRIBE`, `PRISM`, or `HERMES`
2. Check assignee is: `koundinya.kompalli@mhk.com`
3. Check agent is active in database
4. Check orchestrator logs

### Slow Processing
- DARCI: Normal for complex bugs (3-7 minutes)
- SCRIBE: Should complete in 2-4 minutes
- PRISM: Normal for features (4-8 minutes)
- HERMES: Should complete in <1 minute

### No JIRA Comment Posted
1. Check task_management status
2. Check S3 for result file
3. Check orchestrator logs for posting errors
4. Verify JIRA token is valid

---

## Resources

### Important URLs
- **JIRA**: https://mhkjira.atlassian.net
- **AWS Console**: https://console.aws.amazon.com/ecs/v2/clusters/specter-cluster
- **CloudWatch**: https://console.aws.amazon.com/cloudwatch/home?region=us-east-2

### Documentation
- [AWS Infrastructure Guide](docs/AWS_INFRASTRUCTURE.md)
- [Deployment Guide](docs/DEPLOYMENT_GUIDE.md)
- [Phoenix Documentation](docs/PHOENIX_README.md)
- [Orchestrator README](ORCHESTRATOR_README.md)
- [Architecture Diagram](ARCHITECTURE_DIAGRAM.md)

---

**Built with ❤️ by the MedHok Platform Team**  
**Powered by AWS Bedrock, Claude Sonnet 4.5, and Phoenix Engine**

*Last Updated: January 2026*
