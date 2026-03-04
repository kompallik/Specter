# Specter Real-Time Monitoring Guide

## Overview

Specter workers now include a built-in Status Server that provides real-time monitoring of agent task execution. Each worker (DARCI, SCRIBE, PRISM, HERMES) runs its own Status Server on port 8080.

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           SPECTER CLUSTER                                │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐        │
│  │   DARCI Worker  │   │  SCRIBE Worker  │   │  PRISM Worker   │        │
│  │                 │   │                 │   │                 │        │
│  │  ┌───────────┐  │   │  ┌───────────┐  │   │  ┌───────────┐  │        │
│  │  │ Status    │  │   │  │ Status    │  │   │  │ Status    │  │        │
│  │  │ Server    │  │   │  │ Server    │  │   │  │ Server    │  │        │
│  │  │ :8080     │  │   │  │ :8080     │  │   │  │ :8080     │  │        │
│  │  └───────────┘  │   │  └───────────┘  │   │  └───────────┘  │        │
│  │                 │   │                 │   │                 │        │
│  │  ┌───────────┐  │   │  ┌───────────┐  │   │  ┌───────────┐  │        │
│  │  │ Phoenix   │  │   │  │ Phoenix   │  │   │  │ Phoenix   │  │        │
│  │  │ Agent     │  │   │  │ Agent     │  │   │  │ Agent     │  │        │
│  │  └───────────┘  │   │  └───────────┘  │   │  └───────────┘  │        │
│  └─────────────────┘   └─────────────────┘   └─────────────────┘        │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```


## Quick Start

### Connect via SSH Tunnel

Since workers run in a private subnet, access them through the bastion:

```bash
# 1. Get the DARCI task private IP
TASK_ARN=$(aws ecs list-tasks --cluster specter-cluster --service-name DARCI --query 'taskArns[0]' --output text --region us-east-2)

TASK_IP=$(aws ecs describe-tasks --cluster specter-cluster --tasks $TASK_ARN --region us-east-2 --query 'tasks[0].attachments[0].details[?name==`privateIPv4Address`].value | [0]' --output text)

echo "DARCI Task IP: $TASK_IP"

# 2. Create SSH tunnel
ssh -i document-ai-bastion.pem -L 8080:$TASK_IP:8080 ec2-user@3.133.142.220

# 3. Access locally (in another terminal)
curl http://localhost:8080/health
curl http://localhost:8080/status
curl http://localhost:8080/metrics
```

### Monitor with WebSocket (Recommended)

```bash
# SSH to bastion
ssh -i document-ai-bastion.pem ec2-user@3.133.142.220

# Install wscat if not present
npm install -g wscat

# Connect to DARCI status server
wscat -c ws://<TASK_IP>:8080

# In the WebSocket session:
# Subscribe to all events
{"type":"subscribe_all"}

# Get current status
{"type":"get_status"}

# Get event history
{"type":"get_history", "count": 50}
```

### Monitor with Server-Sent Events

```bash
# From bastion
curl -N http://<TASK_IP>:8080/events
```

## REST API Endpoints

### Health Check
```bash
GET /health

# Response
{
  "status": "healthy",
  "timestamp": "2025-01-27T15:30:00.000Z",
  "activeTasks": 3,
  "connectedClients": 2,
  "sseClients": 1
}
```

### All Task Statuses
```bash
GET /status

# Response
{
  "count": 2,
  "tasks": {
    "task-123": { ... },
    "task-456": { ... }
  },
  "timestamp": "2025-01-27T15:30:00.000Z"
}
```

### Single Task Status
```bash
GET /status/task-123

# Response
{
  "taskId": "task-123",
  "jiraNumber": "PROJ-456",
  "agentName": "DARCI",
  "status": "in_progress",
  "iteration": 5,
  "phase": "investigation",
  "startTime": "2025-01-27T15:00:00.000Z",
  "lastUpdate": "2025-01-27T15:30:00.000Z",
  "toolsUsed": ["read_file", "search_files", "git_analysis"],
  "metrics": {
    "inputTokens": 25000,
    "outputTokens": 5000,
    "cost": 0.0125,
    "durationMs": 180000
  },
  "currentTool": "git_analysis",
  "lastMessage": "Using tool: git_analysis"
}
```

### Aggregated Metrics
```bash
GET /metrics

# Response
{
  "totalTasks": 10,
  "completedTasks": 7,
  "failedTasks": 1,
  "inProgressTasks": 2,
  "totalCost": 0.45,
  "totalDurationMs": 3600000,
  "averageCostPerTask": 0.045,
  "connectedClients": 3,
  "timestamp": "2025-01-27T15:30:00.000Z"
}
```

### Event History
```bash
GET /history?count=100

# Response
{
  "count": 100,
  "events": [
    {
      "timestamp": "2025-01-27T15:30:00.000Z",
      "event": "tool:start",
      "data": { "iteration": 5, "tool": "git_analysis" }
    }
  ]
}
```

## WebSocket Protocol

### Connection
```javascript
const ws = new WebSocket('ws://localhost:8080');
```

### Commands

| Command | Description | Example |
|---------|-------------|---------|
| `subscribe` | Subscribe to specific tasks | `{"type":"subscribe","taskIds":["task-123"]}` |
| `subscribe_all` | Subscribe to all tasks | `{"type":"subscribe_all"}` |
| `get_status` | Get task status | `{"type":"get_status","taskId":"task-123"}` |
| `get_history` | Get event history | `{"type":"get_history","count":50}` |
| `ping` | Heartbeat | `{"type":"ping"}` |

### Events Received

```javascript
// Tool execution
{ "taskId": "task-123", "event": "tool:start", "data": { "iteration": 5, "tool": "git_analysis" }, "timestamp": "..." }
{ "taskId": "task-123", "event": "tool:end", "data": { "tool": "git_analysis", "success": true, "duration": 1234 }, "timestamp": "..." }

// Iteration
{ "taskId": "task-123", "event": "iteration:start", "data": { "iteration": 5, "phase": "investigation" }, "timestamp": "..." }
{ "taskId": "task-123", "event": "iteration:end", "data": { "iteration": 5, "duration": 5000, "toolsUsed": ["read_file"] }, "timestamp": "..." }

// Analysis
{ "taskId": "task-123", "event": "analysis:start", "data": { "taskId": "task-123" }, "timestamp": "..." }
{ "taskId": "task-123", "event": "analysis:complete", "data": { "success": true, "cost": 0.05, "duration": 180000 }, "timestamp": "..." }

// Errors
{ "taskId": "task-123", "event": "error", "data": { "error": "Tool execution failed" }, "timestamp": "..." }
```

## JavaScript Client Example

```javascript
// Browser/Node.js WebSocket Client
const ws = new WebSocket('ws://localhost:8080');

ws.onopen = () => {
  console.log('Connected!');
  ws.send(JSON.stringify({ type: 'subscribe_all' }));
};

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  
  switch (data.event) {
    case 'tool:start':
      console.log(`🔧 ${data.data.tool} started`);
      break;
    case 'tool:end':
      const status = data.data.success ? '✅' : '❌';
      console.log(`${status} ${data.data.tool} (${data.data.duration}ms)`);
      break;
    case 'iteration:start':
      console.log(`\n🔄 Iteration ${data.data.iteration} (${data.data.phase})`);
      break;
    case 'analysis:complete':
      console.log(`\n🎯 Complete! Cost: $${data.data.cost.toFixed(4)}`);
      break;
    case 'error':
      console.error(`❌ ${data.data.error}`);
      break;
  }
};

ws.onerror = (err) => console.error('Error:', err);
ws.onclose = () => console.log('Disconnected');
```

## Server-Sent Events (SSE) Example

```javascript
// Browser EventSource
const eventSource = new EventSource('http://localhost:8080/events');

eventSource.addEventListener('tool:start', (event) => {
  const data = JSON.parse(event.data);
  console.log('Tool started:', data.data.tool);
});

eventSource.addEventListener('analysis:complete', (event) => {
  const data = JSON.parse(event.data);
  console.log('Analysis complete! Cost:', data.data.cost);
});

eventSource.onerror = (err) => {
  console.error('SSE error:', err);
  eventSource.close();
};
```

## Webhook Configuration

Enable webhooks via environment variables:

```bash
# In ECS task definition or docker run
ENABLE_WEBHOOKS=true
WEBHOOK_URL=https://your-endpoint.com/specter-webhook
WEBHOOK_SECRET=your-hmac-secret  # Optional
```

### Webhook Payload

```json
{
  "taskId": "task-123",
  "event": "analysis:complete",
  "data": {
    "taskId": "task-123",
    "success": true,
    "iterations": 15,
    "duration": 180000,
    "cost": 0.05
  },
  "timestamp": "2025-01-27T15:30:00.000Z"
}
```

### Webhook Signature

When `WEBHOOK_SECRET` is set, requests include:
```
X-Webhook-Signature: sha256=<HMAC-SHA256 of JSON body>
```

## Event Types Reference

| Event | When | Data |
|-------|------|------|
| `analysis:start` | Task begins | `{ taskId, resumedFrom? }` |
| `analysis:complete` | Task finishes | `{ taskId, success, iterations, duration, cost }` |
| `iteration:start` | New iteration | `{ iteration, phase, timestamp }` |
| `iteration:end` | Iteration done | `{ iteration, duration, toolsUsed, progress }` |
| `llm:request` | LLM call made | `{ iteration, messageCount, estimatedTokens }` |
| `llm:response` | LLM response | `{ iteration, textLength, toolCalls, duration }` |
| `llm:thinking` | LLM reasoning | `{ iteration, text }` |
| `tool:start` | Tool begins | `{ iteration, tool, params, source }` |
| `tool:end` | Tool finishes | `{ iteration, tool, success, duration, fromCache, source }` |
| `verification:start` | Verification | `{ type }` |
| `verification:end` | Verified | `{ verified, confidence, issues }` |
| `context:compacted` | Memory saved | `{ tokensSaved, messagesBefore, messagesAfter }` |
| `checkpoint:saved` | State saved | `{ checkpointId, iteration }` |
| `error` | Error occurs | `{ iteration, error, recoveryAction? }` |
| `guidance` | Hint given | `{ iteration, messages }` |

## Deployment

The Status Server is automatically included when deploying workers:

```bash
# Build
npm run build

# Build Docker image
docker build --platform linux/amd64 --load -f Dockerfile.worker -t specter/darci-worker:latest .

# Push to ECR
aws ecr get-login-password --region us-east-2 | docker login --username AWS --password-stdin 323960442001.dkr.ecr.us-east-2.amazonaws.com
docker tag specter/darci-worker:latest 323960442001.dkr.ecr.us-east-2.amazonaws.com/specter/darci-worker:latest
docker push 323960442001.dkr.ecr.us-east-2.amazonaws.com/specter/darci-worker:latest

# Deploy
aws ecs update-service --cluster specter-cluster --service DARCI --force-new-deployment --region us-east-2
```

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `STATUS_PORT` | 8080 | Status server port |
| `STATUS_HOST` | 0.0.0.0 | Bind address |
| `WORKER_TYPE` | DARCI | Worker type (DARCI, SCRIBE, PRISM, HERMES) |
| `ENABLE_WEBHOOKS` | false | Enable webhook notifications |
| `WEBHOOK_URL` | - | Webhook endpoint URL |
| `WEBHOOK_SECRET` | - | HMAC secret for signatures |

## Troubleshooting

### Can't connect to Status Server

1. Verify the task is running:
   ```bash
   aws ecs list-tasks --cluster specter-cluster --service-name DARCI --region us-east-2
   ```

2. Get the task IP:
   ```bash
   TASK_ARN=$(aws ecs list-tasks --cluster specter-cluster --service-name DARCI --query 'taskArns[0]' --output text --region us-east-2)
   aws ecs describe-tasks --cluster specter-cluster --tasks $TASK_ARN --region us-east-2 --query 'tasks[0].attachments[0].details[?name==`privateIPv4Address`].value | [0]' --output text
   ```

3. Check security group allows port 8080

### WebSocket disconnects

Add reconnection logic:
```javascript
function connect() {
  const ws = new WebSocket('ws://localhost:8080');
  ws.onclose = () => setTimeout(connect, 5000);
  ws.onopen = () => ws.send(JSON.stringify({ type: 'subscribe_all' }));
  return ws;
}
```

### No events received

Ensure you've subscribed:
```javascript
ws.send(JSON.stringify({ type: 'subscribe_all' }));
```

Or subscribe to specific tasks:
```javascript
ws.send(JSON.stringify({ type: 'subscribe', taskIds: ['task-123'] }));
```
