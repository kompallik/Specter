# Contributing to Specter

Guidelines for contributing new agents and improvements to the Specter project.

---

## Table of Contents

1. [Getting Started](#getting-started)
2. [Development Setup](#development-setup)
3. [Creating New Agents](#creating-new-agents)
4. [Code Standards](#code-standards)
5. [Testing Requirements](#testing-requirements)
6. [Documentation Requirements](#documentation-requirements)
7. [Pull Request Process](#pull-request-process)
8. [Agent Naming Conventions](#agent-naming-conventions)

---

## Getting Started

### Prerequisites

- Node.js 18.0.0 or higher
- npm 8.0.0 or higher
- Git
- AWS account with Bedrock access
- Familiarity with TypeScript

### What Can You Contribute?

- **New Agents**: Specialized agents for specific tasks
- **Phoenix Improvements**: Enhancements to the core engine
- **Tool Development**: New built-in tools
- **MCP Integrations**: Connectors for external services
- **Documentation**: Guides, examples, tutorials
- **Bug Fixes**: Issue resolutions
- **Performance Optimizations**: Speed and cost improvements

---

## Development Setup

### 1. Fork and Clone

```bash
# Fork repository on GitHub
# Then clone your fork
git clone https://github.com/YOUR_USERNAME/Specter.git
cd Specter

# Add upstream remote
git remote add upstream https://github.com/ORIGINAL_OWNER/Specter.git
```

### 2. Install Dependencies

```bash
npm install
```

### 3. Configure AWS

```bash
# Set up AWS credentials
aws configure

# Or use environment variables
export AWS_ACCESS_KEY_ID="your-key"
export AWS_SECRET_ACCESS_KEY="your-secret"
export AWS_REGION="us-east-1"
```

### 4. Build Project

```bash
npm run build
```

### 5. Run Tests

```bash
npm test
```

### 6. Verify Setup

```bash
# Run an existing agent to verify everything works
node examples/explain.js /path/to/test/repo "How does authentication work?"
```

---

## Creating New Agents

### Step-by-Step Process

#### 1. Plan Your Agent

Before writing code, define:

**Agent Purpose**:
- What problem does it solve?
- Who is the target user?
- What value does it provide?

**Example**:
- **Purpose**: Detect code smells and suggest refactorings
- **Target**: Developers reviewing legacy code
- **Value**: Automated code quality analysis

**Requirements**:
- What inputs does it need?
- What outputs should it produce?
- What tools are required?

**Example**:
- **Input**: File path or directory
- **Output**: List of code smells with severity and fixes
- **Tools**: read_file, search_files, list_files

#### 2. Create Agent File

**File location**: `src/agents/YourAgent.ts`

**Template**:
```typescript
/**
 * YourAgent - Brief description
 *
 * Detailed description of what this agent does,
 * how it works, and when to use it.
 */

import { Phoenix, registryTools, type PhoenixResult } from '../phoenix/index.js';
import type { EventEmitter } from 'events';

const YOUR_AGENT_PROMPT = `You are a [role] specialist.

Your mission: [Clear one-sentence goal]

Available tools:
- tool_name: Description and when to use

Process:
1. STEP: Description
2. STEP: Description
3. STEP: Description

Output format:
\`\`\`json
{
  "field": "value"
}
\`\`\`
`;

export interface YourAgentOptions {
  repositoryPath: string;
  eventEmitter?: EventEmitter;
  onProgress?: (message: string) => void;
  // Agent-specific options
}

export interface YourAgentResult {
  success: boolean;
  data: any; // Define specific structure
  metrics: {
    iterations: number;
    cost: number;
    durationMs: number;
  };
  error?: string;
}

/**
 * YourAgent - Short description
 *
 * @example
 * ```typescript
 * const agent = new YourAgent({ repositoryPath: '/repo' });
 * const result = await agent.execute('task');
 * ```
 */
export class YourAgent {
  private options: YourAgentOptions;

  constructor(options: YourAgentOptions) {
    this.options = options;
  }

  /**
   * Execute the main agent task
   *
   * @param task - Task description
   * @returns Result with data and metrics
   */
  async execute(task: string): Promise<YourAgentResult> {
    const taskId = `your-agent-${Date.now()}`;

    const phoenix = new Phoenix({
      taskId,
      taskDescription: task,
      repositoryPath: this.options.repositoryPath,
      systemPrompt: YOUR_AGENT_PROMPT,

      toolSources: [
        registryTools([/* list tools */]),
      ],

      outputFormat: {
        name: 'your_agent_output',
        description: 'Description of output',
        schema: {
          type: 'object',
          properties: {
            // Define schema
          },
        },
      },

      eventEmitter: this.options.eventEmitter,
      onProgress: this.options.onProgress,

      maxIterations: 15, // Adjust based on complexity

      // Configure optimizations
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

#### 3. Export Agent

Add to `src/agents/index.ts`:

```typescript
export {
  YourAgent,
  type YourAgentOptions,
  type YourAgentResult,
} from './YourAgent.js';

// Update registry
export const AGENT_REGISTRY = {
  defect: 'DefectAgent',
  explain: 'ExplainAgent',
  config: 'ConfigAgent',
  yourAgent: 'YourAgent', // Add this line
} as const;
```

#### 4. Create Tests

Create `src/agents/__tests__/YourAgent.test.ts`:

```typescript
import { describe, it, expect, beforeEach } from 'vitest';
import { YourAgent } from '../YourAgent.js';
import path from 'path';

describe('YourAgent', () => {
  let agent: YourAgent;
  const testRepoPath = path.join(__dirname, '../../fixtures/test-repo');

  beforeEach(() => {
    agent = new YourAgent({
      repositoryPath: testRepoPath,
    });
  });

  describe('execute', () => {
    it('should successfully execute task', async () => {
      const result = await agent.execute('test task');

      expect(result.success).toBe(true);
      expect(result.data).toBeDefined();
      expect(result.metrics.iterations).toBeGreaterThan(0);
    });

    it('should handle errors gracefully', async () => {
      // Test error scenarios
    });

    it('should return structured output', async () => {
      const result = await agent.execute('valid task');

      expect(result.data).toHaveProperty('expectedField');
    });

    it('should respect cost limits', async () => {
      const result = await agent.execute('task');

      expect(result.metrics.cost).toBeLessThan(0.10);
    });
  });
});
```

#### 5. Create Example

Create `examples/your-agent.ts`:

```typescript
import { YourAgent } from '../dist/agents/YourAgent.js';

async function main() {
  const repoPath = process.argv[2] || process.cwd();

  const agent = new YourAgent({
    repositoryPath: repoPath,
    onProgress: (msg) => console.log(`[YourAgent] ${msg}`),
  });

  console.log('Starting...');
  const result = await agent.execute('task description');

  if (result.success) {
    console.log('Success!');
    console.log('Data:', JSON.stringify(result.data, null, 2));
    console.log(`Cost: $${result.metrics.cost.toFixed(4)}`);
  } else {
    console.error('Failed:', result.error);
    process.exit(1);
  }
}

main().catch(console.error);
```

#### 6. Add Documentation

Create `DOCS/agents/YourAgent.md`:

```markdown
# YourAgent

Brief description of the agent.

## Overview

Detailed description of:
- What it does
- When to use it
- Key features

## Usage

\`\`\`typescript
const agent = new YourAgent({
  repositoryPath: '/path/to/repo',
});

const result = await agent.execute('task');
\`\`\`

## Configuration

### Options

- `repositoryPath` (required): Path to repository
- `option2` (optional): Description

### Outputs

- `field1`: Description
- `field2`: Description

## Examples

### Example 1: Basic Usage

\`\`\`typescript
// Code example
\`\`\`

### Example 2: Advanced Usage

\`\`\`typescript
// Code example
\`\`\`

## Performance

- Typical iterations: 10-15
- Average cost: $0.01-0.02
- Duration: 15-30 seconds

## Limitations

- Limitation 1
- Limitation 2
```

---

## Code Standards

### TypeScript Style

**Naming Conventions**:
```typescript
// Classes: PascalCase
class MyAgent { }

// Interfaces: PascalCase with descriptive names
interface MyAgentOptions { }
interface MyAgentResult { }

// Functions: camelCase
async function executeTask() { }

// Constants: UPPER_SNAKE_CASE
const MAX_ITERATIONS = 20;

// Private members: prefix with underscore
private _internalState: any;
```

**Type Safety**:
```typescript
// ✅ Use explicit types
function execute(task: string): Promise<Result>

// ❌ Avoid any
function execute(task: any): Promise<any>

// ✅ Use proper generics
interface Result<T> {
  data: T;
}
```

**Async/Await**:
```typescript
// ✅ Use async/await
async function execute() {
  const result = await phoenix.run();
  return result;
}

// ❌ Avoid .then()
function execute() {
  return phoenix.run().then(result => result);
}
```

### Code Organization

**File Structure**:
```
src/agents/
  YourAgent.ts           # Main agent implementation
  __tests__/
    YourAgent.test.ts    # Unit tests
  __fixtures__/
    test-data.ts         # Test fixtures
```

**Imports**:
```typescript
// 1. External dependencies
import { EventEmitter } from 'events';
import path from 'path';

// 2. Internal dependencies (Phoenix)
import { Phoenix, registryTools } from '../phoenix/index.js';

// 3. Types
import type { PhoenixResult } from '../phoenix/index.js';

// 4. Local imports
import { helperFunction } from './utils.js';
```

### Documentation

**JSDoc Comments**:
```typescript
/**
 * Execute the agent task
 *
 * @param task - Task description or query
 * @param options - Optional configuration
 * @returns Promise resolving to result with data and metrics
 *
 * @example
 * ```typescript
 * const result = await agent.execute('Find security issues');
 * if (result.success) {
 *   console.log(result.data);
 * }
 * ```
 */
async execute(task: string, options?: ExecuteOptions): Promise<Result> {
  // Implementation
}
```

---

## Testing Requirements

### Test Coverage

All agents must have:
- ✅ Unit tests (>80% coverage)
- ✅ Integration tests (at least 1)
- ✅ Example scripts that work

### Test Structure

```typescript
describe('YourAgent', () => {
  describe('constructor', () => {
    it('should initialize with valid options', () => { });
    it('should throw on invalid options', () => { });
  });

  describe('execute', () => {
    it('should handle successful execution', async () => { });
    it('should handle errors gracefully', async () => { });
    it('should respect timeouts', async () => { });
    it('should respect cost limits', async () => { });
  });

  describe('output format', () => {
    it('should return structured output', async () => { });
    it('should validate against schema', async () => { });
  });
});
```

### Running Tests

```bash
# Run all tests
npm test

# Run specific test file
npm test -- YourAgent.test.ts

# Run with coverage
npm run test:coverage

# Watch mode
npm run test:watch
```

---

## Documentation Requirements

### Required Documentation

For each new agent:

1. **README section** in main README.md
2. **Dedicated doc file** in `DOCS/agents/YourAgent.md`
3. **JSDoc comments** in code
4. **Example script** in `examples/`
5. **Test documentation** in test files

### Documentation Checklist

- [ ] Agent purpose clearly explained
- [ ] Usage examples provided
- [ ] Options documented with types
- [ ] Output format documented
- [ ] Performance characteristics noted
- [ ] Limitations documented
- [ ] Example code works

---

## Pull Request Process

### Before Submitting

**Checklist**:
- [ ] Code follows style guidelines
- [ ] Tests written and passing
- [ ] Documentation updated
- [ ] Example script created and tested
- [ ] No linting errors (`npm run lint`)
- [ ] Build succeeds (`npm run build`)
- [ ] All tests pass (`npm test`)

### Creating PR

1. **Create feature branch**:
```bash
git checkout -b feature/your-agent-name
```

2. **Commit changes**:
```bash
git add .
git commit -m "feat: Add YourAgent for [purpose]"
```

Follow [Conventional Commits](https://www.conventionalcommits.org/):
- `feat:` New feature
- `fix:` Bug fix
- `docs:` Documentation only
- `test:` Adding tests
- `refactor:` Code refactoring
- `perf:` Performance improvement

3. **Push to fork**:
```bash
git push origin feature/your-agent-name
```

4. **Open Pull Request** on GitHub

### PR Template

```markdown
## Description

Brief description of the change.

## Type of Change

- [ ] New agent
- [ ] Bug fix
- [ ] Documentation update
- [ ] Performance improvement

## Agent Details (if applicable)

- **Agent Name**: YourAgent
- **Purpose**: Brief description
- **Tools Used**: List of tools
- **Performance**: ~X iterations, ~$Y cost

## Testing

- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Example script created
- [ ] All tests passing

## Documentation

- [ ] Code comments added
- [ ] README updated
- [ ] Agent documentation created
- [ ] Example provided

## Checklist

- [ ] Code follows style guidelines
- [ ] No linting errors
- [ ] Build succeeds
- [ ] All tests pass
```

### Review Process

1. **Automated checks** run (linting, tests, build)
2. **Code review** by maintainers
3. **Feedback addressed**
4. **Approval** from at least one maintainer
5. **Merge** to main branch

---

## Agent Naming Conventions

### Agent Class Names

Format: `[Purpose]Agent`

**Good examples**:
- `SecurityAgent` - Security scanning
- `TestGeneratorAgent` - Test generation
- `RefactoringAgent` - Code refactoring
- `APIDocsAgent` - API documentation

**Bad examples**:
- `Agent1` - Not descriptive
- `MyAgent` - Too generic
- `TheSecurityScanner` - Wrong format

### File Names

Format: `[Purpose]Agent.ts`

**Examples**:
- `SecurityAgent.ts`
- `TestGeneratorAgent.ts`
- `RefactoringAgent.ts`

### Prompt Names

Format: `[PURPOSE]_[TYPE]_PROMPT`

**Examples**:
- `SECURITY_SCANNER_PROMPT`
- `TEST_GENERATOR_PROMPT`
- `REFACTORING_PROMPT`

---

## Advanced Contributions

### Phoenix Engine Improvements

To contribute to the core Phoenix engine:

1. **Discuss first**: Open issue to discuss proposed change
2. **Test thoroughly**: Phoenix changes affect all agents
3. **Document impact**: Update all affected documentation
4. **Backward compatibility**: Maintain existing agent compatibility

### New Tool Development

To add new built-in tools:

1. **Create tool file**: `src/phoenix/tools/registry/your_tool.ts`
2. **Implement interface**: Follow Tool interface
3. **Add tests**: Tool-specific tests
4. **Register tool**: Add to tool registry
5. **Document**: Update tool registry documentation

Example:
```typescript
export const your_tool: Tool = {
  name: 'your_tool',
  description: 'What it does',
  input_schema: {
    type: 'object',
    properties: {
      param: { type: 'string' },
    },
    required: ['param'],
  },
  execute: async (params: any, context: ToolContext) => {
    // Implementation
    return {
      success: true,
      content: 'result',
    };
  },
};
```

---

## Questions?

- **Documentation**: Check [00-INDEX.md](./00-INDEX.md)
- **Examples**: See [10-EXAMPLES.md](./10-EXAMPLES.md)
- **Issues**: Search GitHub issues
- **Discussions**: Start a GitHub discussion

---

## Code of Conduct

- Be respectful and professional
- Provide constructive feedback
- Help others learn and grow
- Follow project guidelines
- Credit others' work

---

**Thank you for contributing to Specter!** 🎉
