# Troubleshooting Guide

Common issues and their solutions when working with Specter agents.

---

## Table of Contents

1. [Installation Issues](#installation-issues)
2. [Authentication & Credentials](#authentication--credentials)
3. [Agent Execution Issues](#agent-execution-issues)
4. [Performance Problems](#performance-problems)
5. [Cost Issues](#cost-issues)
6. [Output Format Issues](#output-format-issues)
7. [Tool Execution Errors](#tool-execution-errors)
8. [MCP Integration Issues](#mcp-integration-issues)

---

## Installation Issues

### Issue: `npm install` fails with dependency errors

**Symptoms**:
```
npm ERR! code ERESOLVE
npm ERR! ERESOLVE unable to resolve dependency tree
```

**Solutions**:

1. **Update Node.js**:
```bash
node --version  # Should be 18.0.0 or higher
nvm install 18
nvm use 18
```

2. **Clear npm cache**:
```bash
npm cache clean --force
rm -rf node_modules package-lock.json
npm install
```

3. **Use legacy peer deps**:
```bash
npm install --legacy-peer-deps
```

---

### Issue: TypeScript compilation errors

**Symptoms**:
```
error TS2307: Cannot find module './phoenix/index.js'
```

**Solutions**:

1. **Check tsconfig.json**:
```json
{
  "compilerOptions": {
    "module": "ES2022",
    "moduleResolution": "node",
    "target": "ES2022"
  }
}
```

2. **Use .js extensions in imports**:
```typescript
// ✅ Correct
import { Phoenix } from './phoenix/index.js';

// ❌ Wrong
import { Phoenix } from './phoenix/index';
```

3. **Rebuild**:
```bash
npm run build
```

---

## Authentication & Credentials

### Issue: AWS Bedrock authentication fails

**Symptoms**:
```
Error: Unable to authenticate with AWS Bedrock
```

**Solutions**:

1. **Check AWS credentials**:
```bash
aws configure list
# Should show configured credentials
```

2. **Set environment variables**:
```bash
export AWS_ACCESS_KEY_ID="your-key"
export AWS_SECRET_ACCESS_KEY="your-secret"
export AWS_REGION="us-east-1"
```

3. **Use AWS profile**:
```bash
export AWS_PROFILE="your-profile"
```

4. **Verify Bedrock access**:
```bash
aws bedrock list-foundation-models --region us-east-1
```

5. **Check IAM permissions**:
Ensure your IAM user/role has:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bedrock:InvokeModel",
        "bedrock:InvokeModelWithResponseStream"
      ],
      "Resource": "*"
    }
  ]
}
```

---

### Issue: MCP server authentication fails

**Symptoms**:
```
Error: MCP server authentication failed: 401 Unauthorized
```

**Solutions**:

1. **Check token**:
```typescript
const token = process.env.JIRA_TOKEN;
if (!token) {
  throw new Error('JIRA_TOKEN not set');
}
```

2. **Verify token format**:
```bash
# Bearer token
export JIRA_TOKEN="Bearer your-token-here"

# API key
export API_KEY="your-api-key"
```

3. **Test token manually**:
```bash
curl -H "Authorization: Bearer $JIRA_TOKEN" https://api.example.com/health
```

---

## Agent Execution Issues

### Issue: Agent times out or hangs

**Symptoms**:
- Agent runs for a long time without completing
- No progress updates
- Eventually times out

**Solutions**:

1. **Add timeout**:
```typescript
const phoenix = new Phoenix({
  // ...
  timeoutMs: 120000, // 2 minutes
});
```

2. **Reduce max iterations**:
```typescript
maxIterations: 10, // Instead of 0 (adaptive)
```

3. **Enable progress callbacks**:
```typescript
onProgress: (msg) => {
  console.log(`[${new Date().toISOString()}] ${msg}`);
}
```

4. **Check for infinite loops**:
```typescript
// Enable adaptive iterations to detect stalls
enableAdaptiveIterations: true
```

---

### Issue: Agent produces no output

**Symptoms**:
```typescript
result.output === ''
result.structuredOutput === undefined
```

**Solutions**:

1. **Check system prompt**:
```typescript
// ✅ Includes output format
systemPrompt: `...
Output format:
\`\`\`json
{ "field": "value" }
\`\`\`
`

// ❌ Missing output format
systemPrompt: `Do the task.`
```

2. **Verify output format schema**:
```typescript
outputFormat: {
  name: 'my_output',
  schema: {
    type: 'object',
    properties: { /* define properties */ },
    required: ['field1'], // At least one required field
  },
}
```

3. **Check iteration count**:
```typescript
if (result.iterationCount === maxIterations) {
  console.warn('Hit iteration limit without completing');
}
```

---

### Issue: Agent keeps calling the same tool repeatedly

**Symptoms**:
- Agent calls `read_file` with same path 10+ times
- High iteration count with no progress
- Wasted cost

**Solutions**:

1. **Enable caching**:
```typescript
enableCaching: true
```

2. **Enable deduplication**:
```typescript
enableDeduplication: true
```

3. **Improve system prompt**:
```typescript
// ❌ Vague
"Read files as needed"

// ✅ Specific
"1. Use search_files ONCE to find relevant files
 2. Use read_file to read each file ONCE
 3. After reading all files, provide analysis"
```

4. **Enable self-reflection**:
```typescript
enableSelfReflection: true // Agent reviews progress
```

---

## Performance Problems

### Issue: Agent is too slow

**Symptoms**:
- Takes >60s for simple tasks
- High iteration count
- Many sequential tool calls

**Solutions**:

1. **Enable parallel tools**:
```typescript
enableParallelTools: true
```

2. **Reduce tool set**:
```typescript
// ❌ Too many tools
toolSources: [registryTools(['read_file', 'search_files', 'list_files', 'git_analysis', 'write_file'])]

// ✅ Minimal necessary
toolSources: [registryTools(['read_file', 'search_files'])]
```

3. **Disable expensive optimizations**:
```typescript
enableSelfReflection: false,      // Saves 2-5s
enableCheckpointing: false,       // Saves 1-2s
enableToolGuidance: false,        // Saves 1-2s
enableMultiModelVerification: false, // Saves 10-20s
```

4. **Lower max iterations**:
```typescript
maxIterations: 10 // Instead of 20+
```

5. **Use faster model** (for simple tasks):
```typescript
model: 'us.anthropic.claude-haiku-...' // 3x faster
```

---

### Issue: High context window usage

**Symptoms**:
```
Error: Context window exceeded (200k tokens)
```

**Solutions**:

1. **Enable context management**:
```typescript
enableContextManagement: true
```

2. **Read files in chunks**:
```typescript
// ❌ Read entire 10k line file
read_file({ path: 'huge.ts' })

// ✅ Read specific section
read_file({ path: 'huge.ts', start_line: 100, end_line: 200 })
```

3. **Limit search results**:
```typescript
search_files({
  pattern: 'TODO',
  max_results: 50 // Default is 100
})
```

---

## Cost Issues

### Issue: Unexpectedly high costs

**Symptoms**:
- Single agent run costs $0.50+
- Rapid cost accumulation

**Solutions**:

1. **Set cost limit**:
```typescript
costLimit: 0.10 // Stop at $0.10
```

2. **Monitor costs in real-time**:
```typescript
eventEmitter.on('llm:response', ({ cost }) => {
  totalCost += cost;
  if (totalCost > 0.05) {
    console.warn('High cost detected:', totalCost);
  }
});
```

3. **Enable caching**:
```typescript
enableCaching: true // Avoid redundant tool calls
```

4. **Reduce iteration count**:
```typescript
maxIterations: 10 // Instead of adaptive
```

5. **Check for repeated operations**:
```typescript
enableDeduplication: true
```

---

### Issue: Want to estimate costs before running

**Solution**:

```typescript
// Rough cost estimation
function estimateCost(options: {
  iterations: number;
  toolCallsPerIteration: number;
  avgOutputTokens: number;
}) {
  const { iterations, toolCallsPerIteration, avgOutputTokens } = options;

  // Sonnet 4.5 pricing
  const inputCostPer1K = 0.003;
  const outputCostPer1K = 0.015;

  // Rough estimates
  const avgInputTokensPerIter = 2000 + (toolCallsPerIteration * 500);
  const totalInputTokens = iterations * avgInputTokensPerIter;
  const totalOutputTokens = iterations * avgOutputTokens;

  const inputCost = (totalInputTokens / 1000) * inputCostPer1K;
  const outputCost = (totalOutputTokens / 1000) * outputCostPer1K;

  return inputCost + outputCost;
}

// Example: 10 iterations, 2 tools per iteration, 500 output tokens
const estimated = estimateCost({
  iterations: 10,
  toolCallsPerIteration: 2,
  avgOutputTokens: 500,
});

console.log(`Estimated cost: $${estimated.toFixed(3)}`);
```

---

## Output Format Issues

### Issue: Output doesn't match schema

**Symptoms**:
```
Error: Output validation failed
result.structuredOutput === undefined
```

**Solutions**:

1. **Verify schema in system prompt**:
```typescript
// System prompt MUST include exact output format
systemPrompt: `...
Output format:
\`\`\`json
{
  "field1": "value",
  "field2": 123
}
\`\`\`
`
```

2. **Check schema definition**:
```typescript
outputFormat: {
  name: 'my_output',
  schema: {
    type: 'object',
    properties: {
      field1: { type: 'string' },
      field2: { type: 'number' },
    },
    required: ['field1'], // Don't make everything required
  },
}
```

3. **Parse output manually**:
```typescript
if (!result.structuredOutput) {
  // Try to extract JSON from text output
  const match = result.output.match(/```json\n(.*?)\n```/s);
  if (match) {
    result.structuredOutput = JSON.parse(match[1]);
  }
}
```

---

### Issue: Agent returns partial data

**Symptoms**:
- Some fields missing from output
- Incomplete analysis

**Solutions**:

1. **Check iteration limit**:
```typescript
if (result.iterationCount >= maxIterations) {
  console.warn('Agent hit iteration limit, output may be incomplete');
}
```

2. **Increase iterations**:
```typescript
maxIterations: 25 // or 0 for adaptive
```

3. **Enable adaptive iterations**:
```typescript
enableAdaptiveIterations: true
```

4. **Make fewer fields required**:
```typescript
schema: {
  type: 'object',
  properties: {
    critical: { type: 'string' },
    optional: { type: 'string' },
  },
  required: ['critical'], // Only critical field required
}
```

---

## Tool Execution Errors

### Issue: read_file fails with "File not found"

**Symptoms**:
```
Error: File not found: src/auth.ts
```

**Solutions**:

1. **Check path is relative to repositoryPath**:
```typescript
// If repositoryPath is '/repo'
read_file({ path: 'src/auth.ts' }) // ✅ Looks in /repo/src/auth.ts
read_file({ path: '/src/auth.ts' }) // ❌ Looks in root /src/auth.ts
```

2. **Verify file exists**:
```bash
ls /path/to/repo/src/auth.ts
```

3. **Use search_files first**:
```typescript
// Find files first
const files = await search_files({ pattern: 'auth' });
// Then read found files
await read_file({ path: files.matches[0].file });
```

---

### Issue: search_files returns no results

**Symptoms**:
```
{ matches: [], total_matches: 0 }
```

**Solutions**:

1. **Check pattern syntax**:
```typescript
// ❌ Invalid regex
pattern: 'function('  // Unescaped (

// ✅ Escaped
pattern: 'function\\('
```

2. **Try simpler pattern**:
```typescript
// Start broad
pattern: 'authenticate'

// Then narrow down
pattern: 'function authenticate'
```

3. **Check file extensions**:
```typescript
search_files({
  pattern: 'TODO',
  extensions: ['.ts', '.tsx', '.js', '.jsx'] // Include all relevant
})
```

4. **Case insensitivity**:
```typescript
search_files({
  pattern: 'error',
  case_sensitive: false // Match Error, ERROR, error
})
```

---

### Issue: git_analysis fails

**Symptoms**:
```
Error: Not a git repository
```

**Solutions**:

1. **Verify git repo**:
```bash
cd /path/to/repo
git status
```

2. **Initialize if needed**:
```bash
git init
```

3. **Check git command**:
```typescript
// ✅ Valid
command: 'log --oneline -n 10'

// ❌ Invalid (includes 'git')
command: 'git log --oneline' // Remove 'git' prefix
```

---

### Issue: write_file permission denied

**Symptoms**:
```
Error: Permission denied: /path/to/file.ts
```

**Solutions**:

1. **Check file permissions**:
```bash
ls -la /path/to/file.ts
```

2. **Check directory permissions**:
```bash
ls -la /path/to/
```

3. **Run with appropriate permissions**:
```bash
sudo node your-agent.js
# or
chmod +w /path/to/file.ts
```

---

## MCP Integration Issues

### Issue: MCP server connection fails

**Symptoms**:
```
Error: Failed to connect to MCP server: Connection refused
```

**Solutions**:

1. **Verify server is running**:
```bash
curl http://localhost:3000/health
```

2. **Check URL**:
```typescript
mcpTools({
  name: 'jira',
  type: 'http',
  url: 'http://localhost:3000', // Include protocol
  // ...
})
```

3. **Test connection manually**:
```bash
curl -X POST http://localhost:3000/rpc \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'
```

4. **Check firewall/network**:
```bash
nc -zv localhost 3000
```

---

### Issue: MCP tool execution fails

**Symptoms**:
```
Error: MCP tool 'jira_create_issue' failed: 500 Internal Server Error
```

**Solutions**:

1. **Check MCP server logs**:
```bash
# Server-side error logs
```

2. **Verify tool parameters**:
```typescript
// Ensure parameters match MCP tool schema
{
  projectKey: 'PROJ',
  summary: 'Title',
  description: 'Details',
}
```

3. **Enable MCP event logging**:
```typescript
eventEmitter.on('mcp:tool_called', ({ server, tool, params, result }) => {
  console.log('MCP Tool:', tool, params, result);
});
```

---

## Getting More Help

### Enable Debug Logging

```typescript
import { createLogger } from './utils/logger.js';

const logger = createLogger({
  level: 'debug', // 'error' | 'warn' | 'info' | 'debug'
  pretty: true,
});

// Phoenix uses this logger internally
```

### Check Event Logs

```typescript
const emitter = new EventEmitter();

// Log ALL events
emitter.onAny((event, data) => {
  console.log(`[Event] ${event}:`, JSON.stringify(data, null, 2));
});

const phoenix = new Phoenix({
  // ...
  eventEmitter: emitter,
});
```

### Minimal Reproduction

Create a minimal test case:

```typescript
import { Phoenix, registryTools } from './phoenix/index.js';

const phoenix = new Phoenix({
  taskId: 'debug-test',
  taskDescription: 'Simple test',
  repositoryPath: '/tmp/test-repo',
  systemPrompt: 'You are a test. Output: {"result": "success"}',
  toolSources: [registryTools(['read_file'])],
  outputFormat: {
    name: 'test',
    schema: {
      type: 'object',
      properties: { result: { type: 'string' } },
    },
  },
  maxIterations: 5,
  onProgress: console.log,
});

const result = await phoenix.run();
console.log('Result:', JSON.stringify(result, null, 2));
```

### Report Issues

If you've tried all troubleshooting steps:

1. Create minimal reproduction
2. Collect logs (enable debug logging)
3. Check existing GitHub issues
4. Create new issue with:
   - Specter version
   - Node.js version
   - Complete error message
   - Minimal reproduction code
   - Steps to reproduce

---

**Next**: [12-CONTRIBUTING.md](./12-CONTRIBUTING.md)
