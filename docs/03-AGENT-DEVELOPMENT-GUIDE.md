# Agent Development Guide

A comprehensive guide to developing specialized Specter agents.

---

## Table of Contents

1. [Development Process](#development-process)
2. [Agent Patterns](#agent-patterns)
3. [System Prompt Design](#system-prompt-design)
4. [Tool Selection](#tool-selection)
5. [Output Format Design](#output-format-design)
6. [Optimization Configuration](#optimization-configuration)
7. [Testing Strategies](#testing-strategies)
8. [Common Pitfalls](#common-pitfalls)

---

## Development Process

### Step-by-Step Development Workflow

```
1. DEFINE
   ↓
   - What problem does this agent solve?
   - What inputs does it need?
   - What outputs should it produce?

2. DESIGN PROMPT
   ↓
   - Agent role and expertise
   - Clear process/workflow
   - Tool usage instructions
   - Output format specification

3. SELECT TOOLS
   ↓
   - Which tools are needed?
   - Read-only vs write access?
   - Git history needed?
   - External services (MCP)?

4. CONFIGURE OPTIMIZATIONS
   ↓
   - How complex is the task?
   - Speed vs thoroughness tradeoff?
   - Cost constraints?

5. IMPLEMENT
   ↓
   - Create agent class
   - Define interfaces
   - Parse Phoenix output
   - Handle errors

6. TEST
   ↓
   - Unit tests
   - Integration tests
   - Real-world scenarios

7. ITERATE
   ↓
   - Refine prompt
   - Adjust optimizations
   - Improve output parsing
```

---

## Agent Patterns

### Pattern 1: Analysis Agent (Read-Only)

**Use cases**: Code explanation, documentation generation, security scans

```typescript
/**
 * Analysis agents:
 * - Read code without modifying
 * - No git history needed (focus on current state)
 * - Fast iterations
 * - Structured output
 */

const ANALYSIS_PROMPT = `You are a [domain] analysis specialist.

Process:
1. SEARCH: Find relevant code using search_files
2. READ: Examine implementation with read_file
3. ANALYZE: Identify [what to find]
4. REPORT: Output structured findings

Output format:
\`\`\`json
{
  "findings": [...],
  "summary": "..."
}
\`\`\`
`;

export class AnalysisAgent {
  async analyze(query: string): Promise<AnalysisResult> {
    const phoenix = new Phoenix({
      taskDescription: query,
      systemPrompt: ANALYSIS_PROMPT,
      toolSources: [
        registryTools(['read_file', 'search_files', 'list_files']),
      ],
      repositoryPath: this.options.repositoryPath,
      maxIterations: 15,

      // Fast analysis optimizations
      enableContextManagement: true,
      enableCaching: true,
      enableParallelTools: true,
      enableCheckpointing: false,      // No checkpoints needed
      enableSelfReflection: false,     // Speed over thoroughness
      enableErrorRecovery: true,
      enableAdaptiveIterations: true,
      enableToolGuidance: true,
      enableDeduplication: true,
      enableMultiModelVerification: false,
    });

    return this.parseResult(await phoenix.run());
  }
}
```

**Example agents**: ExplainAgent, SecurityAgent, DocumentationAgent

---

### Pattern 2: Modification Agent (Write Access)

**Use cases**: Refactoring, bug fixing, code generation

```typescript
/**
 * Modification agents:
 * - Read AND write code
 * - Needs checkpointing (save progress)
 * - More careful (reflection enabled)
 * - May need git for context
 */

const MODIFICATION_PROMPT = `You are a code modification specialist.

IMPORTANT:
- Read existing code before modifying
- Make minimal, targeted changes
- Preserve existing functionality
- Write clean, maintainable code

Process:
1. UNDERSTAND: Read current implementation
2. PLAN: Determine what changes are needed
3. MODIFY: Use write_file to make changes
4. VERIFY: Explain what was changed and why

Output format:
\`\`\`json
{
  "filesModified": ["file1.ts", "file2.ts"],
  "changesSummary": "...",
  "testingRecommendations": [...]
}
\`\`\`
`;

export class ModificationAgent {
  async modify(task: string): Promise<ModificationResult> {
    const phoenix = new Phoenix({
      taskDescription: task,
      systemPrompt: MODIFICATION_PROMPT,
      toolSources: [
        registryTools(['read_file', 'search_files', 'write_file', 'list_files']),
      ],
      repositoryPath: this.options.repositoryPath,
      maxIterations: 25,

      // Careful modification optimizations
      enableContextManagement: true,
      enableCaching: true,
      enableParallelTools: false,      // Sequential for safety
      enableCheckpointing: true,       // Save progress!
      enableSelfReflection: true,      // Review changes
      enableErrorRecovery: true,
      enableAdaptiveIterations: true,
      enableToolGuidance: true,
      enableDeduplication: true,
      enableMultiModelVerification: false,
    });

    return this.parseResult(await phoenix.run());
  }
}
```

**Example agents**: RefactoringAgent, BugFixAgent, CodeGeneratorAgent

---

### Pattern 3: Historical Analysis Agent (Git Access)

**Use cases**: Defect root cause analysis, blame detection, change impact analysis

```typescript
/**
 * Historical analysis agents:
 * - Need git history access
 * - Complex, multi-step investigation
 * - High iteration count
 * - All optimizations enabled
 */

const HISTORICAL_PROMPT = `You are a software archaeology specialist.

Process:
1. LOCATE: Find relevant code with search_files
2. READ: Examine current implementation with read_file
3. INVESTIGATE: Use git_analysis to find when issues were introduced
4. ANALYZE: Identify root cause
5. PROPOSE: Suggest fix with context

Available git_analysis commands:
- git log --oneline -n 20 -- file.ts
- git blame file.ts
- git show <commit-hash>
- git diff <commit1> <commit2>

Output format:
\`\`\`json
{
  "rootCause": "...",
  "introducingCommit": "abc123",
  "affectedFiles": [...],
  "proposedFix": "..."
}
\`\`\`
`;

export class HistoricalAgent {
  async investigate(issue: string): Promise<HistoricalResult> {
    const phoenix = new Phoenix({
      taskDescription: issue,
      systemPrompt: HISTORICAL_PROMPT,
      toolSources: [
        registryTools(['read_file', 'search_files', 'git_analysis', 'list_files']),
      ],
      repositoryPath: this.options.repositoryPath,
      maxIterations: 0, // Adaptive (will determine dynamically)

      // Full power optimizations
      enableContextManagement: true,
      enableCaching: true,
      enableParallelTools: true,
      enableCheckpointing: true,       // Long-running
      enableSelfReflection: true,      // Complex analysis
      enableErrorRecovery: true,
      enableAdaptiveIterations: true,  // Let it run as long as needed
      enableToolGuidance: true,
      enableDeduplication: true,
      enableMultiModelVerification: false, // Optional for critical tasks
    });

    return this.parseResult(await phoenix.run());
  }
}
```

**Example agents**: DefectAgent, BlameAgent, ImpactAgent

---

### Pattern 4: Lookup Agent (Fast Query)

**Use cases**: Config lookup, quick searches, simple queries

```typescript
/**
 * Lookup agents:
 * - Single-purpose, fast queries
 * - Low iteration count
 * - Minimal optimizations
 * - Predictable output
 */

const LOOKUP_PROMPT = `You are a fast lookup specialist.

Your mission: Find specific information QUICKLY.

Process:
1. SEARCH: Use search_files to locate
2. READ: Use read_file to extract value
3. RETURN: Output the result

Be FAST - use minimal iterations.

Output format:
\`\`\`json
{
  "found": true,
  "value": "...",
  "location": "file.ts:123"
}
\`\`\`
`;

export class LookupAgent {
  async lookup(query: string): Promise<LookupResult> {
    const phoenix = new Phoenix({
      taskDescription: query,
      systemPrompt: LOOKUP_PROMPT,
      toolSources: [
        registryTools(['read_file', 'search_files']),
      ],
      repositoryPath: this.options.repositoryPath,
      maxIterations: 5, // Fixed, low limit

      // Minimal optimizations for speed
      enableContextManagement: true,
      enableCaching: true,
      enableParallelTools: false,      // Overhead not worth it
      enableCheckpointing: false,
      enableSelfReflection: false,     // Skip for speed
      enableErrorRecovery: true,
      enableAdaptiveIterations: false, // Fixed iterations
      enableToolGuidance: false,       // Skip analysis overhead
      enableDeduplication: true,
      enableMultiModelVerification: false,
    });

    return this.parseResult(await phoenix.run());
  }
}
```

**Example agents**: ConfigAgent, SymbolLookupAgent, QuickSearchAgent

---

### Pattern 5: MCP-Integrated Agent (External Services)

**Use cases**: Jira integration, GitHub operations, Slack notifications

```typescript
/**
 * MCP-integrated agents:
 * - Combine local tools with external services
 * - Need proper error handling (network failures)
 * - May need authentication
 * - Structured communication with services
 */

const MCP_INTEGRATION_PROMPT = `You are a [service] integration specialist.

Process:
1. UNDERSTAND: Read the request
2. GATHER: Use read_file/search_files for local context
3. INTEGRATE: Use [service]_* tools to interact with external service
4. REPORT: Summarize what was done

Available [service] tools:
- [service]_create_issue
- [service]_update_issue
- [service]_search

Output format:
\`\`\`json
{
  "action": "created",
  "serviceId": "JIRA-123",
  "url": "...",
  "summary": "..."
}
\`\`\`
`;

export class MCPIntegrationAgent {
  async execute(task: string): Promise<IntegrationResult> {
    const phoenix = new Phoenix({
      taskDescription: task,
      systemPrompt: MCP_INTEGRATION_PROMPT,
      toolSources: [
        registryTools(['read_file', 'search_files']),
        mcpTools({
          name: 'jira',
          type: 'http',
          url: process.env.JIRA_MCP_URL!,
          auth: {
            type: 'bearer',
            token: process.env.JIRA_TOKEN!,
          },
        }, ['jira_create_issue', 'jira_search_issues']),
      ],
      repositoryPath: this.options.repositoryPath,
      maxIterations: 15,

      // Network-aware optimizations
      enableContextManagement: true,
      enableCaching: false,            // Don't cache external calls
      enableParallelTools: true,
      enableCheckpointing: false,
      enableSelfReflection: false,
      enableErrorRecovery: true,       // Important for network failures!
      enableAdaptiveIterations: true,
      enableToolGuidance: true,
      enableDeduplication: false,      // External calls may have side effects
      enableMultiModelVerification: false,
    });

    return this.parseResult(await phoenix.run());
  }
}
```

**Example agents**: JiraAgent, GitHubAgent, SlackAgent

---

## System Prompt Design

### Anatomy of a Good System Prompt

```typescript
const SYSTEM_PROMPT = `
# 1. ROLE DEFINITION
You are a [specific role with expertise].

# 2. MISSION STATEMENT
Your mission: [One clear sentence about the goal]

# 3. AVAILABLE TOOLS (Be explicit!)
Available tools:
- tool_name: Description and when to use it
- another_tool: Description and when to use it

# 4. PROCESS/WORKFLOW (Numbered steps)
Process:
1. STEP_NAME: What to do and which tools to use
2. STEP_NAME: What to do next
3. STEP_NAME: Final step

# 5. CONSTRAINTS/GUIDELINES (If any)
Important:
- Guideline 1
- Guideline 2

# 6. OUTPUT FORMAT (Exact template!)
Output format:
## SECTION_NAME:
\`\`\`json
{
  "field": "value",
  "field2": "value2"
}
\`\`\`
`;
```

### Prompt Best Practices

✅ **DO**:
- Be specific about the agent's expertise
- List all available tools explicitly
- Define a clear numbered process
- Provide exact output format with examples
- Use consistent formatting (markdown headers)
- Include constraints and guidelines

❌ **DON'T**:
- Be vague or general
- Assume the agent knows what tools it has
- Skip the process steps
- Use ambiguous output formats
- Mix instructions with examples
- Forget to specify when to stop

### Example: Bad vs Good Prompts

**❌ Bad Prompt:**
```typescript
const BAD_PROMPT = `You are a code analyzer. Analyze the code and find issues.`;
```

**Problems**:
- Too vague ("code analyzer" could mean anything)
- No tools listed
- No process defined
- No output format

**✅ Good Prompt:**
```typescript
const GOOD_PROMPT = `You are a security vulnerability scanner.

Your mission: Identify security issues in the codebase.

Available tools:
- search_files: Search for security-sensitive patterns (auth, crypto, SQL)
- read_file: Read and analyze specific files

Process:
1. SEARCH: Use search_files to find security-sensitive code
2. READ: Use read_file to examine each file for vulnerabilities
3. ANALYZE: Check against OWASP Top 10
4. REPORT: Output findings in JSON format

Important:
- Focus on HIGH and CRITICAL severity issues first
- Provide specific line numbers
- Include remediation advice

Output format:
## SECURITY SCAN:
\`\`\`json
{
  "vulnerabilities": [
    {
      "type": "SQL Injection",
      "severity": "HIGH",
      "file": "src/db.ts",
      "line": 45,
      "description": "Unparameterized query allows injection",
      "fix": "Use parameterized queries"
    }
  ]
}
\`\`\`
`;
```

---

## Tool Selection

### Tool Selection Decision Tree

```
What does your agent need to do?

READ FILES?
├─ Yes → Include 'read_file'
└─ No → Skip

SEARCH CODEBASE?
├─ Yes → Include 'search_files'
└─ No → Skip

LIST DIRECTORIES?
├─ Yes → Include 'list_files'
└─ No → Skip

MODIFY FILES?
├─ Yes → Include 'write_file' (⚠️ Enable checkpointing!)
└─ No → Skip

ACCESS GIT HISTORY?
├─ Yes → Include 'git_analysis' (⚠️ Increase iterations!)
└─ No → Skip

RUN COMMANDS?
├─ Yes → Include 'execute_command' (⚠️ Security risk!)
└─ No → Skip

INTEGRATE EXTERNAL SERVICES?
├─ Yes → Add MCP server config
└─ No → Skip
```

### Tool Combinations

| Task Type | Recommended Tools |
|-----------|-------------------|
| **Code Reading** | `read_file`, `search_files`, `list_files` |
| **Code Modification** | `read_file`, `search_files`, `write_file` |
| **Git Investigation** | `read_file`, `search_files`, `git_analysis` |
| **Full Analysis** | `read_file`, `search_files`, `list_files`, `git_analysis` |
| **Fast Lookup** | `read_file`, `search_files` (minimal) |
| **External Integration** | `read_file`, `search_files` + MCP tools |

### Tool Usage Guidelines

**read_file**:
- Always read before writing
- Use line ranges for large files
- Fast and cacheable

**search_files**:
- Use specific patterns (not wildcards)
- Specify file extensions when possible
- Combine with read_file for details

**list_files**:
- Use to understand directory structure
- Useful before searching
- Returns file paths only

**write_file**:
- ⚠️ Destructive operation!
- Enable checkpointing when using
- Always read current content first
- Consider git backup

**git_analysis**:
- Slower than other tools
- Use for historical context only
- Common commands:
  - `git log --oneline -- file.ts`
  - `git blame file.ts`
  - `git show <hash>`

**execute_command**:
- ⚠️ Security risk!
- Use sparingly
- Validate commands carefully
- Consider sandboxing

---

## Output Format Design

### Defining Custom Output Formats

```typescript
// Define schema
const MY_OUTPUT_FORMAT: OutputFormat = {
  name: 'my_custom_format',
  description: 'Description of what this format represents',
  schema: {
    type: 'object',
    properties: {
      // Define your structure
      field1: {
        type: 'string',
        description: 'What this field represents',
      },
      field2: {
        type: 'array',
        items: {
          type: 'object',
          properties: {
            nestedField: { type: 'string' },
          },
        },
      },
      field3: {
        type: 'number',
        description: 'Numeric value',
      },
    },
    required: ['field1'], // Which fields are required
  },
};

// Use in agent
const phoenix = new Phoenix({
  // ...
  outputFormat: MY_OUTPUT_FORMAT,
});

// Phoenix will validate output against this schema
```

### Pre-defined Output Formats

```typescript
// Available in OutputFormats enum:
OutputFormats.DEFECT_ANALYSIS
OutputFormats.EXPLANATION
OutputFormats.CONFIG_LOOKUP
OutputFormats.SECURITY_SCAN
OutputFormats.REFACTORING_PLAN
// ... see src/phoenix/PhoenixConfig.ts for full list
```

### Output Format Best Practices

✅ **DO**:
- Use specific field names
- Include descriptions for each field
- Mark required fields
- Use appropriate types (string, number, array, object)
- Keep structure flat when possible

❌ **DON'T**:
- Use ambiguous field names
- Skip field descriptions
- Make everything required (agent may not have all info)
- Over-nest structures
- Use `any` type

---

## Optimization Configuration

### Optimization Decision Matrix

| Optimization | Fast Task | Medium Task | Complex Task | When to Enable |
|--------------|-----------|-------------|--------------|----------------|
| **ContextManagement** | ✅ | ✅ | ✅ | Always |
| **Caching** | ✅ | ✅ | ✅ | Always (unless external side effects) |
| **ParallelTools** | ❌ | ✅ | ✅ | Medium+ complexity |
| **Checkpointing** | ❌ | ❌ | ✅ | Long-running or write operations |
| **SelfReflection** | ❌ | ⚠️ | ✅ | Complex analysis or modifications |
| **ErrorRecovery** | ✅ | ✅ | ✅ | Always |
| **AdaptiveIterations** | ❌ | ✅ | ✅ | When iteration count uncertain |
| **TokenTracking** | ✅ | ✅ | ✅ | Always (for monitoring) |
| **ToolGuidance** | ❌ | ✅ | ✅ | When agent needs help choosing tools |
| **Deduplication** | ✅ | ✅ | ✅ | Always |
| **MultiModelVerify** | ❌ | ❌ | ⚠️ | Only for critical decisions (expensive!) |
| **OutputValidation** | ✅ | ✅ | ✅ | Always (automatically enabled) |

### Optimization Presets

```typescript
// FAST_PRESET: For quick lookups
const FAST_CONFIG = {
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

// BALANCED_PRESET: For medium complexity
const BALANCED_CONFIG = {
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

// THOROUGH_PRESET: For complex analysis
const THOROUGH_CONFIG = {
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

## Testing Strategies

### Unit Testing

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { MyAgent } from '../MyAgent.js';

describe('MyAgent', () => {
  let agent: MyAgent;

  beforeEach(() => {
    agent = new MyAgent({
      repositoryPath: '/path/to/test/repo',
    });
  });

  it('should successfully execute task', async () => {
    const result = await agent.execute('test task');

    expect(result.success).toBe(true);
    expect(result.data).toBeDefined();
    expect(result.metrics.iterations).toBeGreaterThan(0);
  });

  it('should handle errors gracefully', async () => {
    const result = await agent.execute('invalid task');

    expect(result.success).toBe(false);
    expect(result.error).toBeDefined();
  });

  it('should return structured output', async () => {
    const result = await agent.execute('valid task');

    expect(result.data).toHaveProperty('field1');
    expect(result.data).toHaveProperty('field2');
  });
});
```

### Integration Testing

```typescript
describe('MyAgent Integration', () => {
  it('should work with real repository', async () => {
    const agent = new MyAgent({
      repositoryPath: path.join(__dirname, '../../fixtures/sample-repo'),
    });

    const result = await agent.execute('analyze main.ts');

    expect(result.success).toBe(true);
    expect(result.data.findings.length).toBeGreaterThan(0);
  });
});
```

### Manual Testing Script

```typescript
// scripts/test-agent.ts
import { MyAgent } from '../dist/agents/MyAgent.js';

async function testAgent() {
  const agent = new MyAgent({
    repositoryPath: process.argv[2] || '/path/to/repo',
    onProgress: (msg) => console.log(`[Progress] ${msg}`),
  });

  console.log('Testing agent...');
  const start = Date.now();

  const result = await agent.execute('test task');

  const duration = Date.now() - start;

  console.log('\n=== Result ===');
  console.log('Success:', result.success);
  console.log('Duration:', duration, 'ms');
  console.log('Iterations:', result.metrics.iterations);
  console.log('Cost:', `$${result.metrics.cost.toFixed(4)}`);
  console.log('\nData:', JSON.stringify(result.data, null, 2));
}

testAgent().catch(console.error);
```

Run: `node scripts/test-agent.js /path/to/repo`

---

## Common Pitfalls

### Pitfall 1: Vague System Prompts

❌ **Problem**:
```typescript
const BAD = `You are a code analyzer. Find problems.`;
```

✅ **Solution**:
```typescript
const GOOD = `You are a security vulnerability scanner.

Process:
1. SEARCH: Use search_files for 'password', 'secret', 'api_key'
2. READ: Check each file with read_file
3. ANALYZE: Identify hardcoded secrets
4. REPORT: Output JSON with findings`;
```

### Pitfall 2: Wrong Tool Selection

❌ **Problem**: Using `git_analysis` for current code inspection

✅ **Solution**: Use `read_file` and `search_files` for current state, `git_analysis` only for history

### Pitfall 3: Over-Optimization

❌ **Problem**: Enabling all optimizations for a simple lookup

✅ **Solution**: Use FAST_PRESET or disable unnecessary optimizations

### Pitfall 4: Under-Optimization

❌ **Problem**: Disabling checkpointing for long file modification tasks

✅ **Solution**: Enable checkpointing for any task that writes files or takes >30s

### Pitfall 5: Ignoring Output Format

❌ **Problem**: Not defining output format, parsing raw text

✅ **Solution**: Always define structured output format with JSON schema

### Pitfall 6: No Error Handling

❌ **Problem**:
```typescript
const result = await agent.execute(task);
console.log(result.data.field); // May crash if failed
```

✅ **Solution**:
```typescript
const result = await agent.execute(task);
if (result.success) {
  console.log(result.data.field);
} else {
  console.error('Agent failed:', result.error);
}
```

### Pitfall 7: Hardcoded Paths

❌ **Problem**:
```typescript
repositoryPath: '/Users/me/myrepo'
```

✅ **Solution**:
```typescript
repositoryPath: process.env.REPO_PATH || process.cwd()
```

### Pitfall 8: No Progress Callbacks

❌ **Problem**: No visibility into long-running tasks

✅ **Solution**:
```typescript
onProgress: (message) => console.log(`[Agent] ${message}`)
```

---

## Summary

### Agent Development Checklist

- [ ] Define clear problem and use case
- [ ] Choose appropriate agent pattern
- [ ] Design specific system prompt with:
  - [ ] Role definition
  - [ ] Tool list
  - [ ] Clear process
  - [ ] Output format
- [ ] Select minimal necessary tools
- [ ] Configure appropriate optimizations
- [ ] Define structured output format
- [ ] Implement agent class
- [ ] Add error handling
- [ ] Write unit tests
- [ ] Test with real repositories
- [ ] Document usage examples

### Quick Start Template

```typescript
// src/agents/MyAgent.ts
import { Phoenix, registryTools, type PhoenixResult } from '../phoenix/index.js';

const PROMPT = `You are a [role].

Process:
1. [Step 1]
2. [Step 2]

Output: JSON with {...}`;

export interface MyAgentOptions {
  repositoryPath: string;
}

export interface MyAgentResult {
  success: boolean;
  data: any;
  metrics: { iterations: number; cost: number; durationMs: number };
  error?: string;
}

export class MyAgent {
  constructor(private options: MyAgentOptions) {}

  async execute(task: string): Promise<MyAgentResult> {
    const phoenix = new Phoenix({
      taskId: `my-${Date.now()}`,
      taskDescription: task,
      repositoryPath: this.options.repositoryPath,
      systemPrompt: PROMPT,
      toolSources: [registryTools([...])],
      maxIterations: 15,
      // ... optimizations
    });

    const result = await phoenix.run();

    return {
      success: result.success,
      data: result.structuredOutput,
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

---

**Next**: [04-PHOENIX-CONFIGURATION.md](./04-PHOENIX-CONFIGURATION.md)
