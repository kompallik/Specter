# Specter Status API Documentation

The Status API provides real-time monitoring capabilities for DARCI and other agent workers.

## Overview

The Status Server is an HTTP/WebSocket server embedded in each worker that provides:

- **REST endpoints** for querying task status
- **WebSocket** for real-time event streaming
- **Server-Sent Events (SSE)** for simpler real-time updates
- **Webhook notifications** for external integrations

## Endpoints

### Health Check
```
GET /health
```

Returns server health status.

**Response:**
```json
{
  "status": "healthy",
  "timestamp": "2025-01-27T15:30:00.000Z",
  "activeTasks": 3,
  "connectedClients": 2,
  "sseClients": 1
}
```

### All Task Statuses
```
GET /status
```

Returns status for all active tasks.

**Response:**
```json
{
  "count": 2,
  "tasks": {
    "task-123": {
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
  },
  "timestamp": "2025-01-27T15:30:00.000Z"
}
```

### Single Task Status
```
GET /status/:taskId
```

Returns status for a specific task.

**Response:** Same as single task object above.

### Event History
```
GET /history?count=100
```

Returns recent event history.

**Response:**
```json
{
  "count": 100,
  "events": [
    {
      "timestamp": "2025-01-27T15:30:00.000Z",
      "event": "tool:start",
      "data": { "iteration": 5, "tool": "git_analysis" }
    }
  ],
  "timestamp": "2025-01-27T15:30:00.000Z"
}
```

### Metrics
```
GET /metrics
```

Returns aggregated metrics across all tasks.

**Response:**
```json
{
  "totalTasks": 10,
  "completedTasks": 7,
  "failedTasks": 1,
  "inProgressTasks": 2,
  "pendingTasks": 0,
  "totalCost": 0.45,
  "totalDurationMs": 3600000,
  "totalIterations": 250,
  "averageCostPerTask": 0.045,
  "averageDurationMs": 360000,
  "connectedClients": 3,
  "sseClients": 1,
  "timestamp": "2025-01-27T15:30:00.000Z"
}
```

## Real-Time Streaming

### WebSocket

Connect to `ws://host:port` for real-time events.

**Connection:**
```javascript
const ws = new WebSocket('ws://localhost:8080');

ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log(data);
};
```

**Welcome Message:**
```json
{
  "type": "connected",
  "clientId": "ws-1234567890-abc123",
  "timestamp": "2025-01-27T15:30:00.000Z",
  "activeTasks": ["task-123", "task-456"]
}
```

**Client Commands:**

Subscribe to specific tasks:
```json
{ "type": "subscribe", "taskIds": ["task-123", "task-456"] }
```

Subscribe to all tasks:
```json
{ "type": "subscribe_all" }
```

Get task status:
```json
{ "type": "get_status", "taskId": "task-123" }
```

Get all statuses:
```json
{ "type": "get_status" }
```

Get event history:
```json
{ "type": "get_history", "count": 50 }
```

Ping:
```json
{ "type": "ping" }
```

### Server-Sent Events (SSE)

Connect to `GET /events` for SSE stream.

**Example with curl:**
```bash
curl -N http://localhost:8080/events
```

**Example with JavaScript:**
```javascript
const eventSource = new EventSource('http://localhost:8080/events');

eventSource.addEventListener('tool:start', (event) => {
  const data = JSON.parse(event.data);
  console.log('Tool started:', data);
});

eventSource.addEventListener('analysis:complete', (event) => {
  const data = JSON.parse(event.data);
  console.log('Analysis complete:', data);
});
```

## Event Types

Events emitted by Phoenix and forwarded by the Status Server:

| Event | Description |
|-------|-------------|
| `analysis:start` | Task analysis started |
| `analysis:complete` | Task analysis completed |
| `iteration:start` | New iteration started |
| `iteration:end` | Iteration completed |
| `llm:request` | LLM request made |
| `llm:response` | LLM response received |
| `llm:thinking` | LLM thinking text |
| `tool:start` | Tool execution started |
| `tool:end` | Tool execution completed |
| `verification:start` | Verification started |
| `verification:end` | Verification completed |
| `context:compacted` | Context window compacted |
| `checkpoint:saved` | Checkpoint saved |
| `error` | Error occurred |
| `guidance` | Tool guidance injected |

## Webhooks

Configure webhooks via environment variables:

```bash
ENABLE_WEBHOOKS=true
WEBHOOK_URL=https://your-endpoint.com/webhook
WEBHOOK_SECRET=your-secret-key
```

Webhook payloads include:
```json
{
  "taskId": "task-123",
  "event": "analysis:complete",
  "data": { ... },
  "timestamp": "2025-01-27T15:30:00.000Z"
}
```

When `WEBHOOK_SECRET` is set, requests include `X-Webhook-Signature` header with HMAC-SHA256 signature.

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `STATUS_PORT` | 8080 | HTTP/WebSocket port |
| `STATUS_HOST` | 0.0.0.0 | Bind address |
| `WORKER_TYPE` | DARCI | Worker/agent type |
| `ENABLE_WEBHOOKS` | false | Enable webhook notifications |
| `WEBHOOK_URL` | - | Webhook endpoint URL |
| `WEBHOOK_SECRET` | - | Webhook HMAC secret |

## Example: Monitor DARCI in Real-Time

### Using curl
```bash
# Health check
curl http://localhost:8080/health

# Get all task statuses
curl http://localhost:8080/status

# Stream events (SSE)
curl -N http://localhost:8080/events
```

### Using wscat (WebSocket)
```bash
# Install wscat
npm install -g wscat

# Connect
wscat -c ws://localhost:8080

# Subscribe to all events
> {"type":"subscribe_all"}

# Get current status
> {"type":"get_status"}
```

### Using JavaScript (Browser/Node.js)
```javascript
// WebSocket client
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
      console.log(`✅ ${data.data.tool} completed in ${data.data.duration}ms`);
      break;
    case 'iteration:start':
      console.log(`🔄 Iteration ${data.data.iteration} (${data.data.phase})`);
      break;
    case 'analysis:complete':
      console.log(`🎯 Analysis complete! Cost: $${data.data.cost}`);
      break;
    case 'error':
      console.error(`❌ Error: ${data.data.error}`);
      break;
  }
};
```

## Deployment

The Status Server is automatically included in the worker Docker image. Port 8080 is exposed by default.

### ECS Task Definition
```json
{
  "containerDefinitions": [
    {
      "name": "darci-worker",
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        { "name": "STATUS_PORT", "value": "8080" },
        { "name": "WORKER_TYPE", "value": "DARCI" }
      ],
      "healthCheck": {
        "command": ["CMD-SHELL", "wget -qO- http://localhost:8080/health || exit 1"],
        "interval": 30,
        "timeout": 10,
        "retries": 3,
        "startPeriod": 10
      }
    }
  ]
}
```

### Load Balancer Setup
If exposing externally, configure ALB with:
- Target group on port 8080
- Health check path: `/health`
- WebSocket support enabled (sticky sessions)
