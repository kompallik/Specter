# Phoenix Configuration Reference

Complete reference for all Phoenix configuration options.

---

## PhoenixOptions Interface

```typescript
interface PhoenixOptions {
  // === REQUIRED OPTIONS ===
  taskId: string;
  taskDescription: string;
  repositoryPath: string;
  systemPrompt: string;
  toolSources: ToolSource[];

  // === OPTIONAL CONFIGURATION ===
  additionalContext?: string;
  outputFormat?: OutputFormat;
  maxIterations?: number;
  model?: string;

  // === CALLBACKS & EVENTS ===
  eventEmitter?: EventEmitter;
  onProgress?: (message: string) => void;

  // === OPTIMIZATION FLAGS ===
  enableContextManagement?: boolean;
  enableCaching?: boolean;
  enableParallelTools?: boolean;
  enableCheckpointing?: boolean;
  enableSelfReflection?: boolean;
  enableErrorRecovery?: boolean;
  enableAdaptiveIterations?: boolean;
  enableToolGuidance?: boolean;
  enableDeduplication?: boolean;
  enableMultiModelVerification?: boolean;

  // === ADVANCED OPTIONS ===
  tokenBudget?: number;
  costLimit?: number;
  timeoutMs?: number;
  checkpointDirectory?: string;
}
```

---

## Required Options

### `taskId: string`
**Description**: Unique identifier for this task execution.

**Purpose**:
- Used for logging and tracking
- Identifies checkpoints
- Correlates events

**Example**:
```typescript
taskId: `defect-${Date.now()}`
taskId: `analysis-${uuid.v4()}`
taskId: 'user-request-123'
```

**Best Practices**:
- Make it unique per execution
- Include timestamp or UUID
- Keep it readable for logs

---

### `taskDescription: string`
**Description**: Human-readable description of the task.

**Purpose**:
- Passed to the LLM as the user's request
- Appears in logs and metrics
- Helps agent understand the goal

**Example**:
```typescript
taskDescription: 'Analyze login functionality and find security vulnerabilities'
taskDescription: 'Explain how the authentication middleware works'
taskDescription: 'Find configuration value for database.connection.timeout'
```

**Best Practices**:
- Be specific and clear
- Include relevant context
- Use imperative form ("Find X", "Analyze Y")
- Keep under 500 characters

---

### `repositoryPath: string`
**Description**: Absolute path to the repository/codebase to analyze.

**Purpose**:
- Used by file system tools (read_file, search_files, etc.)
- Sets working directory for git operations
- Base path for all relative file references

**Example**:
```typescript
repositoryPath: '/Users/username/projects/my-app'
repositoryPath: process.cwd()
repositoryPath: path.join(__dirname, '../..')
```

**Best Practices**:
- Use absolute paths
- Ensure path exists before passing
- Consider using environment variables
- Validate write permissions if using write_file

**Validation**:
```typescript
import fs from 'fs';
import path from 'path';

const repoPath = path.resolve(process.env.REPO_PATH || '.');
if (!fs.existsSync(repoPath)) {
  throw new Error(`Repository path does not exist: ${repoPath}`);
}
```

---

### `systemPrompt: string`
**Description**: Instructions that define the agent's behavior, role, and process.

**Purpose**:
- Sets agent's persona and expertise
- Defines available tools and usage
- Specifies workflow/process
- Declares output format

**Example**:
```typescript
systemPrompt: `You are a security vulnerability scanner.

Your mission: Find security issues in code.

Available tools:
- read_file: Read source files
- search_files: Search for patterns

Process:
1. SEARCH: Find security-sensitive code
2. READ: Analyze each file
3. REPORT: Output JSON findings

Output format:
\`\`\`json
{
  "vulnerabilities": [...],
  "summary": {...}
}
\`\`\`
`
```

**Best Practices**:
- See [03-AGENT-DEVELOPMENT-GUIDE.md](./03-AGENT-DEVELOPMENT-GUIDE.md#system-prompt-design)
- Be explicit about tools
- Define clear process
- Specify exact output format

---

### `toolSources: ToolSource[]`
**Description**: Array of tool sources (registry or MCP) available to the agent.

**Purpose**:
- Defines which tools the agent can use
- Configures MCP server connections
- Controls agent capabilities

**Example**:
```typescript
// Registry tools only
toolSources: [
  registryTools(['read_file', 'search_files', 'list_files'])
]

// Registry + MCP
toolSources: [
  registryTools(['read_file', 'search_files']),
  mcpTools({
    name: 'jira',
    type: 'http',
    url: 'https://api.example.com/mcp',
    auth: { type: 'bearer', token: process.env.JIRA_TOKEN }
  }, ['jira_create_issue', 'jira_search_issues'])
]
```

**Available Registry Tools**:
- `read_file` - Read file contents
- `search_files` - Search codebase with regex
- `list_files` - List directory contents
- `write_file` - Write/modify files
- `git_analysis` - Git history operations
- `execute_command` - Run shell commands

See [05-TOOL-REGISTRY.md](./05-TOOL-REGISTRY.md) for details.

**Best Practices**:
- Only include tools the agent needs
- Order tools by frequency of use
- Use specific tool names, not 'all'
- Consider security implications

---

## Optional Configuration

### `additionalContext?: string`
**Description**: Extra context to provide to the agent.

**Default**: `undefined`

**Use Cases**:
- Stack traces
- Error messages
- Previous analysis results
- User comments

**Example**:
```typescript
additionalContext: `
Stack trace:
Error: Cannot read property 'user' of undefined
  at authenticate (auth.ts:45)
  at handleLogin (login.ts:123)

Error occurs after password reset flow.
`
```

---

### `outputFormat?: OutputFormat`
**Description**: Expected output structure with JSON schema.

**Default**: Free-form text output

**Purpose**:
- Validates agent output
- Enables structured parsing
- Enforces consistency

**Example**:
```typescript
outputFormat: {
  name: 'security_scan',
  description: 'Security vulnerability report',
  schema: {
    type: 'object',
    properties: {
      vulnerabilities: {
        type: 'array',
        items: {
          type: 'object',
          properties: {
            type: { type: 'string' },
            severity: { type: 'string' },
            file: { type: 'string' },
            line: { type: 'number' },
          },
          required: ['type', 'severity', 'file'],
        },
      },
      summary: {
        type: 'object',
        properties: {
          critical: { type: 'number' },
          high: { type: 'number' },
        },
      },
    },
    required: ['vulnerabilities', 'summary'],
  },
}
```

See [07-OUTPUT-FORMATS.md](./07-OUTPUT-FORMATS.md) for details.

---

### `maxIterations?: number`
**Description**: Maximum number of agent loop iterations.

**Default**: `20`

**Special Values**:
- `0` - Adaptive (no fixed limit, uses AdaptiveIterationManager)
- Positive number - Fixed limit

**Use Cases**:
- `5` - Fast lookups
- `15` - Medium complexity
- `25` - Complex analysis
- `0` - Let agent decide (adaptive)

**Example**:
```typescript
maxIterations: 5    // Fast config lookup
maxIterations: 15   // Code explanation
maxIterations: 25   // Defect analysis
maxIterations: 0    // Adaptive (complex tasks)
```

**Cost Impact**: More iterations = higher cost (~$0.001-0.003 per iteration)

---

### `model?: string`
**Description**: LLM model identifier to use.

**Default**: `'us.anthropic.claude-sonnet-4-5-20250929-v1:0'`

**Available Models**:
- `us.anthropic.claude-sonnet-4-5-20250929-v1:0` (default, best balance)
- `us.anthropic.claude-opus-4-5-...` (most capable, expensive)
- `us.anthropic.claude-haiku-...` (fast, cheap, less capable)

**Example**:
```typescript
model: 'us.anthropic.claude-sonnet-4-5-20250929-v1:0'  // Default
model: 'us.anthropic.claude-opus-4-5-...'              // For critical tasks
model: 'us.anthropic.claude-haiku-...'                  // For simple tasks
```

---

## Callbacks & Events

### `eventEmitter?: EventEmitter`
**Description**: Node.js EventEmitter for receiving Phoenix events.

**Default**: `undefined`

**Available Events**:
- `iteration:start` - Loop iteration starting
- `iteration:complete` - Loop iteration finished
- `tool:execute` - Tool execution starting
- `tool:complete` - Tool execution finished
- `tool:error` - Tool execution failed
- `llm:request` - LLM request sent
- `llm:response` - LLM response received
- `checkpoint:save` - Checkpoint saved
- `checkpoint:load` - Checkpoint loaded
- `reflection:start` - Self-reflection starting
- `reflection:complete` - Self-reflection finished
- `mcp:connect` - MCP server connected
- `mcp:disconnect` - MCP server disconnected
- `complete` - Task completed
- `error` - Task failed

**Example**:
```typescript
import { EventEmitter } from 'events';

const emitter = new EventEmitter();

emitter.on('iteration:start', ({ iteration, maxIterations }) => {
  console.log(`Iteration ${iteration}/${maxIterations}`);
});

emitter.on('tool:execute', ({ toolName, params }) => {
  console.log(`Executing: ${toolName}`, params);
});

emitter.on('complete', ({ success, metrics }) => {
  console.log('Task complete!', metrics);
});

const phoenix = new Phoenix({
  // ...
  eventEmitter: emitter,
});
```

---

### `onProgress?: (message: string) => void`
**Description**: Simple callback for progress messages.

**Default**: `undefined`

**Purpose**: Simplified alternative to eventEmitter for basic progress tracking.

**Example**:
```typescript
onProgress: (message) => {
  console.log(`[Phoenix] ${message}`);
  // or send to UI, log to file, etc.
}
```

**Typical Messages**:
- `"Starting iteration 1/20"`
- `"Executing tool: search_files"`
- `"Self-reflection: Reviewing progress"`
- `"Task complete: 12 iterations, $0.023 cost"`

---

## Optimization Flags

All optimization flags are `boolean` and default to `true` unless otherwise noted.

### `enableContextManagement?: boolean`
**Description**: Intelligently prune conversation history when it grows too large.

**Default**: `true`

**How It Works**:
- Monitors message history size
- Removes old tool results when context fills up
- Keeps recent messages and important context
- Prevents hitting LLM context limits

**When to Disable**: Never (always beneficial)

**Performance Impact**: Prevents context overflow errors, minimal overhead

---

### `enableCaching?: boolean`
**Description**: Cache tool results to avoid redundant tool calls.

**Default**: `true`

**How It Works**:
- Hashes tool name + parameters
- Returns cached result if identical call made before
- Saves time and cost

**When to Disable**:
- Tool has side effects (write_file, MCP calls that modify state)
- Results change over time (time-sensitive queries)

**Performance Impact**: Significant speed improvement (skip tool execution), no cost for cached calls

**Example**:
```typescript
// First call: executes and caches
read_file({ path: 'src/auth.ts' }) → reads file

// Second identical call: returns cached result
read_file({ path: 'src/auth.ts' }) → instant return from cache
```

---

### `enableParallelTools?: boolean`
**Description**: Execute independent tool calls concurrently.

**Default**: `true`

**How It Works**:
- Analyzes tool calls in LLM response
- Identifies independent tools (no dependencies)
- Executes them in parallel with Promise.all()
- Waits for all to complete before continuing

**When to Disable**:
- Tools have dependencies (one needs result from another)
- Tools modify shared state (write operations)
- Fast tasks where overhead isn't worth it

**Performance Impact**: 2-5x faster for multi-tool iterations

**Example**:
```typescript
// Sequential (disabled):
await read_file('auth.ts')      // 100ms
await read_file('login.ts')     // 100ms
await read_file('user.ts')      // 100ms
// Total: 300ms

// Parallel (enabled):
await Promise.all([
  read_file('auth.ts'),
  read_file('login.ts'),
  read_file('user.ts'),
])
// Total: ~100ms
```

---

### `enableCheckpointing?: boolean`
**Description**: Save progress to disk, enabling resume on failure.

**Default**: `true` for write operations, `false` for read-only

**How It Works**:
- Periodically saves conversation history to disk
- On crash/restart, can resume from last checkpoint
- Cleans up checkpoints on successful completion

**When to Enable**:
- Long-running tasks (>30s)
- Tasks with write operations
- Tasks with high failure risk (network, external APIs)

**When to Disable**:
- Fast tasks (<10s)
- Read-only operations
- Local testing

**Performance Impact**: Slight overhead (~50-100ms per checkpoint), disk I/O

**Checkpoint Location**: `{checkpointDirectory}/{taskId}.json`

---

### `enableSelfReflection?: boolean`
**Description**: Periodically pause and review progress, self-correct if needed.

**Default**: `true` for complex tasks

**How It Works**:
- Every N iterations (configurable), pauses
- Asks LLM: "Are we making progress? What should we do differently?"
- Can change strategy based on reflection

**When to Enable**:
- Complex analysis (defect investigation)
- Tasks that may go off track
- Multi-step workflows

**When to Disable**:
- Simple lookups
- Fast queries
- Well-defined processes

**Performance Impact**: Adds 1-2 extra LLM calls, ~2-5s overhead, ~$0.002 cost

**Example**:
```
Iteration 5: (reflection triggered)
Agent: "I've been searching for authentication code but found nothing.
        I should search for 'auth' instead of 'authentication'."
(Changes search strategy)
```

---

### `enableErrorRecovery?: boolean`
**Description**: Automatically retry failed tool calls with exponential backoff.

**Default**: `true`

**How It Works**:
- Catches tool execution errors
- Retries up to 3 times with backoff (1s, 2s, 4s)
- Logs errors
- Returns error message to agent on final failure

**When to Disable**: Never (always beneficial)

**Performance Impact**: Only adds time on errors, no overhead on success

**Example**:
```
Tool: read_file('auth.ts')
Error: ENOENT (file not found)
Retry 1 after 1s: still fails
Retry 2 after 2s: still fails
Retry 3 after 4s: still fails
Return error to agent: "File not found: auth.ts"
```

---

### `enableAdaptiveIterations?: boolean`
**Description**: Dynamically adjust iteration limit based on progress.

**Default**: `true`

**How It Works**:
- Monitors progress indicators:
  - New tools used
  - Output format approaching completion
  - Repeated tool calls (stuck)
- Increases limit if making progress
- Decreases if stuck/spinning

**When to Enable**:
- Uncertain task complexity
- Want agent to decide when to stop
- Complex, variable tasks

**When to Disable**:
- Fixed iteration budget (cost control)
- Simple, predictable tasks
- Testing/debugging

**Performance Impact**: Can increase or decrease total iterations, improves efficiency

---

### `enableToolGuidance?: boolean`
**Description**: Analyze tool usage patterns and suggest next best tool.

**Default**: `true`

**How It Works**:
- Tracks which tools are used when
- Learns patterns (e.g., "search_files usually followed by read_file")
- Provides hints to agent in prompt
- Agent can choose to follow or ignore

**When to Enable**:
- Agent seems uncertain
- Complex tool interactions
- Want to guide toward optimal path

**When to Disable**:
- Simple single-tool tasks
- Agent already efficient
- Want agent to explore freely

**Performance Impact**: Minimal overhead, can reduce iterations by suggesting right tool

---

### `enableDeduplication?: boolean`
**Description**: Detect and skip semantically similar tool calls.

**Default**: `true`

**How It Works**:
- Uses semantic similarity (embeddings) to compare tool calls
- Detects calls that are essentially the same despite different wording
- Skips duplicate, returns previous result

**When to Enable**: Always (always beneficial)

**When to Disable**:
- Tool results change over time
- Debugging (want to see all calls)

**Performance Impact**: Prevents wasted calls, slight overhead for embedding generation

**Example**:
```typescript
// Call 1: search_files({ pattern: 'authentication' })
// Call 2: search_files({ pattern: 'auth' })
// Detected as similar, skips second call
```

---

### `enableMultiModelVerification?: boolean`
**Description**: Cross-check critical outputs with multiple LLM models.

**Default**: `false` (⚠️ expensive!)

**How It Works**:
- Runs final output through 2-3 different models
- Compares results for consistency
- Flags discrepancies

**When to Enable**:
- Critical business decisions
- High-risk operations (financial, security)
- Need high confidence

**When to Disable**:
- Cost-sensitive applications
- Fast tasks
- Low-risk operations

**Performance Impact**: 2-3x cost, 2-3x time for final validation

---

## Advanced Options

### `tokenBudget?: number`
**Description**: Maximum tokens to use (input + output combined).

**Default**: `undefined` (no limit)

**Use Cases**:
- Cost control
- Testing with limited budget
- Prevent runaway costs

**Example**:
```typescript
tokenBudget: 50000  // Stop after 50k tokens (~$0.015)
```

**Behavior**: Phoenix stops when budget exceeded, returns partial result.

---

### `costLimit?: number`
**Description**: Maximum cost in USD.

**Default**: `undefined` (no limit)

**Use Cases**:
- Strict budget enforcement
- Production cost controls
- Prevent accidents

**Example**:
```typescript
costLimit: 0.10  // Stop after $0.10
```

**Behavior**: Phoenix stops when cost limit exceeded.

---

### `timeoutMs?: number`
**Description**: Maximum execution time in milliseconds.

**Default**: `undefined` (no limit)

**Use Cases**:
- API timeout requirements
- Prevent hung tasks
- Resource management

**Example**:
```typescript
timeoutMs: 60000  // 60 second timeout
```

**Behavior**: Phoenix stops after timeout, returns partial result or error.

---

### `checkpointDirectory?: string`
**Description**: Directory to store checkpoint files.

**Default**: `'./checkpoints'`

**Example**:
```typescript
checkpointDirectory: '/tmp/phoenix-checkpoints'
checkpointDirectory: path.join(__dirname, '../.checkpoints')
```

**Requirements**: Directory must be writable.

---

## Configuration Presets

### Fast Preset (5s, $0.005)
```typescript
const FAST_CONFIG: Partial<PhoenixOptions> = {
  maxIterations: 5,
  enableContextManagement: true,
  enableCaching: true,
  enableParallelTools: false,
  enableCheckpointing: false,
  enableSelfReflection: false,
  enableErrorRecovery: true,
  enableAdaptiveIterations: false,
  enableToolGuidance: false,
  enableDeduplication: true,
  enableMultiModelVerification: false,
};
```

### Balanced Preset (20s, $0.015)
```typescript
const BALANCED_CONFIG: Partial<PhoenixOptions> = {
  maxIterations: 15,
  enableContextManagement: true,
  enableCaching: true,
  enableParallelTools: true,
  enableCheckpointing: false,
  enableSelfReflection: false,
  enableErrorRecovery: true,
  enableAdaptiveIterations: true,
  enableToolGuidance: true,
  enableDeduplication: true,
  enableMultiModelVerification: false,
};
```

### Thorough Preset (60s, $0.04)
```typescript
const THOROUGH_CONFIG: Partial<PhoenixOptions> = {
  maxIterations: 0, // Adaptive
  enableContextManagement: true,
  enableCaching: true,
  enableParallelTools: true,
  enableCheckpointing: true,
  enableSelfReflection: true,
  enableErrorRecovery: true,
  enableAdaptiveIterations: true,
  enableToolGuidance: true,
  enableDeduplication: true,
  enableMultiModelVerification: false,
};
```

---

## Complete Example

```typescript
import { Phoenix, registryTools, mcpTools, OutputFormats } from './phoenix/index.js';
import { EventEmitter } from 'events';

const emitter = new EventEmitter();
emitter.on('tool:execute', ({ toolName }) => console.log(`→ ${toolName}`));

const phoenix = new Phoenix({
  // Required
  taskId: `analysis-${Date.now()}`,
  taskDescription: 'Find all TODO comments in the codebase',
  repositoryPath: '/path/to/repo',
  systemPrompt: `You are a TODO finder.

  Process:
  1. Use search_files to find all TODO comments
  2. Use read_file to get context around each TODO
  3. Output structured list`,

  toolSources: [
    registryTools(['read_file', 'search_files']),
  ],

  // Optional
  outputFormat: {
    name: 'todo_list',
    schema: {
      type: 'object',
      properties: {
        todos: {
          type: 'array',
          items: {
            type: 'object',
            properties: {
              file: { type: 'string' },
              line: { type: 'number' },
              text: { type: 'string' },
              context: { type: 'string' },
            },
          },
        },
      },
    },
  },

  maxIterations: 10,

  eventEmitter: emitter,
  onProgress: (msg) => console.log(`[Progress] ${msg}`),

  // Optimizations
  enableContextManagement: true,
  enableCaching: true,
  enableParallelTools: true,
  enableCheckpointing: false,
  enableSelfReflection: false,
  enableErrorRecovery: true,
  enableAdaptiveIterations: true,
  enableToolGuidance: true,
  enableDeduplication: true,
  enableMultiModelVerification: false,

  // Advanced
  tokenBudget: 100000,
  costLimit: 0.05,
  timeoutMs: 120000,
});

const result = await phoenix.run();
console.log('TODOs found:', result.structuredOutput);
```

---

**Next**: [05-TOOL-REGISTRY.md](./05-TOOL-REGISTRY.md)
