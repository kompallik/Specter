# DARCI Debugging Guide - CloudWatch Logs

This guide shows you how to monitor and debug DARCI (Defect Analysis & Root Cause Identification) agent using AWS CloudWatch logs.

## 📍 CloudWatch Log Groups

DARCI logs are stored in:
```
/ecs/specter/darci-worker
```

Related log groups:
- **Orchestrator**: `/ecs/specter/orchestrator` - Task assignment and result collection
- **SCRIBE**: `/ecs/specter/scribe-worker`
- **PRISM**: `/ecs/specter/prism-worker`
- **HERMES**: `/ecs/specter/hermes-worker`

## 🔍 Quick Access Commands

### 1. Tail DARCI Logs in Real-Time

```bash
# Real-time tail (like tail -f)
aws logs tail /ecs/specter/darci-worker \
  --follow \
  --region us-east-2 \
  --format short

# With timestamps
aws logs tail /ecs/specter/darci-worker \
  --follow \
  --region us-east-2 \
  --format detailed
```

### 2. Filter for Specific Task

```bash
# Replace MHK-12345 with your Jira ticket
aws logs tail /ecs/specter/darci-worker \
  --follow \
  --region us-east-2 \
  --filter-pattern "MHK-12345"
```

### 3. Filter by Log Level

```bash
# Errors only
aws logs tail /ecs/specter/darci-worker \
  --follow \
  --region us-east-2 \
  --filter-pattern '"level":50'  # ERROR level

# Warnings and above (WARN=40, ERROR=50)
aws logs tail /ecs/specter/darci-worker \
  --follow \
  --region us-east-2 \
  --filter-pattern '{ $.level >= 40 }'
```

### 4. Watch Specific Phoenix Events

```bash
# Watch iterations starting
aws logs tail /ecs/specter/darci-worker \
  --follow \
  --region us-east-2 \
  --filter-pattern '"Phoenix analysis starting"'

# Watch tool executions
aws logs tail /ecs/specter/darci-worker \
  --follow \
  --region us-east-2 \
  --filter-pattern '"tool"'

# Watch completion
aws logs tail /ecs/specter/darci-worker \
  --follow \
  --region us-east-2 \
  --filter-pattern '"Phoenix task complete"'
```

## 📊 Log Structure & Patterns

### Log Levels (Pino)
```
10 = TRACE
20 = DEBUG
30 = INFO
40 = WARN
50 = ERROR
60 = FATAL
```

### Key Log Modules
- `orchestrator` - DARCIWorker task processing
- `Phoenix` - Core agent loop execution
- `BedrockClient` - LLM interactions
- `ToolRegistry` - Tool execution

## 🎯 DARCI Execution Flow Logs

### 1. Task Pickup
```json
{
  "level": 30,
  "time": 1706371200000,
  "module": "orchestrator",
  "msg": "Processing defect analysis",
  "jira": "MHK-12345",
  "taskId": 42,
  "repositoryPath": "/mnt/efs/specter-repo/repositories/project-name"
}
```

### 2. Phoenix Initialization
```json
{
  "level": 30,
  "time": 1706371201000,
  "module": "Phoenix",
  "msg": "Phoenix initialized",
  "taskId": "darci-1706371200000"
}
```

### 3. Analysis Start
```json
{
  "level": 30,
  "time": 1706371202000,
  "module": "Phoenix",
  "msg": "🔥 Phoenix analysis starting",
  "taskId": "darci-1706371200000",
  "resumedFrom": null
}
```

### 4. Iteration Loop
```json
{
  "level": 30,
  "time": 1706371205000,
  "module": "Phoenix",
  "msg": "Iteration 1 starting"
}
```

### 5. Tool Execution
```json
{
  "level": 30,
  "time": 1706371206000,
  "module": "Phoenix",
  "msg": "Executing tool",
  "tool": "search_files",
  "params": { "path": "src", "regex": "login" }
}

{
  "level": 20,
  "time": 1706371207000,
  "module": "Phoenix",
  "msg": "Cache hit",
  "tool": "read_file"
}
```

### 6. Completion
```json
{
  "level": 30,
  "time": 1706371300000,
  "module": "Phoenix",
  "msg": "🔥 Phoenix task complete",
  "taskId": "darci-1706371200000",
  "success": true,
  "iterations": 12,
  "cost": 0.0234,
  "duration": 98000
}
```

## 🐛 Common Issues & How to Spot Them

### Issue 1: DARCI Not Picking Up Tasks
**Symptoms:**
```
No logs appearing in /ecs/specter/darci-worker
```

**Debug:**
```bash
# Check if DARCI container is running
aws ecs list-tasks \
  --cluster specter-cluster \
  --service-name DARCI \
  --region us-east-2

# Check for startup errors
aws logs tail /ecs/specter/darci-worker \
  --region us-east-2 \
  --since 30m \
  --filter-pattern '"level":50'
```

### Issue 2: Task Stuck in Loop
**Symptoms:**
```json
{
  "msg": "Iteration 50 starting"
}
{
  "msg": "Iteration 100 starting"
}
```

**Debug:**
```bash
# Watch iteration count
aws logs tail /ecs/specter/darci-worker \
  --follow \
  --region us-east-2 \
  --filter-pattern '"Iteration"'

# Check for "Iteration limit reached"
aws logs tail /ecs/specter/darci-worker \
  --follow \
  --region us-east-2 \
  --filter-pattern '"Iteration limit reached"'
```

### Issue 3: Tool Execution Failures
**Symptoms:**
```json
{
  "level": 50,
  "msg": "Tool execution failed",
  "tool": "git_analysis",
  "error": "Repository not found"
}
```

**Debug:**
```bash
# Watch tool failures
aws logs tail /ecs/specter/darci-worker \
  --follow \
  --region us-east-2 \
  --filter-pattern '{ $.tool = * && $.error = * }'

# Check EFS mount issues
aws logs tail /ecs/specter/darci-worker \
  --region us-east-2 \
  --since 1h \
  --filter-pattern '"/mnt/efs"'
```

### Issue 4: Bedrock/LLM Errors
**Symptoms:**
```json
{
  "level": 50,
  "msg": "LLM request failed",
  "error": "ThrottlingException"
}
```

**Debug:**
```bash
# Watch Bedrock errors
aws logs tail /ecs/specter/darci-worker \
  --follow \
  --region us-east-2 \
  --filter-pattern '"ThrottlingException"'

# Watch retries
aws logs tail /ecs/specter/darci-worker \
  --follow \
  --region us-east-2 \
  --filter-pattern '"LLM request failed"'
```

### Issue 5: High Cost / Token Usage
**Symptoms:**
```json
{
  "msg": "Token budget exceeded",
  "totalCost": 5.00
}
```

**Debug:**
```bash
# Watch token usage
aws logs tail /ecs/specter/darci-worker \
  --follow \
  --region us-east-2 \
  --filter-pattern '"cost"'

# Watch context compactions
aws logs tail /ecs/specter/darci-worker \
  --follow \
  --region us-east-2 \
  --filter-pattern '"Pruned expired cache entries"'
```

## 📈 Advanced CloudWatch Insights Queries

### Query 1: Average Task Duration
```
fields @timestamp, taskId, duration
| filter msg like /Phoenix task complete/
| stats avg(duration) as avgDuration, 
        max(duration) as maxDuration, 
        min(duration) as minDuration
```

### Query 2: Tool Usage Statistics
```
fields @timestamp, tool, duration
| filter tool != ""
| stats count() as calls, avg(duration) as avgDuration by tool
| sort calls desc
```

### Query 3: Error Rate by Hour
```
fields @timestamp
| filter level >= 50
| stats count() as errors by bin(@timestamp, 1h)
```

### Query 4: Cache Hit Rate
```
fields @timestamp, fromCache
| filter tool != ""
| stats sum(fromCache)/count() * 100 as cacheHitRate
```

### Query 5: Iteration Distribution
```
fields @timestamp, iterations, success
| filter msg like /Phoenix task complete/
| stats avg(iterations) as avgIterations, 
        max(iterations) as maxIterations by success
```

### Query 6: Find Slow Tools
```
fields @timestamp, tool, duration
| filter tool != ""
| filter duration > 5000
| sort duration desc
| limit 20
```

## 🖥️ CloudWatch Console Navigation

### Option 1: AWS Console (Web UI)
1. Go to: https://us-east-2.console.aws.amazon.com/cloudwatch/
2. Navigate to: **Logs** → **Log groups**
3. Click: `/ecs/specter/darci-worker`
4. Click: **Log streams** → Select latest stream
5. Enable: **Auto-refresh** (top right)

### Option 2: CloudWatch Insights
1. Go to: **CloudWatch** → **Logs Insights**
2. Select log group: `/ecs/specter/darci-worker`
3. Use queries above or write custom ones
4. Click: **Run query**

## 🚀 Quick Debug Script

Save this as `debug-darci.sh`:

```bash
#!/bin/bash
# Debug DARCI in real-time

REGION="us-east-2"
LOG_GROUP="/ecs/specter/darci-worker"

echo "🔍 Debugging DARCI..."
echo "Log Group: $LOG_GROUP"
echo "Region: $REGION"
echo ""

# Check if DARCI is running
echo "✅ Checking DARCI service status..."
aws ecs describe-services \
  --cluster specter-cluster \
  --services DARCI \
  --region $REGION \
  --query 'services[0].{Status:status,Running:runningCount,Desired:desiredCount}' \
  --output table

echo ""
echo "📊 Recent errors (last 30 min)..."
aws logs tail $LOG_GROUP \
  --region $REGION \
  --since 30m \
  --filter-pattern '{ $.level >= 40 }' \
  --format short | tail -20

echo ""
echo "🔥 Tailing logs (CTRL+C to stop)..."
aws logs tail $LOG_GROUP \
  --follow \
  --region $REGION \
  --format short
```

Make it executable:
```bash
chmod +x debug-darci.sh
./debug-darci.sh
```

## 🎨 Pretty Log Viewer

For better readability, pipe logs through jq:

```bash
aws logs tail /ecs/specter/darci-worker \
  --follow \
  --region us-east-2 \
  --format short | \
  jq -R 'fromjson? | 
    "\(.time | todate) [\(.level)] \(.module): \(.msg) \(.jira // "")"'
```

## 📝 Debugging Checklist

When debugging DARCI issues:

- [ ] Check if DARCI service is running (`aws ecs describe-services`)
- [ ] Verify EFS is mounted (`/mnt/efs/specter-repo`)
- [ ] Check database connectivity (look for DB errors)
- [ ] Verify Bedrock permissions (IAM role)
- [ ] Monitor token/cost usage
- [ ] Check for tool failures
- [ ] Review iteration counts (stuck loops?)
- [ ] Look for cache issues
- [ ] Check S3 result upload (final step)

## 🆘 Emergency Commands

### Stop DARCI Immediately
```bash
aws ecs update-service \
  --cluster specter-cluster \
  --service DARCI \
  --desired-count 0 \
  --region us-east-2
```

### Restart DARCI
```bash
aws ecs update-service \
  --cluster specter-cluster \
  --service DARCI \
  --force-new-deployment \
  --region us-east-2
```

### Get Last 100 Errors
```bash
aws logs tail /ecs/specter/darci-worker \
  --region us-east-2 \
  --since 24h \
  --filter-pattern '{ $.level >= 50 }' \
  --format short | head -100 > darci-errors.log
```

## 📚 Related Documentation

- [AWS Infrastructure](./AWS_INFRASTRUCTURE.md) - ECS and CloudWatch setup
- [Phoenix Implementation](./PHOENIX_IMPLEMENTATION.md) - Agent loop details
- [Tool Registry](./05-TOOL-REGISTRY.md) - Available tools for DARCI

---

**Pro Tip**: Keep the log tail running in a terminal while manually testing DARCI tasks to see real-time execution flow!
