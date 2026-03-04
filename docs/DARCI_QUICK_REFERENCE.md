# DARCI Debug Quick Reference Card

## 🚀 Quick Start

```bash
# Use the debug script
./scripts/debug-darci.sh tail

# Or use AWS CLI directly
aws logs tail /ecs/specter/darci-worker --follow --region us-east-2
```

## 📍 Key CloudWatch Locations

| Component | Log Group | Region |
|-----------|-----------|--------|
| **DARCI Worker** | `/ecs/specter/darci-worker` | us-east-2 |
| Orchestrator | `/ecs/specter/orchestrator` | us-east-2 |
| SCRIBE Worker | `/ecs/specter/scribe-worker` | us-east-2 |
| PRISM Worker | `/ecs/specter/prism-worker` | us-east-2 |
| HERMES Worker | `/ecs/specter/hermes-worker` | us-east-2 |

## 🎯 Common Debug Commands

```bash
# Watch DARCI in real-time
./scripts/debug-darci.sh tail

# Show only errors
./scripts/debug-darci.sh errors

# Track specific Jira task
./scripts/debug-darci.sh task MHK-12345

# Watch tool executions
./scripts/debug-darci.sh tools

# Check service status
./scripts/debug-darci.sh status

# Show CloudWatch Insights queries
./scripts/debug-darci.sh insights
```

## 🔍 Log Level Reference

```
10 = TRACE   (verbose debugging)
20 = DEBUG   (detailed info)
30 = INFO    (normal operation)
40 = WARN    (warnings)
50 = ERROR   (errors)
60 = FATAL   (critical failures)
```

## 🎬 DARCI Execution Sequence

```
1. [orchestrator] "Processing defect analysis" (task pickup)
2. [Phoenix] "Phoenix initialized" (agent created)
3. [Phoenix] "🔥 Phoenix analysis starting" (start)
4. [Phoenix] "Iteration X starting" (loop begins)
5. [Phoenix] Tool executions (read_file, search_files, etc.)
6. [Phoenix] "🎯 Root cause identified" (found solution)
7. [Phoenix] "🔥 Phoenix task complete" (finished)
8. [orchestrator] Result uploaded to S3
```

## 🐛 Emergency Commands

```bash
# Stop DARCI immediately
aws ecs update-service \
  --cluster specter-cluster \
  --service DARCI \
  --desired-count 0 \
  --region us-east-2

# Restart DARCI
aws ecs update-service \
  --cluster specter-cluster \
  --service DARCI \
  --force-new-deployment \
  --region us-east-2

# Check if running
aws ecs describe-services \
  --cluster specter-cluster \
  --services DARCI \
  --region us-east-2 \
  --query 'services[0].runningCount'
```

## 📊 Key Metrics to Watch

| Metric | Healthy Range | Red Flag |
|--------|---------------|----------|
| Iterations | 5-50 | > 100 |
| Cost per task | $0.01-0.10 | > $0.50 |
| Duration | 30s-10min | > 15min |
| Cache hit rate | 20-60% | < 10% |
| Tool failures | 0-5% | > 20% |

## 🎨 Useful Filter Patterns

```bash
# Errors only
--filter-pattern '{ $.level >= 50 }'

# Specific task
--filter-pattern '"MHK-12345"'

# Tool executions
--filter-pattern '"tool"'

# Phoenix events
--filter-pattern '"Phoenix"'

# Cost tracking
--filter-pattern '"cost"'

# Iterations
--filter-pattern '"Iteration"'

# Cache hits
--filter-pattern '"Cache hit"'
```

## 🔗 Quick Links

- **CloudWatch Console**: https://us-east-2.console.aws.amazon.com/cloudwatch/
- **CloudWatch Insights**: https://us-east-2.console.aws.amazon.com/cloudwatch/home?region=us-east-2#logsV2:logs-insights
- **ECS Console**: https://us-east-2.console.aws.amazon.com/ecs/v2/clusters/specter-cluster/services
- **Full Guide**: [DARCI_DEBUGGING_GUIDE.md](./DARCI_DEBUGGING_GUIDE.md)

## 💡 Pro Tips

1. **Use the debug script** - It's easier than raw AWS CLI commands
2. **Keep logs streaming** - Run in a dedicated terminal window
3. **Filter early** - Use filter patterns to reduce noise
4. **Watch iterations** - If stuck in loop, investigate tool failures
5. **Check EFS first** - Most issues are missing repos on `/mnt/efs`
6. **Monitor cost** - Set alerts if tasks cost > $0.20

## 🆘 Troubleshooting Decision Tree

```
No logs appearing?
  ├─ Check service status → ./scripts/debug-darci.sh status
  └─ Check ECS tasks → aws ecs list-tasks

Task stuck in loop?
  ├─ Watch iterations → ./scripts/debug-darci.sh iterations
  └─ Check tool failures → ./scripts/debug-darci.sh tools

High cost?
  ├─ Watch token usage → ./scripts/debug-darci.sh cost
  └─ Check context compactions (should see periodically)

Tool failures?
  ├─ Check EFS mount → Look for "/mnt/efs" errors
  └─ Check repo exists → SSH to bastion, ls /mnt/efs/specter-repo

LLM errors?
  └─ Check for "ThrottlingException" → Need to slow down
```

---

**Remember**: Most DARCI issues are either:
1. Missing repo on EFS (`/mnt/efs/specter-repo/repositories/<project>`)
2. Database connectivity
3. Bedrock throttling
4. Task assignment from Jira (check orchestrator logs)
