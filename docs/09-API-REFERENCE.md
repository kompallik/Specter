# API Reference

Complete API documentation for Specter and Phoenix.

---

## Core Exports

```typescript
import {
  // Agents
  DefectAgent,
  ExplainAgent,
  ConfigAgent,

  // Phoenix Engine
  Phoenix,
  type PhoenixOptions,
  type PhoenixResult,

  // Configuration Helpers
  registryTools,
  mcpTools,
  OutputFormats,

  // MCP Client
  MCPClient,

  // Utilities
  createLogger,
} from 'specter';
```

---

## Phoenix Class

### Constructor

```typescript
new Phoenix(options: PhoenixOptions)
```

Creates a new Phoenix instance with the provided configuration.

**Parameters**:
- `options`: PhoenixOptions - Configuration object

**Example**:
```typescript
const phoenix = new Phoenix({
  taskId: 'task-123',
  taskDescription: 'Analyze authentication code',
  repositoryPath: '/path/to/repo',
  systemPrompt: SYSTEM_PROMPT,
  toolSources: [registryTools(['read_file', 'search_files'])],
});
```

---

### run()

```typescript
async run(): Promise<PhoenixResult>
```

Executes the agent task and returns the result.

**Returns**: `Promise<PhoenixResult>`

**Example**:
```typescript
const result = await phoenix.run();

if (result.success) {
  console.log('Output:', result.output);
  console.log('Cost:', result.metrics.totalCost);
}
```

---

## PhoenixOptions Interface

Complete configuration options for Phoenix.

```typescript
interface PhoenixOptions {
  // === REQUIRED ===
  taskId: string;
  taskDescription: string;
  repositoryPath: string;
  systemPrompt: string;
  toolSources: ToolSource[];

  // === OPTIONAL ===
  additionalContext?: string;
  outputFormat?: OutputFormat;
  maxIterations?: number;  // Default: 20
  model?: string;  // Default: Sonnet 4.5

  // === CALLBACKS ===
  eventEmitter?: EventEmitter;
  onProgress?: (message: string) => void;

  // === OPTIMIZATIONS ===
  enableContextManagement?: boolean;  // Default: true
  enableCaching?: boolean;  // Default: true
  enableParallelTools?: boolean;  // Default: true
  enableCheckpointing?: boolean;  // Default: true
  enableSelfReflection?: boolean;  // Default: true
  enableErrorRecovery?: boolean;  // Default: true
  enableAdaptiveIterations?: boolean;  // Default: true
  enableToolGuidance?: boolean;  // Default: true
  enableDeduplication?: boolean;  // Default: true
  enableMultiModelVerification?: boolean;  // Default: false

  // === ADVANCED ===
  tokenBudget?: number;
  costLimit?: number;
  timeoutMs?: number;
  checkpointDirectory?: string;  // Default: './checkpoints'
}
```

See [04-PHOENIX-CONFIGURATION.md](./04-PHOENIX-CONFIGURATION.md) for detailed descriptions.

---

## PhoenixResult Interface

Result returned by `phoenix.run()`.

```typescript
interface PhoenixResult {
  // Execution status
  success: boolean;
  error?: string;

  // Output
  output: string;  // Raw text output
  structuredOutput?: any;  // Parsed JSON (if outputFormat provided)

  // Metrics
  iterationCount: number;
  toolsUsed: string[];
  metrics: {
    totalCost: number;  // USD
    totalDurationMs: number;
    totalTokens: number;
    inputTokens: number;
    outputTokens: number;
  };

  // History
  conversationHistory: Message[];
}
```

**Example**:
```typescript
const result = await phoenix.run();

// Check success
if (result.success) {
  // Access outputs
  console.log('Text:', result.output);
  console.log('Structured:', result.structuredOutput);

  // Check metrics
  console.log('Iterations:', result.iterationCount);
  console.log('Cost:', result.metrics.totalCost);
  console.log('Duration:', result.metrics.totalDurationMs);
  console.log('Tools:', result.toolsUsed);
} else {
  console.error('Error:', result.error);
}
```

---

## Agent Classes

### DefectAgent

```typescript
class DefectAgent {
  constructor(options: DefectAgentOptions)

  async analyze(
    defectDescription: string,
    additionalContext?: string
  ): Promise<DefectAnalysisResult>

  async quickTriage(
    defectDescription: string
  ): Promise<DefectAnalysisResult>
}
```

**DefectAgentOptions**:
```typescript
interface DefectAgentOptions {
  repositoryPath: string;
  eventEmitter?: EventEmitter;
  onProgress?: (message: string) => void;
  maxIterations?: number;  // Default: 0 (adaptive)
  fullOptimizations?: boolean;  // Default: true
}
```

**DefectAnalysisResult**:
```typescript
interface DefectAnalysisResult {
  success: boolean;
  analysis: string;
  structured?: {
    rootCause?: string;
    problematicCode?: string;
    filePath?: string;
    commitHash?: string;
    proposedFix?: string;
  };
  metrics: {
    iterations: number;
    toolsUsed: string[];
    cost: number;
    durationMs: number;
  };
  error?: string;
}
```

**Example**:
```typescript
const agent = new DefectAgent({
  repositoryPath: '/repo',
  onProgress: (msg) => console.log(msg),
});

const result = await agent.analyze('Login fails after password reset');

if (result.success) {
  console.log('Root cause:', result.structured?.rootCause);
  console.log('Fix:', result.structured?.proposedFix);
}
```

---

### ExplainAgent

```typescript
class ExplainAgent {
  constructor(options: ExplainAgentOptions)

  async explain(question: string): Promise<ExplanationResult>
}
```

**ExplainAgentOptions**:
```typescript
interface ExplainAgentOptions {
  repositoryPath: string;
  eventEmitter?: EventEmitter;
  onProgress?: (message: string) => void;
  maxIterations?: number;  // Default: 15
}
```

**ExplanationResult**:
```typescript
interface ExplanationResult {
  success: boolean;
  explanation: string;
  metrics: {
    iterations: number;
    toolsUsed: string[];
    cost: number;
    durationMs: number;
  };
  error?: string;
}
```

**Example**:
```typescript
const agent = new ExplainAgent({ repositoryPath: '/repo' });

const result = await agent.explain('How does authentication work?');

if (result.success) {
  console.log(result.explanation);
}
```

---

### ConfigAgent

```typescript
class ConfigAgent {
  constructor(options: ConfigAgentOptions)

  async lookup(configKey: string): Promise<ConfigLookupResult>

  async findAll(pattern: string): Promise<ConfigLookupResult>
}
```

**ConfigAgentOptions**:
```typescript
interface ConfigAgentOptions {
  repositoryPath: string;
  eventEmitter?: EventEmitter;
  onProgress?: (message: string) => void;
}
```

**ConfigLookupResult**:
```typescript
interface ConfigLookupResult {
  success: boolean;
  found: boolean;
  config?: {
    key: string;
    value: string;
    file: string;
    line?: number;
    description?: string;
    type?: string;
    defaultValue?: string;
  };
  searchInfo?: {
    searched: string[];
    filesChecked: string[];
    suggestions: string[];
  };
  metrics: {
    iterations: number;
    toolsUsed: string[];
    cost: number;
    durationMs: number;
  };
  error?: string;
}
```

**Example**:
```typescript
const agent = new ConfigAgent({ repositoryPath: '/repo' });

const result = await agent.lookup('database.connection.timeout');

if (result.found) {
  console.log(`${result.config.key} = ${result.config.value}`);
  console.log(`Location: ${result.config.file}:${result.config.line}`);
}
```

---

## Configuration Helpers

### registryTools()

```typescript
function registryTools(toolNames: string[]): ToolSource
```

Creates a tool source from built-in registry tools.

**Parameters**:
- `toolNames`: Array of tool names to include

**Available Tools**:
- `'read_file'`
- `'write_file'`
- `'search_files'`
- `'list_files'`
- `'git_analysis'`
- `'execute_command'`

**Example**:
```typescript
toolSources: [
  registryTools(['read_file', 'search_files', 'git_analysis'])
]
```

---

### mcpTools()

```typescript
function mcpTools(
  config: MCPServerConfig,
  toolNames?: string[]
): ToolSource
```

Creates a tool source from an MCP server.

**Parameters**:
- `config`: MCP server configuration
- `toolNames`: Optional array of specific tools to use (if omitted, uses all)

**MCPServerConfig**:
```typescript
interface MCPServerConfig {
  name: string;
  type: 'http' | 'stdio' | 'websocket';

  // HTTP/WebSocket
  url?: string;
  timeout?: number;

  // stdio
  command?: string;
  args?: string[];
  env?: Record<string, string>;

  // Authentication
  auth?: {
    type: 'bearer' | 'basic' | 'api-key';
    token?: string;
    username?: string;
    password?: string;
    headerName?: string;
    key?: string;
  };
}
```

**Example**:
```typescript
toolSources: [
  registryTools(['read_file']),
  mcpTools({
    name: 'jira',
    type: 'http',
    url: 'https://jira.example.com/mcp',
    auth: {
      type: 'bearer',
      token: process.env.JIRA_TOKEN,
    },
  }, ['jira_create_issue', 'jira_search_issues']),
]
```

---

### OutputFormats

```typescript
enum OutputFormats {
  DEFECT_ANALYSIS,
  EXPLANATION,
  CONFIG_LOOKUP,
  SECURITY_SCAN,
  REFACTORING_PLAN,
  TEST_GENERATION,
  DOCUMENTATION,
}
```

Pre-defined output format schemas.

**Example**:
```typescript
outputFormat: OutputFormats.DEFECT_ANALYSIS
```

---

## MCPClient Class

Low-level MCP client for direct integration.

```typescript
class MCPClient extends EventEmitter {
  constructor(config: MCPServerConfig)

  async connect(): Promise<void>

  async disconnect(): Promise<void>

  async getTools(toolNames?: string[]): Promise<ToolDefinition[]>

  async executeTool(
    toolName: string,
    params: Record<string, unknown>
  ): Promise<ToolResult>

  isConnected(): boolean
}
```

**Events**:
- `'connected'`
- `'disconnected'`
- `'tool_called'`
- `'error'`

**Example**:
```typescript
const client = new MCPClient({
  name: 'jira',
  type: 'http',
  url: 'https://jira.example.com/mcp',
  auth: { type: 'bearer', token: process.env.JIRA_TOKEN },
});

client.on('connected', () => console.log('Connected'));
client.on('error', (err) => console.error('Error:', err));

await client.connect();

const tools = await client.getTools();
console.log('Available tools:', tools);

const result = await client.executeTool('jira_create_issue', {
  projectKey: 'PROJ',
  summary: 'Test issue',
});

await client.disconnect();
```

---

## Events

### Phoenix Events

When using `eventEmitter`, Phoenix emits:

```typescript
// Iteration events
'iteration:start'     { iteration: number, maxIterations: number }
'iteration:complete'  { iteration: number, toolsUsed: string[] }

// Tool events
'tool:execute'        { toolName: string, params: any, source: string }
'tool:complete'       { toolName: string, result: any, durationMs: number }
'tool:error'          { toolName: string, error: string }

// LLM events
'llm:request'         { tokens: number, cost: number }
'llm:response'        { tokens: number, cost: number, durationMs: number }

// Optimization events
'checkpoint:save'     { taskId: string, iteration: number }
'checkpoint:load'     { taskId: string, iteration: number }
'reflection:start'    { iteration: number }
'reflection:complete' { decision: string }

// MCP events
'mcp:connect'         { server: string, success: boolean }
'mcp:disconnect'      { server: string }
'mcp:tool_called'     { server: string, tool: string, durationMs: number }

// Completion
'complete'            { success: boolean, metrics: Metrics }
'error'               { error: string }
```

**Example**:
```typescript
const emitter = new EventEmitter();

emitter.on('tool:execute', ({ toolName, params }) => {
  console.log(`Executing: ${toolName}`, params);
});

emitter.on('llm:response', ({ cost, durationMs }) => {
  console.log(`LLM call: $${cost.toFixed(4)} in ${durationMs}ms`);
});

const phoenix = new Phoenix({
  // ...
  eventEmitter: emitter,
});
```

---

## Utilities

### createLogger()

```typescript
function createLogger(options?: {
  level?: 'error' | 'warn' | 'info' | 'debug';
  pretty?: boolean;
}): Logger
```

Creates a configured logger instance.

**Example**:
```typescript
import { createLogger } from 'specter';

const logger = createLogger({
  level: 'debug',
  pretty: true,
});

logger.info('Starting agent');
logger.debug('Debug details', { data: '...' });
logger.error('Error occurred', error);
```

---

## Type Exports

```typescript
// Phoenix types
export type {
  PhoenixOptions,
  PhoenixResult,
  PhoenixEvents,
};

// Configuration types
export type {
  ToolSource,
  OutputFormat,
  MCPServerConfig,
};

// Agent types
export type {
  DefectAgentOptions,
  DefectAnalysisResult,
  ExplainAgentOptions,
  ExplanationResult,
  ConfigAgentOptions,
  ConfigLookupResult,
};
```

---

## Constants

### VERSION

```typescript
export const VERSION: string
```

Current Specter version.

**Example**:
```typescript
import { VERSION } from 'specter';
console.log(`Specter v${VERSION}`);
```

---

## Error Handling

All async methods may throw errors. Use try-catch:

```typescript
try {
  const result = await phoenix.run();
  // Handle result
} catch (error) {
  if (error instanceof TimeoutError) {
    console.error('Task timed out');
  } else if (error instanceof CostLimitError) {
    console.error('Cost limit exceeded');
  } else {
    console.error('Unexpected error:', error);
  }
}
```

---

## TypeScript Support

Specter is written in TypeScript and includes full type definitions.

```typescript
import type { PhoenixOptions, PhoenixResult } from 'specter';

function createAgent(options: PhoenixOptions): Phoenix {
  return new Phoenix(options);
}

async function runAgent(phoenix: Phoenix): Promise<PhoenixResult> {
  return await phoenix.run();
}
```

---

## Complete Example

```typescript
import {
  Phoenix,
  registryTools,
  mcpTools,
  OutputFormats,
  createLogger,
} from 'specter';
import { EventEmitter } from 'events';

// Setup logging
const logger = createLogger({ level: 'debug' });

// Setup events
const emitter = new EventEmitter();
emitter.on('tool:execute', ({ toolName }) => {
  logger.info(`Executing tool: ${toolName}`);
});

// Create Phoenix instance
const phoenix = new Phoenix({
  taskId: `task-${Date.now()}`,
  taskDescription: 'Analyze authentication flow',
  repositoryPath: '/path/to/repo',
  systemPrompt: `You are a code analyst...`,

  toolSources: [
    registryTools(['read_file', 'search_files', 'git_analysis']),
    mcpTools({
      name: 'jira',
      type: 'http',
      url: process.env.JIRA_URL!,
      auth: {
        type: 'bearer',
        token: process.env.JIRA_TOKEN!,
      },
    }),
  ],

  outputFormat: OutputFormats.DEFECT_ANALYSIS,

  maxIterations: 20,
  costLimit: 0.10,

  eventEmitter: emitter,
  onProgress: (msg) => logger.info(msg),

  enableContextManagement: true,
  enableCaching: true,
  enableParallelTools: true,
  enableErrorRecovery: true,
});

// Run agent
try {
  const result = await phoenix.run();

  if (result.success) {
    logger.info('Analysis complete');
    console.log('Output:', result.output);
    console.log('Cost:', `$${result.metrics.totalCost.toFixed(4)}`);
    console.log('Duration:', `${result.metrics.totalDurationMs}ms`);
  } else {
    logger.error('Analysis failed:', result.error);
  }
} catch (error) {
  logger.error('Unexpected error:', error);
  throw error;
}
```

---

**Documentation Complete** ✅

See other guides:
- [Getting Started](./01-GETTING-STARTED.md)
- [Architecture](./02-ARCHITECTURE.md)
- [Agent Development](./03-AGENT-DEVELOPMENT-GUIDE.md)
- [Best Practices](./08-BEST-PRACTICES.md)
