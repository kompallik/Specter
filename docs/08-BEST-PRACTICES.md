# Best Practices & Patterns

Proven patterns and optimization strategies for building effective Specter agents.

---

## Table of Contents

1. [Agent Design Principles](#agent-design-principles)
2. [System Prompt Best Practices](#system-prompt-best-practices)
3. [Optimization Strategies](#optimization-strategies)
4. [Cost Optimization](#cost-optimization)
5. [Performance Tuning](#performance-tuning)
6. [Error Handling Patterns](#error-handling-patterns)
7. [Testing Strategies](#testing-strategies)
8. [Common Anti-Patterns](#common-anti-patterns)

---

## Agent Design Principles

### Principle 1: Single Responsibility

**❌ Bad**: One agent that does everything
```typescript
class SuperAgent {
  async doEverything(task: string) {
    // Tries to handle: defects, explanation, config, security, etc.
  }
}
```

**✅ Good**: Specialized agents for specific tasks
```typescript
class DefectAgent {
  async analyze(defect: string) { /* Focus on defect analysis */ }
}

class ExplainAgent {
  async explain(question: string) { /* Focus on explanation */ }
}
```

**Why**: Specialized agents have:
- Clearer system prompts
- Better tool selection
- Optimized configurations
- Predictable behavior

---

### Principle 2: Explicit Over Implicit

**❌ Bad**: Vague instructions
```typescript
systemPrompt: `You are a code analyzer. Find problems.`
```

**✅ Good**: Explicit instructions
```typescript
systemPrompt: `You are a security vulnerability scanner.

Available tools:
- search_files: Find security-sensitive code
- read_file: Examine implementations

Process:
1. SEARCH: Use search_files for 'password', 'secret', 'eval'
2. READ: Check each file with read_file
3. ANALYZE: Identify OWASP Top 10 vulnerabilities
4. REPORT: Output JSON with severity levels`
```

**Why**: Explicit instructions:
- Reduce ambiguity
- Guide agent behavior
- Improve consistency
- Enable better results

---

### Principle 3: Read Before Write

**❌ Bad**: Direct modification
```typescript
await write_file({ path: 'auth.ts', content: newContent });
```

**✅ Good**: Read, understand, then modify
```typescript
// 1. Read current state
const current = await read_file({ path: 'auth.ts' });

// 2. Understand context
const context = await search_files({ pattern: 'authenticate' });

// 3. Make informed modification
await write_file({ path: 'auth.ts', content: improvedContent });
```

**Why**: Reading first:
- Preserves existing functionality
- Understands dependencies
- Makes targeted changes
- Avoids breaking code

---

### Principle 4: Fail Fast, Recover Gracefully

**❌ Bad**: No error handling
```typescript
const result = await agent.execute(task);
console.log(result.data.field); // Crashes if failed
```

**✅ Good**: Check success, handle errors
```typescript
const result = await agent.execute(task);

if (result.success) {
  console.log('Success:', result.data);
} else {
  console.error('Failed:', result.error);
  // Fallback behavior or retry
}
```

**Why**: Graceful error handling:
- Prevents crashes
- Provides user feedback
- Enables retry logic
- Maintains system stability

---

## System Prompt Best Practices

### Template Structure

```typescript
const SYSTEM_PROMPT = `
# 1. ROLE (Who you are)
You are a [specific expertise] specialist.

# 2. MISSION (What you do)
Your mission: [One clear sentence]

# 3. TOOLS (What you have)
Available tools:
- tool_name: When to use it and what it does
- tool_name: When to use it and what it does

# 4. PROCESS (How to do it)
Process:
1. STEP_NAME: Action with specific tool
2. STEP_NAME: Next action
3. STEP_NAME: Final action

# 5. CONSTRAINTS (Important rules)
Important:
- Guideline 1
- Guideline 2

# 6. OUTPUT (What to return)
Output format:
\`\`\`json
{ "field": "value" }
\`\`\`
`;
```

### Best Practices

✅ **DO**:
- Start with role definition
- List ALL available tools explicitly
- Provide numbered process steps
- Include exact output format with example
- Use consistent terminology
- Be specific about when to stop

❌ **DON'T**:
- Assume agent knows its tools
- Use vague process descriptions
- Skip output format specification
- Mix multiple personas
- Use ambiguous language
- Forget stopping conditions

### Example: Effective Security Scanner Prompt

```typescript
const SECURITY_SCANNER_PROMPT = `You are a security vulnerability scanner specializing in OWASP Top 10.

Your mission: Identify security vulnerabilities in the codebase with specific file locations and remediation advice.

Available tools:
- search_files: Search codebase for security patterns (SQL, auth, crypto, etc.)
- read_file: Read and analyze specific file implementations
- list_files: List directory structure to understand codebase

Process:
1. SEARCH: Use search_files to find security-sensitive code:
   - SQL queries: 'SELECT', 'INSERT', 'UPDATE', 'DELETE'
   - Authentication: 'password', 'authenticate', 'login'
   - Secrets: 'api_key', 'secret', 'token'
   - Dangerous functions: 'eval', 'exec', 'innerHTML'

2. READ: For each match, use read_file to examine full context

3. ANALYZE: Check for vulnerabilities:
   - SQL Injection (unparameterized queries)
   - XSS (unescaped output)
   - Hardcoded secrets
   - Weak crypto
   - Missing authentication

4. REPORT: Output findings with:
   - Vulnerability type
   - Severity (CRITICAL, HIGH, MEDIUM, LOW)
   - Exact file and line number
   - Code snippet
   - Fix recommendation

Important:
- Focus on HIGH and CRITICAL first
- Provide actionable fix advice
- Include code examples in recommendations
- Stop after finding 20 vulnerabilities or scanning all files

Output format:
\`\`\`json
{
  "vulnerabilities": [
    {
      "type": "SQL Injection",
      "severity": "HIGH",
      "file": "src/db/queries.ts",
      "line": 45,
      "code": "db.query('SELECT * FROM users WHERE id = ' + userId)",
      "description": "Unparameterized SQL query allows injection",
      "fix": "Use parameterized queries: db.query('SELECT * FROM users WHERE id = ?', [userId])"
    }
  ],
  "summary": {
    "critical": 0,
    "high": 2,
    "medium": 5,
    "low": 8
  }
}
\`\`\`
`;
```

---

## Optimization Strategies

### Strategy 1: Match Optimizations to Task Complexity

| Task Complexity | Iterations | Key Optimizations |
|-----------------|-----------|-------------------|
| **Simple** (config lookup) | 5 | Context, Cache, Dedup |
| **Medium** (explanation) | 15 | + Parallel, Adaptive, Guidance |
| **Complex** (defect analysis) | 25+ | + Reflection, Checkpointing, All |

**Simple Task Example**:
```typescript
const SIMPLE_CONFIG = {
  maxIterations: 5,
  enableContextManagement: true,
  enableCaching: true,
  enableParallelTools: false,      // Not worth overhead
  enableCheckpointing: false,      // Too fast to need it
  enableSelfReflection: false,     // Skip for speed
  enableErrorRecovery: true,
  enableAdaptiveIterations: false, // Fixed is fine
  enableToolGuidance: false,       // Overhead not worth it
  enableDeduplication: true,
  enableMultiModelVerification: false,
};
```

**Complex Task Example**:
```typescript
const COMPLEX_CONFIG = {
  maxIterations: 0, // Adaptive
  enableContextManagement: true,
  enableCaching: true,
  enableParallelTools: true,       // Worth it for multi-tool
  enableCheckpointing: true,       // Long-running, save progress
  enableSelfReflection: true,      // Complex needs review
  enableErrorRecovery: true,
  enableAdaptiveIterations: true,  // Let agent decide when done
  enableToolGuidance: true,        // Help choose right tool
  enableDeduplication: true,
  enableMultiModelVerification: false, // Unless critical
};
```

---

### Strategy 2: Progressive Enhancement

Start minimal, add optimizations as needed:

**Phase 1 - Baseline**:
```typescript
{
  maxIterations: 10,
  enableContextManagement: true,
  enableCaching: true,
  enableErrorRecovery: true,
  // All others: false
}
```

**Phase 2 - Add Parallelism** (if multiple tools used):
```typescript
{
  // ... baseline ...
  enableParallelTools: true,
}
```

**Phase 3 - Add Intelligence** (if agent struggles):
```typescript
{
  // ... baseline + parallelism ...
  enableAdaptiveIterations: true,
  enableToolGuidance: true,
  enableDeduplication: true,
}
```

**Phase 4 - Add Robustness** (for production):
```typescript
{
  // ... all above ...
  enableCheckpointing: true,     // Long-running tasks
  enableSelfReflection: true,    // Complex analysis
}
```

---

### Strategy 3: Selective Optimization by Agent Type

**Read-Only Analysis Agents**:
```typescript
{
  enableContextManagement: true,   // ✅ Always
  enableCaching: true,             // ✅ High benefit
  enableParallelTools: true,       // ✅ Multiple files
  enableCheckpointing: false,      // ❌ No writes
  enableSelfReflection: false,     // ⚠️ Only if complex
  enableErrorRecovery: true,       // ✅ Always
  enableAdaptiveIterations: true,  // ✅ Uncertain scope
  enableToolGuidance: true,        // ✅ Helps efficiency
  enableDeduplication: true,       // ✅ High benefit
}
```

**Code Modification Agents**:
```typescript
{
  enableContextManagement: true,
  enableCaching: true,
  enableParallelTools: false,      // ❌ Sequential for safety
  enableCheckpointing: true,       // ✅ Critical for writes
  enableSelfReflection: true,      // ✅ Review changes
  enableErrorRecovery: true,
  enableAdaptiveIterations: true,
  enableToolGuidance: true,
  enableDeduplication: false,      // ❌ Each write is unique
}
```

**Fast Lookup Agents**:
```typescript
{
  enableContextManagement: true,
  enableCaching: true,
  enableParallelTools: false,      // ❌ Overhead > benefit
  enableCheckpointing: false,      // ❌ Too fast
  enableSelfReflection: false,     // ❌ Costs time
  enableErrorRecovery: true,
  enableAdaptiveIterations: false, // ❌ Fixed iterations
  enableToolGuidance: false,       // ❌ Overhead > benefit
  enableDeduplication: true,
}
```

---

## Cost Optimization

### Understanding Costs

**Token Costs (Claude Sonnet 4.5)**:
- Input: ~$0.003 per 1K tokens
- Output: ~$0.015 per 1K tokens

**Typical Agent Costs**:
- Simple lookup: $0.005-0.01 (3-5 iterations)
- Medium analysis: $0.01-0.03 (10-15 iterations)
- Complex analysis: $0.03-0.08 (20-30 iterations)

---

### Cost Reduction Strategies

#### 1. Reduce Iteration Count
```typescript
// ❌ Expensive: Unlimited iterations
maxIterations: 0

// ✅ Better: Fixed reasonable limit
maxIterations: 15

// ✅ Best: Adaptive with early stopping
maxIterations: 0,
enableAdaptiveIterations: true  // Stops when making no progress
```

#### 2. Use Caching Aggressively
```typescript
// ✅ Cache identical tool calls
enableCaching: true

// Result: 2nd+ calls are free
read_file({ path: 'auth.ts' })  // $0.0001
read_file({ path: 'auth.ts' })  // $0 (cached)
```

#### 3. Minimize Context Size
```typescript
// ❌ Reading entire large files
read_file({ path: 'huge-file.ts' })  // 10k tokens

// ✅ Read only relevant sections
read_file({
  path: 'huge-file.ts',
  start_line: 100,
  end_line: 200  // 100 lines, ~500 tokens
})
```

#### 4. Deduplication
```typescript
// ✅ Detect semantically similar calls
enableDeduplication: true

// Result: Avoid redundant operations
search_files({ pattern: 'authentication' })
search_files({ pattern: 'auth' })  // Detected as similar, skipped
```

#### 5. Use Haiku for Simple Tasks
```typescript
// For simple lookups
model: 'us.anthropic.claude-haiku-...'  // 3x cheaper
```

#### 6. Implement Cost Limits
```typescript
costLimit: 0.05  // Stop after $0.05
```

#### 7. Monitor and Alert
```typescript
const result = await phoenix.run();

if (result.metrics.totalCost > 0.10) {
  console.warn('High cost detected:', result.metrics.totalCost);
}
```

---

## Performance Tuning

### Speed Optimization Strategies

#### 1. Parallel Tool Execution
```typescript
enableParallelTools: true

// Impact: 2-5x faster for multi-tool iterations
```

#### 2. Reduce Tool Calls
```typescript
// ❌ Multiple searches
search_files({ pattern: 'auth' })
search_files({ pattern: 'login' })
search_files({ pattern: 'password' })

// ✅ One combined search
search_files({ pattern: 'auth|login|password' })
```

#### 3. Use Specific Tool Selection
```typescript
// ❌ Broad tool access
toolSources: [registryTools(['read_file', 'search_files', 'list_files', 'git_analysis'])]

// ✅ Minimal necessary tools
toolSources: [registryTools(['read_file', 'search_files'])]
```

#### 4. Skip Expensive Optimizations
```typescript
// For fast tasks:
enableSelfReflection: false,      // Saves 2-5s
enableToolGuidance: false,        // Saves 1-2s
enableMultiModelVerification: false,  // Saves 10-20s
```

#### 5. Use Search Filters
```typescript
// ❌ Broad search
search_files({ pattern: 'config' })  // Searches everything

// ✅ Filtered search
search_files({
  pattern: 'config',
  path: 'src',
  extensions: ['.ts', '.js']
})  // Much faster
```

---

## Error Handling Patterns

### Pattern 1: Defensive Parsing

```typescript
const result = await agent.execute(task);

if (result.success) {
  // Don't assume structure exists
  const data = result.data || {};
  const field = data.field || 'default';
  const items = Array.isArray(data.items) ? data.items : [];

  // Safe to use
  console.log(field, items.length);
} else {
  console.error('Agent failed:', result.error);
}
```

### Pattern 2: Retry with Backoff

```typescript
async function executeWithRetry(
  agent: MyAgent,
  task: string,
  maxRetries = 3
): Promise<Result> {
  let lastError;

  for (let i = 0; i < maxRetries; i++) {
    try {
      const result = await agent.execute(task);
      if (result.success) return result;
      lastError = result.error;
    } catch (error) {
      lastError = error;
    }

    // Exponential backoff
    await new Promise(resolve => setTimeout(resolve, 1000 * Math.pow(2, i)));
  }

  throw new Error(`Failed after ${maxRetries} retries: ${lastError}`);
}
```

### Pattern 3: Fallback Strategies

```typescript
async function robustAnalysis(defect: string): Promise<Result> {
  // Try full analysis first
  try {
    const result = await defectAgent.analyze(defect);
    if (result.success) return result;
  } catch (error) {
    console.warn('Full analysis failed, trying quick triage');
  }

  // Fallback to quick triage
  try {
    return await defectAgent.quickTriage(defect);
  } catch (error) {
    console.error('All methods failed');
    throw error;
  }
}
```

---

## Testing Strategies

### Unit Testing

```typescript
describe('SecurityAgent', () => {
  let agent: SecurityAgent;
  let testRepo: string;

  beforeEach(async () => {
    // Create test repository
    testRepo = await createTestRepo({
      'vulnerable.ts': `
        const query = 'SELECT * FROM users WHERE id = ' + userId;
        db.execute(query);
      `
    });

    agent = new SecurityAgent({ repositoryPath: testRepo });
  });

  afterEach(async () => {
    await cleanupTestRepo(testRepo);
  });

  it('should detect SQL injection', async () => {
    const result = await agent.scan();

    expect(result.success).toBe(true);
    expect(result.vulnerabilities).toHaveLength(1);
    expect(result.vulnerabilities[0].type).toBe('SQL Injection');
  });
});
```

### Integration Testing

```typescript
describe('SecurityAgent Integration', () => {
  it('should scan real repository', async () => {
    const agent = new SecurityAgent({
      repositoryPath: '/path/to/real/repo',
    });

    const result = await agent.scan();

    expect(result.success).toBe(true);
    expect(result.metrics.iterations).toBeGreaterThan(0);
    expect(result.metrics.cost).toBeLessThan(0.10);
  });
});
```

### Snapshot Testing

```typescript
it('should produce consistent output format', async () => {
  const result = await agent.scan();

  expect(result).toMatchSnapshot({
    metrics: {
      durationMs: expect.any(Number),
      cost: expect.any(Number),
    },
  });
});
```

---

## Common Anti-Patterns

### Anti-Pattern 1: God Agent

**❌ Problem**: One agent that does everything

**✅ Solution**: Create specialized agents for each task type

---

### Anti-Pattern 2: Implicit Instructions

**❌ Problem**: Vague system prompt
```typescript
systemPrompt: 'Analyze the code'
```

**✅ Solution**: Explicit process
```typescript
systemPrompt: `
Process:
1. SEARCH: Use search_files for X
2. READ: Use read_file for Y
3. REPORT: Output JSON with Z
`
```

---

### Anti-Pattern 3: Over-Tooling

**❌ Problem**: Giving agent all tools
```typescript
toolSources: [registryTools([
  'read_file', 'write_file', 'search_files',
  'list_files', 'git_analysis', 'execute_command'
])]
```

**✅ Solution**: Minimal necessary tools
```typescript
toolSources: [registryTools(['read_file', 'search_files'])]
```

---

### Anti-Pattern 4: Ignoring Metrics

**❌ Problem**: Not tracking cost/performance
```typescript
await agent.execute(task);
// No metrics tracking
```

**✅ Solution**: Monitor and optimize
```typescript
const result = await agent.execute(task);

console.log('Cost:', result.metrics.cost);
console.log('Duration:', result.metrics.durationMs);

if (result.metrics.cost > threshold) {
  // Alert or optimize
}
```

---

### Anti-Pattern 5: No Error Handling

**❌ Problem**: Assuming success
```typescript
const result = await agent.execute(task);
console.log(result.data.field); // Crashes if failed
```

**✅ Solution**: Always check success
```typescript
const result = await agent.execute(task);

if (result.success) {
  console.log(result.data.field);
} else {
  console.error('Failed:', result.error);
}
```

---

### Anti-Pattern 6: Hardcoded Paths

**❌ Problem**: Non-portable paths
```typescript
repositoryPath: '/Users/me/myproject'
```

**✅ Solution**: Environment variables
```typescript
repositoryPath: process.env.REPO_PATH || process.cwd()
```

---

### Anti-Pattern 7: No Progress Feedback

**❌ Problem**: Silent execution
```typescript
const agent = new MyAgent({ repositoryPath: '...' });
```

**✅ Solution**: Progress callbacks
```typescript
const agent = new MyAgent({
  repositoryPath: '...',
  onProgress: (msg) => console.log(`[Agent] ${msg}`)
});
```

---

## Quick Reference Checklist

### Agent Development Checklist
- [ ] Agent has single, clear responsibility
- [ ] System prompt is explicit and detailed
- [ ] Only necessary tools included
- [ ] Optimizations match task complexity
- [ ] Error handling implemented
- [ ] Metrics tracked and monitored
- [ ] Progress callbacks provided
- [ ] Tests written (unit + integration)
- [ ] Documentation updated

### Performance Checklist
- [ ] Parallel tools enabled (if applicable)
- [ ] Caching enabled
- [ ] Deduplication enabled
- [ ] Tool calls minimized
- [ ] Context size managed
- [ ] Iteration limit set appropriately
- [ ] Expensive optimizations disabled for fast tasks

### Cost Checklist
- [ ] Iteration limit set
- [ ] Caching enabled
- [ ] Deduplication enabled
- [ ] Cost limit set (if applicable)
- [ ] Metrics monitored
- [ ] Alert on high cost

---

**Next**: [09-API-REFERENCE.md](./09-API-REFERENCE.md)
