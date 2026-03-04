# MCP Integration Guide

Guide to integrating external services via Model Context Protocol (MCP).

---

## Overview

MCP (Model Context Protocol) allows agents to interact with external services like Jira, GitHub, Slack, etc. through a standardized protocol.

**Benefits**:
- ✅ Access external data sources
- ✅ Perform actions in external systems
- ✅ Standardized interface across services
- ✅ Authentication handled by MCP client

---

## MCP Architecture

```
┌──────────────────────────────────────────┐
│           Specter Agent                   │
│                                           │
│  ┌─────────────────────────────────────┐ │
│  │        Phoenix Engine               │ │
│  │                                     │ │
│  │  Registry Tools + MCP Tools         │ │
│  └────────────┬────────────────────────┘ │
└───────────────┼──────────────────────────┘
                │
                │ Tool calls routed by source
                │
        ┌───────┴────────┐
        │                │
        ▼                ▼
┌──────────────┐  ┌──────────────┐
│   Registry   │  │  MCPClient   │
│    Tools     │  │              │
└──────────────┘  └──────┬───────┘
                         │
                         │ JSON-RPC 2.0
                         ▼
                  ┌──────────────┐
                  │  MCP Server  │
                  │  (Jira, etc.)│
                  └──────────────┘
```

---

## Connection Types

### HTTP Connection

Most common for REST-based services.

```typescript
import { mcpTools } from './phoenix/PhoenixConfig.js';

const phoenix = new Phoenix({
  // ... other options
  toolSources: [
    registryTools(['read_file', 'search_files']),
    mcpTools({
      name: 'jira',
      type: 'http',
      url: 'https://your-jira-instance.com/mcp',
      auth: {
        type: 'bearer',
        token: process.env.JIRA_TOKEN,
      },
      timeout: 30000, // Optional: 30 second timeout
    }, ['jira_create_issue', 'jira_search_issues']),
  ],
});
```

### Stdio Connection

For spawned processes.

```typescript
mcpTools({
  name: 'local-service',
  type: 'stdio',
  command: 'node',
  args: ['./mcp-server.js'],
  env: {
    API_KEY: process.env.API_KEY,
  },
}, ['service_tool1', 'service_tool2'])
```

### WebSocket Connection

For real-time services.

```typescript
mcpTools({
  name: 'realtime-service',
  type: 'websocket',
  url: 'wss://service.example.com/mcp',
  auth: {
    type: 'bearer',
    token: process.env.SERVICE_TOKEN,
  },
}, ['subscribe', 'publish'])
```

---

## Authentication

### Bearer Token

```typescript
auth: {
  type: 'bearer',
  token: process.env.API_TOKEN,
}
```

### Basic Auth

```typescript
auth: {
  type: 'basic',
  username: process.env.USERNAME,
  password: process.env.PASSWORD,
}
```

### API Key

```typescript
auth: {
  type: 'api-key',
  headerName: 'X-API-Key',
  key: process.env.API_KEY,
}
```

---

## Real-World Examples

### Jira Integration

```typescript
import { Phoenix, registryTools, mcpTools } from './phoenix/index.js';

const JIRA_INTEGRATION_PROMPT = `You are a Jira integration specialist.

Available tools:
- read_file: Read local code files
- search_files: Search codebase
- jira_create_issue: Create new Jira issue
- jira_update_issue: Update existing issue
- jira_search_issues: Search for issues

Process:
1. READ: Use read_file/search_files to understand the defect
2. SEARCH: Use jira_search_issues to check for duplicates
3. CREATE: Use jira_create_issue if new issue
4. UPDATE: Use jira_update_issue if duplicate found

Output format:
\`\`\`json
{
  "action": "created",
  "issueKey": "PROJ-123",
  "issueUrl": "https://jira.../PROJ-123",
  "summary": "Brief description"
}
\`\`\`
`;

export class JiraAgent {
  constructor(
    private options: {
      repositoryPath: string;
      jiraUrl: string;
      jiraToken: string;
    }
  ) {}

  async createIssueFromDefect(defectDescription: string) {
    const phoenix = new Phoenix({
      taskId: `jira-${Date.now()}`,
      taskDescription: `Create Jira issue for: ${defectDescription}`,
      repositoryPath: this.options.repositoryPath,
      systemPrompt: JIRA_INTEGRATION_PROMPT,

      toolSources: [
        registryTools(['read_file', 'search_files']),
        mcpTools({
          name: 'jira',
          type: 'http',
          url: `${this.options.jiraUrl}/mcp`,
          auth: {
            type: 'bearer',
            token: this.options.jiraToken,
          },
        }, ['jira_create_issue', 'jira_search_issues', 'jira_update_issue']),
      ],

      outputFormat: {
        name: 'jira_action',
        schema: {
          type: 'object',
          properties: {
            action: { type: 'string', enum: ['created', 'updated'] },
            issueKey: { type: 'string' },
            issueUrl: { type: 'string' },
            summary: { type: 'string' },
          },
          required: ['action', 'issueKey'],
        },
      },

      maxIterations: 15,
      enableCaching: false, // Don't cache external service calls
      enableDeduplication: false, // External calls may have side effects
    });

    const result = await phoenix.run();
    return result.structuredOutput;
  }
}

// Usage:
const agent = new JiraAgent({
  repositoryPath: '/repo',
  jiraUrl: 'https://your-jira.atlassian.net',
  jiraToken: process.env.JIRA_TOKEN!,
});

const result = await agent.createIssueFromDefect('Login fails after password reset');
console.log(`Issue created: ${result.issueUrl}`);
```

---

### GitHub Integration

```typescript
const GITHUB_PR_PROMPT = `You are a GitHub PR specialist.

Available tools:
- read_file: Read code changes
- git_analysis: Check git history
- github_create_pr: Create pull request
- github_add_comment: Add comment to PR

Process:
1. ANALYZE: Use git_analysis to see changes
2. READ: Use read_file to understand code
3. CREATE: Use github_create_pr with:
   - Title
   - Description
   - Labels
4. COMMENT: Add testing instructions

Output format:
\`\`\`json
{
  "prNumber": 123,
  "prUrl": "https://github.../pull/123",
  "title": "...",
  "filesChanged": 5
}
\`\`\`
`;

export class GitHubAgent {
  constructor(
    private options: {
      repositoryPath: string;
      githubToken: string;
    }
  ) {}

  async createPR(branch: string, baseBranch: string) {
    const phoenix = new Phoenix({
      taskId: `gh-pr-${Date.now()}`,
      taskDescription: `Create PR for ${branch} -> ${baseBranch}`,
      repositoryPath: this.options.repositoryPath,
      systemPrompt: GITHUB_PR_PROMPT,

      toolSources: [
        registryTools(['read_file', 'git_analysis']),
        mcpTools({
          name: 'github',
          type: 'http',
          url: 'https://api.github.com/mcp',
          auth: {
            type: 'bearer',
            token: this.options.githubToken,
          },
        }, ['github_create_pr', 'github_add_comment']),
      ],

      maxIterations: 10,
    });

    return await phoenix.run();
  }
}
```

---

### Slack Integration

```typescript
const SLACK_NOTIFICATION_PROMPT = `You are a Slack notification specialist.

Available tools:
- read_file: Read relevant files
- slack_post_message: Post message to channel

Process:
1. READ: Gather context from files
2. FORMAT: Create well-formatted message with:
   - Summary
   - Key details
   - Action items
3. POST: Use slack_post_message

Output format:
\`\`\`json
{
  "channel": "#alerts",
  "timestamp": "1234567890",
  "messageUrl": "https://slack.com/..."
}
\`\`\`
`;

export class SlackAgent {
  constructor(
    private options: {
      repositoryPath: string;
      slackToken: string;
    }
  ) {}

  async notifyChannel(channel: string, message: string) {
    const phoenix = new Phoenix({
      taskId: `slack-${Date.now()}`,
      taskDescription: `Send message to ${channel}: ${message}`,
      repositoryPath: this.options.repositoryPath,
      systemPrompt: SLACK_NOTIFICATION_PROMPT,

      toolSources: [
        registryTools(['read_file']),
        mcpTools({
          name: 'slack',
          type: 'http',
          url: 'https://slack.com/api/mcp',
          auth: {
            type: 'bearer',
            token: this.options.slackToken,
          },
        }, ['slack_post_message', 'slack_upload_file']),
      ],

      maxIterations: 5,
    });

    return await phoenix.run();
  }
}
```

---

## MCP Tool Discovery

Phoenix automatically discovers available tools from MCP servers:

```typescript
// After connection, Phoenix calls:
// POST /rpc
// { "jsonrpc": "2.0", "method": "tools/list", "id": 1 }

// Server responds with:
{
  "jsonrpc": "2.0",
  "result": {
    "tools": [
      {
        "name": "jira_create_issue",
        "description": "Create a new Jira issue",
        "inputSchema": {
          "type": "object",
          "properties": {
            "projectKey": { "type": "string" },
            "summary": { "type": "string" },
            "description": { "type": "string" },
          },
          "required": ["projectKey", "summary"]
        }
      }
    ]
  }
}

// Phoenix adds these tools to available tools list
```

---

## Tool Execution

When agent calls an MCP tool:

```typescript
// Agent calls:
await jira_create_issue({
  projectKey: 'PROJ',
  summary: 'Login bug',
  description: 'Users cannot login after password reset'
});

// Phoenix sends to MCP server:
// POST /rpc
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "params": {
    "name": "jira_create_issue",
    "arguments": {
      "projectKey": "PROJ",
      "summary": "Login bug",
      "description": "Users cannot login after password reset"
    }
  },
  "id": 2
}

// Server responds:
{
  "jsonrpc": "2.0",
  "result": {
    "content": [
      {
        "type": "text",
        "text": "Issue PROJ-123 created successfully"
      }
    ]
  },
  "id": 2
}

// Phoenix returns result to agent
```

---

## Error Handling

### Connection Errors

```typescript
import { EventEmitter } from 'events';

const emitter = new EventEmitter();

emitter.on('mcp:error', ({ server, error }) => {
  console.error(`MCP ${server} error:`, error);
  // Alert, retry, or fallback
});

const phoenix = new Phoenix({
  // ...
  eventEmitter: emitter,
});
```

### Tool Execution Errors

MCP tools automatically retry with exponential backoff (if `enableErrorRecovery: true`):

```typescript
// First attempt fails
jira_create_issue(...) // Network error

// Phoenix retries:
// Attempt 2 after 1s
// Attempt 3 after 2s
// Attempt 4 after 4s

// After 3 retries, returns error to agent
```

### Fallback Strategies

```typescript
const PROMPT_WITH_FALLBACK = `...
If jira_create_issue fails:
1. Try jira_search_issues to check if issue already exists
2. If server unavailable, save details locally with write_file
3. Report that issue will be created manually
`;
```

---

## Best Practices

### 1. Don't Cache External Calls

```typescript
enableCaching: false, // External state may change
```

### 2. Don't Deduplicate External Calls

```typescript
enableDeduplication: false, // Calls may have side effects
```

### 3. Handle Authentication Securely

```typescript
// ✅ Use environment variables
auth: {
  type: 'bearer',
  token: process.env.JIRA_TOKEN,
}

// ❌ Don't hardcode tokens
auth: {
  type: 'bearer',
  token: 'hardcoded-token-here',
}
```

### 4. Set Appropriate Timeouts

```typescript
mcpTools({
  name: 'slow-service',
  type: 'http',
  url: '...',
  timeout: 60000, // 60 second timeout for slow services
}, [...])
```

### 5. Monitor MCP Events

```typescript
emitter.on('mcp:connect', ({ server }) => {
  console.log(`Connected to ${server}`);
});

emitter.on('mcp:disconnect', ({ server }) => {
  console.warn(`Disconnected from ${server}`);
});

emitter.on('mcp:tool_called', ({ server, tool, duration }) => {
  console.log(`${server}.${tool} took ${duration}ms`);
});
```

### 6. Graceful Degradation

```typescript
try {
  const result = await agent.executeWithMCP(task);
  return result;
} catch (error) {
  console.warn('MCP service unavailable, falling back to local analysis');
  return await agent.executeWithoutMCP(task);
}
```

---

## Testing with MCP

### Mock MCP Server

```typescript
// tests/mocks/mcp-server.ts
export class MockMCPServer {
  tools = new Map();

  registerTool(name: string, handler: Function) {
    this.tools.set(name, handler);
  }

  async handleRequest(request: any) {
    if (request.method === 'tools/list') {
      return {
        tools: Array.from(this.tools.keys()).map(name => ({
          name,
          description: `Mock ${name}`,
          inputSchema: { type: 'object', properties: {} },
        })),
      };
    }

    if (request.method === 'tools/call') {
      const handler = this.tools.get(request.params.name);
      if (handler) {
        return await handler(request.params.arguments);
      }
    }

    throw new Error('Method not found');
  }
}
```

### Test with Mock

```typescript
import { MockMCPServer } from './mocks/mcp-server.js';

describe('JiraAgent', () => {
  let mockServer: MockMCPServer;

  beforeEach(() => {
    mockServer = new MockMCPServer();

    mockServer.registerTool('jira_create_issue', async (params) => {
      return {
        content: [{
          type: 'text',
          text: JSON.stringify({ key: 'PROJ-123', id: '10000' }),
        }],
      };
    });

    // Start mock server on localhost:3000
  });

  it('should create Jira issue', async () => {
    const agent = new JiraAgent({
      repositoryPath: '/test',
      jiraUrl: 'http://localhost:3000',
      jiraToken: 'test-token',
    });

    const result = await agent.createIssueFromDefect('Test defect');

    expect(result.issueKey).toBe('PROJ-123');
  });
});
```

---

## Available MCP Integrations

Phoenix includes example MCP configurations for:

- **Jira**: Issue tracking
- **GitHub**: Pull requests, issues
- **Slack**: Notifications
- **PagerDuty**: Incident management
- **Confluence**: Documentation
- **DataDog**: Monitoring

See `examples/mcp/` directory for implementations.

---

**Next**: [07-OUTPUT-FORMATS.md](./07-OUTPUT-FORMATS.md)
