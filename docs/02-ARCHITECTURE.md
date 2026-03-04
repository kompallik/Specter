# Specter Architecture

A comprehensive guide to understanding Specter's architecture and design principles.

---

## Table of Contents

1. [High-Level Architecture](#high-level-architecture)
2. [Phoenix Engine Deep Dive](#phoenix-engine-deep-dive)
3. [Agent Specialization Pattern](#agent-specialization-pattern)
4. [Tool System](#tool-system)
5. [MCP Integration](#mcp-integration)
6. [Data Flow](#data-flow)
7. [Optimization Systems](#optimization-systems)

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        SPECTER CLUSTER                       │
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │ DefectAgent  │  │ ExplainAgent │  │ ConfigAgent  │ ... │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘     │
│         │                  │                  │              │
│         └──────────────────┴──────────────────┘              │
│                            │                                 │
│                            ▼                                 │
│              ┌─────────────────────────┐                    │
│              │    PHOENIX ENGINE       │                    │
│              │  (Autonomous Loop)      │                    │
│              └─────────┬───────────────┘                    │
│                        │                                     │
│         ┌──────────────┼──────────────┐                    │
│         │              │              │                     │
│         ▼              ▼              ▼                     │
│  ┌───────────┐  ┌──────────┐  ┌──────────┐               │
│  │   Tools   │  │   MCP    │  │   LLM    │               │
│  │ Registry  │  │ Servers  │  │ (Bedrock)│               │
│  └───────────┘  └──────────┘  └──────────┘               │
└─────────────────────────────────────────────────────────────┘
```

### Key Components

#### 1. Specter Cluster
- Collection of specialized agents
- Each agent optimized for specific task type
- Shared Phoenix execution engine
- Deployed as ECS services (containerized)

#### 2. Agents
- **DefectAgent**: Software defect root cause analysis
- **ExplainAgent**: Code functionality explanation
- **ConfigAgent**: Configuration value lookup
- **[Your Custom Agents]**: Domain-specific tasks

#### 3. Phoenix Engine
- Autonomous agent loop
- 12 integrated optimizations
- Tool orchestration
- LLM interaction management

#### 4. Tool System
- **Registry Tools**: Built-in file system tools
- **MCP Servers**: External service integration (Jira, GitHub, etc.)

#### 5. LLM Backend
- AWS Bedrock (Claude Sonnet 4.5)
- Token tracking and cost monitoring
- Streaming support

---

## Phoenix Engine Deep Dive

Phoenix is the core execution engine that powers all Specter agents.

### Architecture

```
┌────────────────────────────────────────────────────────────┐
│                      PHOENIX                                │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │              AGENTIC LOOP (run)                       │ │
│  │                                                        │ │
│  │  1. Build Prompt                                      │ │
│  │  2. Call LLM (Bedrock)                               │ │
│  │  3. Parse Response (Text + Tool Calls)               │ │
│  │  4. Execute Tools (Parallel if enabled)              │ │
│  │  5. Add Results to History                           │ │
│  │  6. Check Completion                                  │ │
│  │  7. Repeat until done or max iterations              │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │           12 OPTIMIZATION SYSTEMS                     │ │
│  │                                                        │ │
│  │  1. ContextManager      - Smart history pruning      │ │
│  │  2. ToolResultCache     - Deduplicate tool calls     │ │
│  │  3. ParallelExecutor    - Concurrent tool execution  │ │
│  │  4. CheckpointManager   - Save/restore progress      │ │
│  │  5. SelfReflection      - Agent reviews own work     │ │
│  │  6. ErrorRecovery       - Auto-retry failed tools    │ │
│  │  7. AdaptiveIterations  - Dynamic iteration limit    │ │
│  │  8. TokenTracker        - Cost and usage monitoring  │ │
│  │  9. ToolUsageAnalyzer   - Suggest next best tool    │ │
│  │ 10. SemanticDedup       - Skip similar operations    │ │
│  │ 11. MultiModelVerifier  - Cross-check with models    │ │
│  │ 12. OutputValidator     - Validate against schema    │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐ │
│  │              TOOL ORCHESTRATION                       │ │
│  │                                                        │ │
│  │  - Route to Registry or MCP                          │ │
│  │  - Parameter validation                              │ │
│  │  - Error handling                                     │ │
│  │  - Result formatting                                  │ │
│  └──────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────┘
```

### Phoenix Lifecycle

```typescript
// 1. INITIALIZATION
const phoenix = new Phoenix({
  taskId: 'task-123',
  taskDescription: 'Analyze login bug',
  repositoryPath: '/repo',
  systemPrompt: DEFECT_PROMPT,
  toolSources: [registryTools(['read_file', 'git_analysis'])],
  outputFormat: OutputFormats.DEFECT_ANALYSIS,
});

// 2. EXECUTION
const result = await phoenix.run();
/*
  Internally:
  - Initialize optimization systems
  - Connect to MCP servers (if any)
  - Start agentic loop:
    while (!done && iterations < max) {
      1. Build prompt from history
      2. Call LLM with tools
      3. Parse response
      4. Execute tool calls (parallel if enabled)
      5. Run optimizations (cache check, dedup, reflection)
      6. Add to history
      7. Check if task complete
    }
  - Validate output format
  - Calculate metrics
  - Disconnect MCP servers
  - Return result
*/

// 3. RESULT
result = {
  success: true,
  output: "## ROOT CAUSE IDENTIFIED...",
  structuredOutput: { rootCause: "...", fix: "..." },
  iterationCount: 12,
  toolsUsed: ['read_file', 'search_files', 'git_analysis'],
  metrics: {
    totalCost: 0.0234,
    totalDurationMs: 45000,
    totalTokens: 98234,
  }
}
```

### Message History Structure

Phoenix maintains a conversation history:

```typescript
history: [
  {
    role: 'user',
    content: 'Analyze this defect: Users cannot login after password reset'
  },
  {
    role: 'assistant',
    content: [
      { type: 'text', text: 'Let me search for authentication code.' },
      {
        type: 'tool_use',
        id: 'tool_1',
        name: 'search_files',
        input: { pattern: 'authentication', extensions: ['.ts'] }
      }
    ]
  },
  {
    role: 'user',
    content: [
      {
        type: 'tool_result',
        tool_use_id: 'tool_1',
        content: 'Found files: auth.ts, login.ts, ...'
      }
    ]
  },
  // ... more iterations
]
```

**Context Management** prunes this history when it grows too large.

---

## Agent Specialization Pattern

Specter uses a **composition over inheritance** pattern:

### Pattern Structure

```typescript
// Each agent is a thin wrapper that:
// 1. Defines domain-specific system prompt
// 2. Configures Phoenix with appropriate tools
// 3. Sets optimization levels
// 4. Transforms Phoenix result to agent-specific format

export class MyAgent {
  constructor(private options: AgentOptions) {}

  async execute(task: string): Promise<Result> {
    // Configure Phoenix for this specific task
    const phoenix = new Phoenix({
      taskDescription: task,
      systemPrompt: MY_DOMAIN_PROMPT,
      toolSources: [registryTools([...])],
      repositoryPath: this.options.repositoryPath,
      // ... optimizations
    });

    // Run Phoenix
    const phoenixResult = await phoenix.run();

    // Transform to agent-specific result
    return {
      success: phoenixResult.success,
      domainData: this.parseOutput(phoenixResult),
      metrics: this.extractMetrics(phoenixResult),
    };
  }
}
```

### Why This Pattern?

**Pros**:
- ✅ No code duplication (Phoenix handles all the complexity)
- ✅ Agents are simple, focused wrappers
- ✅ Easy to add new agents (just configure Phoenix differently)
- ✅ All agents benefit from Phoenix improvements
- ✅ Clear separation of concerns

**Cons**:
- ⚠️ Agents can't customize Phoenix internals (by design)
- ⚠️ All agents share same optimization systems (configurable on/off)

### Agent Comparison

| Agent | Tools | Max Iter | Reflection | Checkpoints | Use Case |
|-------|-------|----------|------------|-------------|----------|
| **DefectAgent** | read, search, git | adaptive | ✅ | ✅ | Complex root cause analysis |
| **ExplainAgent** | read, search, list | 15 | ❌ | ❌ | Fast code explanation |
| **ConfigAgent** | read, search | 5 | ❌ | ❌ | Quick config lookup |
| **SecurityAgent** | read, search, list | 15 | ✅ | ❌ | Security vulnerability scan |

---

## Tool System

### Tool Architecture

```
┌──────────────────────────────────────────────┐
│              TOOL EXECUTION                   │
│                                               │
│  Agent requests tool:                        │
│  { name: 'read_file', input: { path: '...' }}│
│                                               │
│                    │                          │
│                    ▼                          │
│            ┌──────────────┐                  │
│            │   Phoenix    │                  │
│            │ Tool Router  │                  │
│            └──────┬───────┘                  │
│                   │                           │
│         ┌─────────┴──────────┐              │
│         │                    │               │
│         ▼                    ▼               │
│  ┌─────────────┐      ┌────────────┐       │
│  │   Registry  │      │    MCP     │       │
│  │    Tool     │      │   Server   │       │
│  └─────┬───────┘      └─────┬──────┘       │
│        │                     │               │
│        ▼                     ▼               │
│  Execute locally      Send JSON-RPC         │
│  (file system)        (HTTP/stdio)          │
│                                               │
│        │                     │               │
│        └─────────┬───────────┘              │
│                  ▼                           │
│           ┌──────────────┐                  │
│           │ Tool Result  │                  │
│           └──────────────┘                  │
└──────────────────────────────────────────────┘
```

### Registry Tools (Built-in)

Located in `src/phoenix/tools/registry/`:

- **read_file**: Read file contents with line numbers
- **search_files**: Ripgrep-based code search
- **list_files**: List directory contents
- **write_file**: Create or modify files
- **git_analysis**: Analyze git history, blame, commits
- **execute_command**: Run shell commands

Each tool:
```typescript
export const read_file: Tool = {
  name: 'read_file',
  description: 'Read contents of a file',
  input_schema: {
    type: 'object',
    properties: {
      path: { type: 'string', description: 'File path' },
      start_line: { type: 'number' },
      end_line: { type: 'number' },
    },
    required: ['path'],
  },
  execute: async (params: any, context: ToolContext) => {
    // Implementation
    return { content: '...', success: true };
  },
};
```

### MCP Tools (External)

MCP servers provide tools from external services:

```typescript
// Example: Jira MCP server provides:
- jira_create_issue
- jira_update_issue
- jira_search_issues
- jira_add_comment

// Phoenix treats them identically to registry tools
```

See [06-MCP-INTEGRATION.md](./06-MCP-INTEGRATION.md) for details.

---

## MCP Integration

### MCP Client Architecture

```
┌─────────────────────────────────────────────────┐
│              MCP CLIENT                          │
│                                                  │
│  Connection Types:                              │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│  │  stdio   │  │   HTTP   │  │WebSocket │     │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘     │
│       │             │              │            │
│       └─────────────┴──────────────┘            │
│                     │                            │
│              JSON-RPC 2.0                       │
│                     │                            │
│       ┌─────────────┴──────────────┐            │
│       │                            │             │
│       ▼                            ▼             │
│  ┌─────────┐                ┌──────────┐       │
│  │ Request │                │ Response │       │
│  │ Handler │                │ Handler  │       │
│  └─────────┘                └──────────┘       │
│                                                  │
│  Events:                                        │
│  - connected                                     │
│  - disconnected                                 │
│  - tool_called                                  │
│  - error                                        │
└─────────────────────────────────────────────────┘
```

### Integration Flow

```typescript
// 1. Configure MCP servers in agent
const phoenix = new Phoenix({
  // ... other options
  toolSources: [
    registryTools(['read_file']),
    mcpTools({
      name: 'jira',
      type: 'http',
      url: 'https://api.example.com/mcp',
      auth: { type: 'bearer', token: process.env.JIRA_TOKEN },
    }, ['jira_create_issue', 'jira_search_issues']),
  ],
});

// 2. Phoenix automatically:
//    - Connects to MCP server
//    - Discovers available tools
//    - Adds to tool list
//    - Routes tool calls appropriately
//    - Disconnects on completion
```

---

## Data Flow

### Complete Request Flow

```
┌──────────┐
│   User   │
└────┬─────┘
     │
     │ agent.analyze("bug description")
     ▼
┌──────────────────┐
│  DefectAgent     │
│  - Creates task  │
│  - Configures    │
│    Phoenix       │
└────┬─────────────┘
     │
     │ phoenix.run()
     ▼
┌──────────────────────────────┐
│     PHOENIX LOOP             │
│                              │
│  Iteration 1:                │
│  ┌────────────────────────┐ │
│  │ 1. Build prompt        │ │
│  │ 2. Call Bedrock        │◄┼── AWS Bedrock API
│  │ 3. Parse response      │ │
│  │ 4. Execute tools       │◄┼── Registry Tools
│  │    - search_files      │ │   or MCP Servers
│  │ 5. Add to history      │ │
│  └────────────────────────┘ │
│                              │
│  Iteration 2:                │
│  ┌────────────────────────┐ │
│  │ 1. Build prompt        │ │
│  │    (with tool results) │ │
│  │ 2. Call Bedrock        │◄┼── AWS Bedrock API
│  │ 3. Parse response      │ │
│  │ 4. Execute tools       │ │
│  │    - read_file         │ │
│  │ 5. Add to history      │ │
│  └────────────────────────┘ │
│                              │
│  ... more iterations ...     │
│                              │
│  Final iteration:            │
│  ┌────────────────────────┐ │
│  │ Agent outputs result   │ │
│  │ Validate output format │ │
│  │ Calculate metrics      │ │
│  └────────────────────────┘ │
└──────────────┬───────────────┘
               │
               │ PhoenixResult
               ▼
┌──────────────────────────────┐
│  DefectAgent                 │
│  - Parse structured output   │
│  - Transform to              │
│    DefectAnalysisResult      │
└────┬─────────────────────────┘
     │
     │ DefectAnalysisResult
     ▼
┌──────────┐
│   User   │
└──────────┘
```

---

## Optimization Systems

### 1. Context Management
**Problem**: LLM context window fills up with history
**Solution**: Intelligently prune old messages, keep important context

### 2. Tool Result Caching
**Problem**: Same tool called multiple times with identical params
**Solution**: Cache results, return instantly

### 3. Parallel Tool Execution
**Problem**: Sequential tool calls are slow
**Solution**: Execute independent tool calls concurrently

### 4. Checkpointing
**Problem**: Long-running tasks can fail mid-execution
**Solution**: Save progress to disk, resume on restart

### 5. Self-Reflection
**Problem**: Agent makes mistakes or misses things
**Solution**: Periodically review own work, self-correct

### 6. Error Recovery
**Problem**: Tool calls fail (network, permissions, etc.)
**Solution**: Auto-retry with exponential backoff

### 7. Adaptive Iterations
**Problem**: Fixed iteration limits are inefficient
**Solution**: Dynamically adjust based on progress

### 8. Token Tracking
**Problem**: Unknown costs until completion
**Solution**: Track tokens and costs in real-time

### 9. Tool Usage Analysis
**Problem**: Agent doesn't know which tool to use next
**Solution**: Analyze patterns, suggest next best tool

### 10. Semantic Deduplication
**Problem**: Agent repeats similar operations
**Solution**: Detect semantically similar tool calls, skip

### 11. Multi-Model Verification
**Problem**: Single model might make mistakes
**Solution**: Cross-check critical decisions with multiple models

### 12. Output Validation
**Problem**: Agent output doesn't match expected format
**Solution**: Validate against JSON schema, retry if invalid

See [08-BEST-PRACTICES.md](./08-BEST-PRACTICES.md) for optimization strategies.

---

## Design Principles

### 1. Separation of Concerns
- **Agents**: Define task domain and configuration
- **Phoenix**: Handle execution and optimizations
- **Tools**: Perform atomic operations
- **MCP**: External service integration

### 2. Composition Over Inheritance
- Agents compose Phoenix, don't extend it
- Phoenix composes optimization systems
- Tool sources are composable

### 3. Configuration Over Code
- Agents configure Phoenix via options
- No need to modify Phoenix internals
- Tool sources defined declaratively

### 4. Fail-Safe Defaults
- Optimizations enabled by default
- Graceful degradation on errors
- Reasonable limits (iterations, costs)

### 5. Observability
- Event emissions at every step
- Detailed metrics tracking
- Progress callbacks

---

## Performance Characteristics

### Typical Agent Performance

| Agent | Iterations | Duration | Cost | Tokens |
|-------|-----------|----------|------|--------|
| DefectAgent (complex) | 15-25 | 30-60s | $0.02-0.05 | 50k-100k |
| ExplainAgent (medium) | 8-15 | 15-30s | $0.01-0.02 | 20k-50k |
| ConfigAgent (fast) | 3-5 | 5-10s | $0.005-0.01 | 5k-15k |

### Bottlenecks

1. **LLM API latency**: 1-3s per call
   - **Mitigation**: Parallel tool execution, caching

2. **Tool execution time**: Variable (file I/O, git, network)
   - **Mitigation**: Parallel execution, caching

3. **Context window limits**: 200k tokens (Claude)
   - **Mitigation**: Context management, pruning

4. **Cost accumulation**: $0.03 per 100k tokens
   - **Mitigation**: Token tracking, adaptive iterations

---

## Deployment Architecture (ECS)

```
┌────────────────────────────────────────────────────┐
│                 AWS ECS CLUSTER                     │
│                                                     │
│  ┌─────────────┐  ┌─────────────┐  ┌───────────┐ │
│  │   Service   │  │   Service   │  │  Service  │ │
│  │   Defect    │  │   Explain   │  │  Config   │ │
│  │   Agent     │  │   Agent     │  │  Agent    │ │
│  └──────┬──────┘  └──────┬──────┘  └─────┬─────┘ │
│         │                 │                │        │
│  ┌──────┴─────────────────┴────────────────┴────┐ │
│  │           Application Load Balancer           │ │
│  └───────────────────────┬───────────────────────┘ │
└────────────────────────────┼───────────────────────┘
                             │
                             ▼
                      ┌────────────┐
                      │   Client   │
                      └────────────┘
```

Each agent runs as a containerized ECS service with:
- Auto-scaling based on load
- Health checks
- CloudWatch logging
- IAM roles for Bedrock access

---

## Summary

**Specter** = Multiple specialized agents
**Phoenix** = Autonomous execution engine with 12 optimizations
**Agents** = Thin wrappers that configure Phoenix
**Tools** = Registry (built-in) + MCP (external)

This architecture enables:
- ✅ Rapid agent development (just configure Phoenix)
- ✅ Consistent quality (all agents use same engine)
- ✅ Easy maintenance (improvements benefit all agents)
- ✅ Flexible deployment (containerized, scalable)

---

**Next**: [03-AGENT-DEVELOPMENT-GUIDE.md](./03-AGENT-DEVELOPMENT-GUIDE.md)
