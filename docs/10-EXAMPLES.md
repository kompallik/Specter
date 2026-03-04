# Real-World Agent Examples

Complete, working examples of Specter agents for common use cases.

---

## Table of Contents

1. [Security Scanner Agent](#security-scanner-agent)
2. [Test Generator Agent](#test-generator-agent)
3. [Documentation Generator Agent](#documentation-generator-agent)
4. [Refactoring Agent](#refactoring-agent)
5. [API Documentation Agent](#api-documentation-agent)
6. [Dependency Analyzer Agent](#dependency-analyzer-agent)

---

## Security Scanner Agent

**Goal**: Scan codebase for common security vulnerabilities (OWASP Top 10).

**File**: `src/agents/SecurityAgent.ts`

```typescript
import { Phoenix, registryTools, type PhoenixResult } from '../phoenix/index.js';
import type { EventEmitter } from 'events';

const SECURITY_SCANNER_PROMPT = `You are a security vulnerability scanner specializing in OWASP Top 10.

Available tools:
- search_files: Search codebase for security patterns
- read_file: Read and analyze file implementations
- list_files: List directory structure

Process:
1. SEARCH: Use search_files for security-sensitive patterns:
   - SQL: 'SELECT.*WHERE', 'INSERT.*VALUES', '\$\{.*\}'
   - XSS: 'innerHTML', 'dangerouslySetInnerHTML', 'eval('
   - Auth: 'password.*=.*["\']', 'api_key.*=.*["\']', 'secret.*=.*["\']'
   - Crypto: 'md5', 'sha1' (weak algorithms)

2. READ: For each match, use read_file to examine context

3. ANALYZE: Identify vulnerabilities with:
   - Type (SQL Injection, XSS, Hardcoded Secret, etc.)
   - Severity (CRITICAL, HIGH, MEDIUM, LOW)
   - Exact location (file:line)
   - Vulnerable code snippet
   - Fix recommendation

4. REPORT: Output JSON with all findings

Output format:
\`\`\`json
{
  "vulnerabilities": [
    {
      "type": "SQL Injection",
      "severity": "HIGH",
      "file": "src/db/queries.ts",
      "line": 45,
      "code": "db.query('SELECT * WHERE id = ' + userId)",
      "description": "Unparameterized SQL query",
      "fix": "Use parameterized queries"
    }
  ],
  "summary": { "critical": 0, "high": 2, "medium": 5, "low": 3 }
}
\`\`\`
`;

export interface SecurityScanResult {
  success: boolean;
  vulnerabilities: Array<{
    type: string;
    severity: 'CRITICAL' | 'HIGH' | 'MEDIUM' | 'LOW';
    file: string;
    line: number;
    code: string;
    description: string;
    fix: string;
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
  constructor(
    private options: {
      repositoryPath: string;
      eventEmitter?: EventEmitter;
      onProgress?: (message: string) => void;
    }
  ) {}

  async scan(): Promise<SecurityScanResult> {
    const phoenix = new Phoenix({
      taskId: `security-${Date.now()}`,
      taskDescription: 'Scan codebase for security vulnerabilities',
      repositoryPath: this.options.repositoryPath,
      systemPrompt: SECURITY_SCANNER_PROMPT,

      toolSources: [
        registryTools(['read_file', 'search_files', 'list_files']),
      ],

      outputFormat: {
        name: 'security_scan',
        description: 'Security vulnerability report',
        schema: {
          type: 'object',
          properties: {
            vulnerabilities: { type: 'array' },
            summary: { type: 'object' },
          },
          required: ['vulnerabilities', 'summary'],
        },
      },

      eventEmitter: this.options.eventEmitter,
      onProgress: this.options.onProgress,

      maxIterations: 20,

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
    const data = result.structuredOutput as any;

    return {
      success: result.success,
      vulnerabilities: data?.vulnerabilities || [],
      summary: data?.summary || { critical: 0, high: 0, medium: 0, low: 0 },
      metrics: {
        iterations: result.iterationCount,
        cost: result.metrics.totalCost,
        durationMs: result.metrics.totalDurationMs,
      },
      error: result.error,
    };
  }
}

// Usage example:
async function runSecurityScan() {
  const agent = new SecurityAgent({
    repositoryPath: '/path/to/repo',
    onProgress: (msg) => console.log(`[Security] ${msg}`),
  });

  const result = await agent.scan();

  if (result.success) {
    console.log('\n=== Security Scan Results ===');
    console.log(`Total vulnerabilities: ${result.vulnerabilities.length}`);
    console.log(`Summary:`, result.summary);

    // Group by severity
    const bySeverity = result.vulnerabilities.reduce((acc, vuln) => {
      acc[vuln.severity] = acc[vuln.severity] || [];
      acc[vuln.severity].push(vuln);
      return acc;
    }, {} as Record<string, typeof result.vulnerabilities>);

    // Show critical/high first
    ['CRITICAL', 'HIGH', 'MEDIUM', 'LOW'].forEach(severity => {
      const vulns = bySeverity[severity] || [];
      if (vulns.length > 0) {
        console.log(`\n### ${severity} (${vulns.length}):`);
        vulns.forEach((v, i) => {
          console.log(`${i + 1}. ${v.type} in ${v.file}:${v.line}`);
          console.log(`   Issue: ${v.description}`);
          console.log(`   Fix: ${v.fix}\n`);
        });
      }
    });
  }
}
```

---

## Test Generator Agent

**Goal**: Generate unit tests for existing code.

**File**: `src/agents/TestGeneratorAgent.ts`

```typescript
import { Phoenix, registryTools } from '../phoenix/index.js';

const TEST_GENERATOR_PROMPT = `You are a test generation specialist.

Available tools:
- read_file: Read source code
- search_files: Find files to test
- write_file: Write test files

Process:
1. READ: Use read_file to read the source file
2. ANALYZE: Identify:
   - Functions/methods to test
   - Input/output types
   - Edge cases
   - Dependencies

3. GENERATE: Create test file with:
   - Describe blocks for each function
   - Test cases for happy path
   - Test cases for edge cases
   - Test cases for error conditions
   - Proper mocking for dependencies

4. WRITE: Use write_file to save test file

Test framework: Use vitest
Test file naming: [filename].test.ts
Import paths: Relative imports

Output format:
\`\`\`json
{
  "sourceFile": "src/auth.ts",
  "testFile": "src/auth.test.ts",
  "testCount": 12,
  "coverage": {
    "functions": ["authenticate", "validateToken"],
    "edgeCases": ["empty password", "invalid token", "expired token"]
  }
}
\`\`\`
`;

export class TestGeneratorAgent {
  constructor(
    private options: {
      repositoryPath: string;
      onProgress?: (message: string) => void;
    }
  ) {}

  async generate(sourceFile: string) {
    const phoenix = new Phoenix({
      taskId: `test-gen-${Date.now()}`,
      taskDescription: `Generate tests for ${sourceFile}`,
      repositoryPath: this.options.repositoryPath,
      systemPrompt: TEST_GENERATOR_PROMPT,

      toolSources: [
        registryTools(['read_file', 'search_files', 'write_file']),
      ],

      outputFormat: {
        name: 'test_generation',
        schema: {
          type: 'object',
          properties: {
            sourceFile: { type: 'string' },
            testFile: { type: 'string' },
            testCount: { type: 'number' },
            coverage: { type: 'object' },
          },
        },
      },

      onProgress: this.options.onProgress,
      maxIterations: 15,

      enableContextManagement: true,
      enableCaching: true,
      enableParallelTools: false, // Sequential for writes
      enableCheckpointing: true,  // Saving files
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
    };
  }
}

// Usage:
const agent = new TestGeneratorAgent({ repositoryPath: '/repo' });
await agent.generate('src/auth.ts');
```

---

## Documentation Generator Agent

**Goal**: Generate comprehensive README documentation for a codebase.

**File**: `src/agents/DocsGeneratorAgent.ts`

```typescript
import { Phoenix, registryTools } from '../phoenix/index.js';

const DOCS_GENERATOR_PROMPT = `You are a technical documentation specialist.

Available tools:
- list_files: Understand project structure
- read_file: Read package.json, source files
- search_files: Find key functionality

Process:
1. STRUCTURE: Use list_files to understand project layout

2. OVERVIEW: Use read_file on package.json for:
   - Project name and description
   - Dependencies
   - Scripts

3. FEATURES: Use search_files and read_file to identify:
   - Main entry points
   - Key classes/functions
   - API endpoints
   - Configuration

4. GENERATE: Create README with:
   - Project title and description
   - Installation instructions
   - Usage examples
   - API documentation
   - Configuration options
   - Development setup
   - Contributing guidelines

Output: Full README.md content as markdown text
`;

export class DocsGeneratorAgent {
  constructor(
    private options: {
      repositoryPath: string;
      onProgress?: (message: string) => void;
    }
  ) {}

  async generate() {
    const phoenix = new Phoenix({
      taskId: `docs-gen-${Date.now()}`,
      taskDescription: 'Generate comprehensive README documentation',
      repositoryPath: this.options.repositoryPath,
      systemPrompt: DOCS_GENERATOR_PROMPT,

      toolSources: [
        registryTools(['read_file', 'search_files', 'list_files']),
      ],

      // No structured output, plain markdown
      outputFormat: {
        name: 'documentation',
        description: 'Generated README content',
        schema: {
          type: 'object',
          properties: {
            content: { type: 'string' },
          },
        },
      },

      onProgress: this.options.onProgress,
      maxIterations: 20,

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
      readme: (result.structuredOutput as any)?.content || result.output,
      metrics: {
        iterations: result.iterationCount,
        cost: result.metrics.totalCost,
        durationMs: result.metrics.totalDurationMs,
      },
    };
  }
}

// Usage:
const agent = new DocsGeneratorAgent({
  repositoryPath: '/repo',
  onProgress: (msg) => console.log(msg),
});

const result = await agent.generate();
if (result.success) {
  // Write to file
  fs.writeFileSync('README.md', result.readme);
}
```

---

## Refactoring Agent

**Goal**: Refactor code to improve quality while preserving functionality.

**File**: `src/agents/RefactoringAgent.ts`

```typescript
import { Phoenix, registryTools } from '../phoenix/index.js';

const REFACTORING_PROMPT = `You are a code refactoring specialist.

Available tools:
- read_file: Read source code
- search_files: Find related code
- write_file: Write refactored code

Process:
1. READ: Use read_file to read the target file

2. ANALYZE: Identify refactoring opportunities:
   - Long functions (>50 lines) → Extract methods
   - Repeated code → Extract common functions
   - Complex conditionals → Simplify with guard clauses
   - Magic numbers → Extract constants
   - Poor naming → Improve variable/function names
   - Missing types → Add TypeScript types

3. SEARCH: Use search_files to find all usages/dependencies

4. REFACTOR: Apply improvements while:
   - Preserving exact functionality
   - Maintaining backward compatibility
   - Following existing code style
   - Adding comments where needed

5. WRITE: Use write_file to save refactored code

Important:
- Make MINIMAL changes
- Don't change public APIs
- Preserve all functionality
- Add comments explaining changes

Output format:
\`\`\`json
{
  "file": "src/auth.ts",
  "refactorings": [
    {
      "type": "Extract Function",
      "before": "function authenticate(...) { 200 lines }",
      "after": "Split into 4 smaller functions",
      "reason": "Improved readability and testability"
    }
  ]
}
\`\`\`
`;

export class RefactoringAgent {
  constructor(
    private options: {
      repositoryPath: string;
      onProgress?: (message: string) => void;
    }
  ) {}

  async refactor(targetFile: string) {
    const phoenix = new Phoenix({
      taskId: `refactor-${Date.now()}`,
      taskDescription: `Refactor ${targetFile}`,
      repositoryPath: this.options.repositoryPath,
      systemPrompt: REFACTORING_PROMPT,

      toolSources: [
        registryTools(['read_file', 'search_files', 'write_file']),
      ],

      outputFormat: {
        name: 'refactoring',
        schema: {
          type: 'object',
          properties: {
            file: { type: 'string' },
            refactorings: { type: 'array' },
          },
        },
      },

      onProgress: this.options.onProgress,
      maxIterations: 20,

      enableContextManagement: true,
      enableCaching: true,
      enableParallelTools: false, // Sequential for safety
      enableCheckpointing: true,  // Save progress
      enableSelfReflection: true, // Review changes
      enableErrorRecovery: true,
      enableAdaptiveIterations: true,
      enableToolGuidance: true,
      enableDeduplication: false, // Each change is unique
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
    };
  }
}
```

---

## API Documentation Agent

**Goal**: Generate API documentation from source code.

**File**: `src/agents/APIDocsAgent.ts`

```typescript
import { Phoenix, registryTools } from '../phoenix/index.js';

const API_DOCS_PROMPT = `You are an API documentation specialist.

Available tools:
- search_files: Find API routes/controllers
- read_file: Read implementation details

Process:
1. SEARCH: Use search_files to find API endpoints:
   - Express: 'app.get', 'app.post', 'router.get'
   - Fastify: 'fastify.get', 'fastify.post'
   - NestJS: '@Get', '@Post', '@Controller'

2. READ: For each endpoint, use read_file to understand:
   - HTTP method
   - Route path
   - Parameters (path, query, body)
   - Response format
   - Status codes
   - Authentication requirements
   - Validation rules

3. DOCUMENT: Create OpenAPI-style documentation

Output format:
\`\`\`json
{
  "endpoints": [
    {
      "method": "POST",
      "path": "/api/users",
      "summary": "Create new user",
      "parameters": [
        {
          "name": "username",
          "in": "body",
          "type": "string",
          "required": true
        }
      ],
      "responses": {
        "201": { "description": "User created" },
        "400": { "description": "Invalid input" }
      },
      "auth": "Bearer token required"
    }
  ]
}
\`\`\`
`;

export class APIDocsAgent {
  constructor(
    private options: {
      repositoryPath: string;
      onProgress?: (message: string) => void;
    }
  ) {}

  async generate() {
    const phoenix = new Phoenix({
      taskId: `api-docs-${Date.now()}`,
      taskDescription: 'Generate API documentation',
      repositoryPath: this.options.repositoryPath,
      systemPrompt: API_DOCS_PROMPT,

      toolSources: [
        registryTools(['read_file', 'search_files']),
      ],

      outputFormat: {
        name: 'api_documentation',
        schema: {
          type: 'object',
          properties: {
            endpoints: { type: 'array' },
          },
        },
      },

      onProgress: this.options.onProgress,
      maxIterations: 20,

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
      endpoints: (result.structuredOutput as any)?.endpoints || [],
      metrics: {
        iterations: result.iterationCount,
        cost: result.metrics.totalCost,
        durationMs: result.metrics.totalDurationMs,
      },
    };
  }
}
```

---

## Dependency Analyzer Agent

**Goal**: Analyze and report on project dependencies.

**File**: `src/agents/DependencyAgent.ts`

```typescript
import { Phoenix, registryTools } from '../phoenix/index.js';

const DEPENDENCY_ANALYZER_PROMPT = `You are a dependency analysis specialist.

Available tools:
- read_file: Read package.json and lock files
- search_files: Find import statements
- execute_command: Run npm commands

Process:
1. READ: Use read_file to read package.json

2. ANALYZE: Identify:
   - All dependencies (prod + dev)
   - Version constraints
   - Outdated packages
   - Security vulnerabilities
   - Unused dependencies

3. SEARCH: Use search_files to find actual imports

4. EXECUTE: Use execute_command to run:
   - 'npm outdated --json'
   - 'npm audit --json'

5. REPORT: Output dependency analysis

Output format:
\`\`\`json
{
  "dependencies": {
    "total": 45,
    "outdated": 8,
    "vulnerable": 2,
    "unused": 3
  },
  "recommendations": [
    {
      "package": "lodash",
      "current": "4.17.20",
      "latest": "4.17.21",
      "severity": "high",
      "action": "Update immediately"
    }
  ]
}
\`\`\`
`;

export class DependencyAgent {
  constructor(
    private options: {
      repositoryPath: string;
      onProgress?: (message: string) => void;
    }
  ) {}

  async analyze() {
    const phoenix = new Phoenix({
      taskId: `deps-${Date.now()}`,
      taskDescription: 'Analyze project dependencies',
      repositoryPath: this.options.repositoryPath,
      systemPrompt: DEPENDENCY_ANALYZER_PROMPT,

      toolSources: [
        registryTools(['read_file', 'search_files', 'execute_command']),
      ],

      outputFormat: {
        name: 'dependency_analysis',
        schema: {
          type: 'object',
          properties: {
            dependencies: { type: 'object' },
            recommendations: { type: 'array' },
          },
        },
      },

      onProgress: this.options.onProgress,
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
    };
  }
}
```

---

## Running Examples

All examples can be run with:

```bash
# Build first
npm run build

# Run individual agents
node examples/security-scan.js /path/to/repo
node examples/test-generator.js /path/to/repo src/auth.ts
node examples/docs-generator.js /path/to/repo
node examples/refactor.js /path/to/repo src/legacy.ts
node examples/api-docs.js /path/to/repo
node examples/deps-analyzer.js /path/to/repo
```

---

**Next**: [11-TROUBLESHOOTING.md](./11-TROUBLESHOOTING.md)
