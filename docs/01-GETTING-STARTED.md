# Getting Started with Specter

**Goal**: Build your first Specter agent in 10 minutes.

---

## Prerequisites

- Node.js 18+ installed
- TypeScript knowledge
- AWS credentials configured (for Bedrock)
- Git repository to analyze

---

## Installation

```bash
# Clone or navigate to Specter project
cd /path/to/Specter

# Install dependencies
npm install

# Build the project
npm run build
```

---

## Your First Agent in 5 Steps

Let's build a **SecurityAgent** that scans code for security vulnerabilities.

### Step 1: Create Agent File

Create `src/agents/SecurityAgent.ts`:

```typescript
/**
 * SecurityAgent - Scans code for security vulnerabilities
 */

import { Phoenix, registryTools, OutputFormats, type PhoenixResult } from '../phoenix/index.js';
import type { EventEmitter } from 'events';

const SECURITY_SCAN_PROMPT = `You are a security analysis specialist.

Your mission: Identify security vulnerabilities in code.

Available tools:
- read_file: Read source code files
- search_files: Search for security patterns

Process:
1. SEARCH: Use search_files to find security-sensitive code (auth, crypto, SQL, etc.)
2. READ: Use read_file to examine implementations
3. ANALYZE: Check for common vulnerabilities (OWASP Top 10)
4. REPORT: List findings with severity levels

Output format (JSON):
## SECURITY SCAN RESULTS:
\`\`\`json
{
  "vulnerabilities": [
    {
      "type": "SQL Injection",
      "severity": "HIGH",
      "file": "src/db/queries.ts",
      "line": 45,
      "description": "Unparameterized SQL query",
      "recommendation": "Use parameterized queries"
    }
  ],
  "summary": {
    "critical": 0,
    "high": 1,
    "medium": 2,
    "low": 3
  }
}
\`\`\`
`;

export interface SecurityAgentOptions {
  repositoryPath: string;
  eventEmitter?: EventEmitter;
  onProgress?: (message: string) => void;
}

export interface SecurityScanResult {
  success: boolean;
  vulnerabilities: Array<{
    type: string;
    severity: string;
    file: string;
    line?: number;
    description: string;
    recommendation: string;
  }>;
  summary: {
    critical: number;
    high: number;
    medium: number;
    low: number;
  };
  metrics: {
    iterations: number;
    cost: number;
    durationMs: number;
  };
  error?: string;
}

export class SecurityAgent {
  private options: SecurityAgentOptions;

  constructor(options: SecurityAgentOptions) {
    this.options = options;
  }

  async scan(): Promise<SecurityScanResult> {
    const taskId = `security-${Date.now()}`;

    const phoenix = new Phoenix({
      taskId,
      taskDescription: 'Scan codebase for security vulnerabilities',
      repositoryPath: this.options.repositoryPath,
      systemPrompt: SECURITY_SCAN_PROMPT,

      // Read-only tools (no modification)
      toolSources: [
        registryTools(['read_file', 'search_files', 'list_files']),
      ],

      outputFormat: {
        name: 'security_scan',
        description: 'Security vulnerability scan results',
        schema: {
          type: 'object',
          properties: {
            vulnerabilities: { type: 'array' },
            summary: { type: 'object' },
          },
        },
      },

      eventEmitter: this.options.eventEmitter,
      onProgress: this.options.onProgress,

      maxIterations: 15,

      // Optimizations for thorough analysis
      enableContextManagement: true,
      enableCaching: true,
      enableParallelTools: true,
      enableCheckpointing: false,
      enableSelfReflection: true,
      enableErrorRecovery: true,
      enableAdaptiveIterations: true,
      enableToolGuidance: true,
      enableDeduplication: true,
      enableMultiModelVerification: false,
    });

    const result = await phoenix.run();

    const scanData = result.structuredOutput as any;

    return {
      success: result.success,
      vulnerabilities: scanData?.vulnerabilities || [],
      summary: scanData?.summary || { critical: 0, high: 0, medium: 0, low: 0 },
      metrics: {
        iterations: result.iterationCount,
        cost: result.metrics.totalCost,
        durationMs: result.metrics.totalDurationMs,
      },
      error: result.error,
    };
  }
}
```

### Step 2: Export Your Agent

Add to `src/agents/index.ts`:

```typescript
export { SecurityAgent, type SecurityAgentOptions, type SecurityScanResult } from './SecurityAgent.js';

// Update AGENT_REGISTRY
export const AGENT_REGISTRY = {
  defect: 'DefectAgent',
  explain: 'ExplainAgent',
  config: 'ConfigAgent',
  security: 'SecurityAgent', // Add this line
} as const;
```

### Step 3: Build

```bash
npm run build
```

### Step 4: Use Your Agent

Create `examples/security-scan.ts`:

```typescript
import { SecurityAgent } from '../dist/agents/index.js';

async function main() {
  const agent = new SecurityAgent({
    repositoryPath: '/path/to/your/repo',
    onProgress: (message) => console.log('Progress:', message),
  });

  console.log('Starting security scan...');
  const result = await agent.scan();

  if (result.success) {
    console.log('\n=== Security Scan Results ===');
    console.log(`Found ${result.vulnerabilities.length} vulnerabilities`);
    console.log('Summary:', result.summary);

    result.vulnerabilities.forEach((vuln, i) => {
      console.log(`\n${i + 1}. [${vuln.severity}] ${vuln.type}`);
      console.log(`   File: ${vuln.file}:${vuln.line}`);
      console.log(`   Issue: ${vuln.description}`);
      console.log(`   Fix: ${vuln.recommendation}`);
    });

    console.log(`\nMetrics: ${result.metrics.iterations} iterations, $${result.metrics.cost.toFixed(4)} cost`);
  } else {
    console.error('Scan failed:', result.error);
  }
}

main().catch(console.error);
```

### Step 5: Run

```bash
node examples/security-scan.js
```

---

## What Just Happened?

You created a complete autonomous AI agent that:

1. ✅ **Accepts a task** - "Scan codebase for security vulnerabilities"
2. ✅ **Uses tools autonomously** - Searches and reads code files
3. ✅ **Runs iteratively** - Multiple LLM calls with tool use
4. ✅ **Returns structured output** - JSON with vulnerabilities
5. ✅ **Tracks metrics** - Cost, iterations, duration

---

## Key Concepts Explained

### System Prompt
The system prompt is **the most important part** of your agent. It defines:
- Agent's role and expertise
- Available tools and when to use them
- Process/workflow to follow
- Output format and structure

**Best practices**:
- Be specific about the task
- List available tools explicitly
- Define a clear process (numbered steps)
- Specify exact output format with examples

### Tool Sources
```typescript
toolSources: [
  registryTools(['read_file', 'search_files']),
]
```

This defines which tools the agent can use. Available tools:
- `read_file` - Read file contents
- `search_files` - Search for patterns
- `list_files` - List directory contents
- `git_analysis` - Analyze git history
- `write_file` - Write/modify files
- `execute_command` - Run shell commands

See [05-TOOL-REGISTRY.md](./05-TOOL-REGISTRY.md) for complete list.

### Output Format
```typescript
outputFormat: {
  name: 'security_scan',
  description: 'Security vulnerability scan results',
  schema: {
    type: 'object',
    properties: {
      vulnerabilities: { type: 'array' },
      summary: { type: 'object' },
    },
  },
}
```

Defines the expected JSON structure. Phoenix validates the output against this schema.

See [07-OUTPUT-FORMATS.md](./07-OUTPUT-FORMATS.md) for details.

### Phoenix Optimizations

These flags control Phoenix's 12 optimization systems:

```typescript
enableContextManagement: true,    // Smart context pruning
enableCaching: true,               // Cache tool results
enableParallelTools: true,         // Execute tools in parallel
enableCheckpointing: false,        // Save progress (for long tasks)
enableSelfReflection: true,        // Agent reviews its work
enableErrorRecovery: true,         // Auto-retry on errors
enableAdaptiveIterations: true,    // Adjust iteration limit
enableToolGuidance: true,          // Suggest next tools
enableDeduplication: true,         // Skip duplicate tool calls
enableMultiModelVerification: false, // Use multiple models (expensive)
```

**Rule of thumb**:
- **Fast tasks** (config lookup): Disable reflection, checkpointing, parallel tools
- **Medium tasks** (code explanation): Enable most, disable checkpointing
- **Complex tasks** (defect analysis): Enable all

See [08-BEST-PRACTICES.md](./08-BEST-PRACTICES.md) for optimization strategies.

---

## Common Agent Patterns

### Pattern 1: Read-Only Analysis Agent
```typescript
// For: Code explanation, security scans, documentation
toolSources: [registryTools(['read_file', 'search_files', 'list_files'])],
enableCheckpointing: false,
```

### Pattern 2: Code Modification Agent
```typescript
// For: Refactoring, bug fixes
toolSources: [registryTools(['read_file', 'search_files', 'write_file'])],
enableCheckpointing: true, // Save progress
```

### Pattern 3: Historical Analysis Agent
```typescript
// For: Defect analysis, blame detection
toolSources: [registryTools(['read_file', 'git_analysis'])],
enableSelfReflection: true, // Important for complex analysis
```

### Pattern 4: Fast Lookup Agent
```typescript
// For: Config lookup, quick queries
toolSources: [registryTools(['read_file', 'search_files'])],
maxIterations: 5,
enableParallelTools: false, // Overhead not worth it
enableSelfReflection: false,
```

---

## Testing Your Agent

Create a test file `src/agents/__tests__/SecurityAgent.test.ts`:

```typescript
import { SecurityAgent } from '../SecurityAgent.js';
import { describe, it, expect } from 'vitest';

describe('SecurityAgent', () => {
  it('should scan repository for vulnerabilities', async () => {
    const agent = new SecurityAgent({
      repositoryPath: '/path/to/test/repo',
    });

    const result = await agent.scan();

    expect(result.success).toBe(true);
    expect(result.vulnerabilities).toBeInstanceOf(Array);
    expect(result.summary).toHaveProperty('critical');
    expect(result.metrics.iterations).toBeGreaterThan(0);
  });
});
```

Run tests:
```bash
npm test
```

---

## Next Steps

Now that you've built your first agent:

1. **Learn patterns**: [03-AGENT-DEVELOPMENT-GUIDE.md](./03-AGENT-DEVELOPMENT-GUIDE.md)
2. **Understand architecture**: [02-ARCHITECTURE.md](./02-ARCHITECTURE.md)
3. **Master tools**: [05-TOOL-REGISTRY.md](./05-TOOL-REGISTRY.md)
4. **Optimize performance**: [08-BEST-PRACTICES.md](./08-BEST-PRACTICES.md)
5. **Integrate external tools**: [06-MCP-INTEGRATION.md](./06-MCP-INTEGRATION.md)

---

## Quick Reference

### Creating New Agent Checklist
- [ ] Create `src/agents/MyAgent.ts`
- [ ] Define system prompt with clear process
- [ ] Choose appropriate tool sources
- [ ] Define output format
- [ ] Configure Phoenix optimizations
- [ ] Export from `src/agents/index.ts`
- [ ] Add to `AGENT_REGISTRY`
- [ ] Build: `npm run build`
- [ ] Test with example script
- [ ] Write unit tests

### Agent File Template
```typescript
import { Phoenix, registryTools } from '../phoenix/index.js';

const PROMPT = `You are...`;

export interface MyAgentOptions {
  repositoryPath: string;
}

export interface MyAgentResult {
  success: boolean;
  data: any;
  metrics: {
    iterations: number;
    cost: number;
    durationMs: number;
  };
}

export class MyAgent {
  constructor(private options: MyAgentOptions) {}

  async execute(task: string): Promise<MyAgentResult> {
    const phoenix = new Phoenix({
      taskDescription: task,
      systemPrompt: PROMPT,
      toolSources: [registryTools([...])],
      repositoryPath: this.options.repositoryPath,
    });

    const result = await phoenix.run();
    return { success: result.success, data: result.structuredOutput, metrics: {...} };
  }
}
```

---

**Questions?** See [11-TROUBLESHOOTING.md](./11-TROUBLESHOOTING.md)
