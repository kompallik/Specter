# Migration Guide: Phoenix to Claude Agent SDK

This document explains the migration from the custom Phoenix execution engine to the official Claude Agent SDK.

## Overview

**Before (Phoenix)**: Custom autonomous agent loop with 12 built-in optimizations
**After (Claude Agent SDK)**: Official Anthropic SDK with native Bedrock integration

## Why Migrate?

1. **Native Bedrock Support**: Claude Agent SDK has built-in support for Amazon Bedrock with Task IAM Role authentication
2. **Official Support**: Maintained by Anthropic with regular updates
3. **Better Tool Integration**: Native MCP (Model Context Protocol) support for custom tools
4. **Simplified Code**: Less custom code to maintain
5. **Security**: Built-in hooks system for policy enforcement

## Architecture Changes

### Before (Phoenix)

```
Agent (DARCI, SCRIBE, etc.)
    │
    └── Phoenix (Custom execution engine)
            │
            ├── BedrockClient (Custom LLM wrapper)
            ├── ToolRegistry (Custom tool management)
            ├── ContextManager
            ├── TokenTracker
            ├── CheckpointManager
            └── ... (12 optimization modules)
```

### After (Claude Agent SDK)

```
Agent (DARCI, SCRIBE, etc.)
    │
    └── runAgent() (Claude Agent SDK wrapper)
            │
            ├── Claude Agent SDK (query function)
            │       │
            │       └── Bedrock (via AWS credentials)
            │
            ├── gitMcp.ts (Safe git MCP server)
            └── hooks.ts (Security & audit hooks)
```

## File Changes

### New Files (src/claude-sdk/)

| File | Purpose |
|------|---------|
| `config.ts` | Repo registry, output schemas, tool sets |
| `gitMcp.ts` | Safe git MCP server (no Bash) |
| `hooks.ts` | Security hooks, webhooks, audit logging |
| `index.ts` | Main agent runner |

### Modified Files

| File | Changes |
|------|---------|
| `src/agents/darci/DARCIAgent.ts` | Uses `runAgent()` instead of `Phoenix` |
| `src/agents/scribe/SCRIBEAgent.ts` | Uses `runAgent()` instead of `Phoenix` |
| `src/agents/prism/PRISMAgent.ts` | Uses `runAgent()` instead of `Phoenix` |
| `src/agents/hermes/HERMESAgent.ts` | Uses `runAgent()` instead of `Phoenix` |
| `package.json` | Added `@anthropic-ai/claude-agent-sdk` |
| `Dockerfile.orchestrator` | Added Claude Code CLI |
| `terraform/iam.tf` | Added Bedrock IAM policies |
| `.env.example` | Added Bedrock configuration |

### Deprecated (can be removed after migration)

```
src/phoenix/
├── Phoenix.ts               # Replaced by claude-sdk/index.ts
├── AgentLoop.ts             # Legacy
├── ContextManager.ts        # SDK handles internally
├── TokenTracker.ts          # SDK handles internally
├── CheckpointManager.ts     # Not needed with SDK
├── SelfReflection.ts        # SDK has native verification
├── ErrorRecovery.ts         # SDK handles internally
├── AdaptiveIterationManager.ts
├── ToolUsageAnalyzer.ts
├── SemanticDeduplicator.ts
├── MultiModelVerifier.ts
├── ToolResultCache.ts
├── ParallelToolExecutor.ts
├── OutputValidator.ts
├── tools/                   # Replaced by gitMcp.ts
│   ├── ReadFileTool.ts      # SDK has native Read tool
│   ├── SearchFilesTool.ts   # SDK has native Grep/Glob
│   ├── ListFilesTool.ts     # SDK has native Glob
│   ├── ExecuteCommandTool.ts # Removed for security
│   └── GitAnalysisTool.ts   # Replaced by gitMcp.ts
└── mcp/
    └── MCPClient.ts         # SDK has native MCP support
```

## Code Migration Examples

### Before (Phoenix)

```typescript
import { Phoenix, registryTools, OutputFormats } from '../../phoenix/index.js';

const phoenix = new Phoenix({
    taskId: `darci-${Date.now()}`,
    taskDescription: defectDescription,
    repositoryPath: this.options.repositoryPath,
    systemPrompt: DARCI_SYSTEM_PROMPT,
    
    toolSources: [
        registryTools([
            'read_file',
            'search_files',
            'git_analysis',
        ]),
    ],
    
    outputFormat: OutputFormats.DEFECT_ANALYSIS,
    maxIterations: 200,
    
    enableContextManagement: true,
    enableCaching: true,
    enableParallelTools: true,
    // ... 12 feature flags
});

const result = await phoenix.run();
```

### After (Claude Agent SDK)

```typescript
import { runAgent, getAgentPrompt, DARCI_OUTPUT_SCHEMA } from '../../claude-sdk/index.js';

const result = await runAgent({
    taskId: `darci-${Date.now()}`,
    taskPrompt: defectDescription,
    repoIds: [this.options.repoId],
    agentType: 'darci',
    systemAppend: getAgentPrompt('darci'),
    permissionLevel: 'analysis',
    outputSchema: DARCI_OUTPUT_SCHEMA,
    timeoutMs: 10 * 60 * 1000,
});
```

## Configuration Changes

### Environment Variables

**New Required Variables:**

```env
# Bedrock mode (required)
CLAUDE_CODE_USE_BEDROCK=1

# Region (required - SDK doesn't read from .aws config)
AWS_REGION=us-east-1
```

**New Optional Variables:**

```env
# Model selection
ANTHROPIC_MODEL=global.anthropic.claude-sonnet-4-5-20250929-v1:0
ANTHROPIC_SMALL_FAST_MODEL=us.anthropic.claude-haiku-4-5-20251001-v1:0

# Token settings (recommended)
CLAUDE_CODE_MAX_OUTPUT_TOKENS=4096
MAX_THINKING_TOKENS=1024
```

### IAM Permissions

The Task IAM Role needs these permissions:

```json
{
  "Effect": "Allow",
  "Action": [
    "bedrock:InvokeModel",
    "bedrock:InvokeModelWithResponseStream",
    "bedrock:ListInferenceProfiles"
  ],
  "Resource": [
    "arn:aws:bedrock:*:*:inference-profile/global.anthropic.claude-*"
  ]
}
```

## Repo Registration

### Before (Phoenix)

Agents received a `repositoryPath` string directly:

```typescript
const agent = new DARCIAgent({
    repositoryPath: '/mnt/efs/repos/my-project',
});
```

### After (Claude Agent SDK)

Repos must be registered and accessed by ID (security improvement):

```typescript
// In config.ts or at startup
import { registerRepo } from './claude-sdk/index.js';

registerRepo('my-project', '/mnt/efs/repos/my-project');
registerRepo('core-app', '/mnt/efs/repos/core-app');

// In agent usage
const agent = new DARCIAgent({
    repoId: 'my-project',  // Reference by ID, not path
});
```

## Tool Mapping

| Phoenix Tool | Claude SDK Equivalent |
|--------------|----------------------|
| `read_file` | `Read` (native) |
| `search_files` | `Grep` + `Glob` (native) |
| `list_files` | `Glob` (native) |
| `git_analysis` | `mcp__git__*` (custom MCP) |
| `execute_command` | **REMOVED** (security) |
| `find_usages` | `Grep` patterns |
| `get_class_hierarchy` | `Grep` patterns |

## Security Improvements

### 1. No Raw Bash

Phoenix allowed `execute_command` which could run arbitrary commands.
Claude SDK implementation removes this entirely - git operations go through safe MCP server.

### 2. Hooks for Policy Enforcement

```typescript
// Automatically applied:
- denyWritesOutsideRepos()  // Blocks writes outside registered repos
- denyDangerousCommands()   // Blocks dangerous patterns
- makeReadOnlyHook()        // For read-only agents
- makeRateLimitHook()       // Rate limiting
```

### 3. Repo Registration

Agents can only access pre-registered repos by ID, not arbitrary paths.

## Deployment Steps

1. **Update dependencies:**
   ```bash
   npm install @anthropic-ai/claude-agent-sdk
   npm install -g @anthropic-ai/claude-code
   ```

2. **Build new Docker image:**
   ```bash
   docker build -f Dockerfile.orchestrator -t specter:2.0.0 .
   ```

3. **Update Terraform IAM:**
   ```bash
   cd terraform
   terraform plan
   terraform apply
   ```

4. **Update environment variables in ECS task definition:**
   - Add `CLAUDE_CODE_USE_BEDROCK=1`
   - Add `AWS_REGION=us-east-1`
   - Optionally add model selection variables

5. **Register repos at startup:**
   ```typescript
   // In orchestrator startup
   import { registerRepo } from './claude-sdk/index.js';
   
   // Scan EFS and register repos
   const repos = await scanEfsForRepos('/mnt/efs/repos');
   for (const repo of repos) {
       registerRepo(repo.name, repo.path);
   }
   ```

6. **Deploy:**
   ```bash
   aws ecs update-service --cluster specter --service orchestrator --force-new-deployment
   ```

## Rollback Plan

If issues occur, you can rollback by:

1. Reverting to previous Docker image tag
2. Phoenix code is still in the codebase (just not used)
3. Agent files can be reverted to use Phoenix imports

## Testing

### Unit Tests

```bash
npm test
```

### Integration Test

```typescript
import { runAgent, registerRepo } from './claude-sdk/index.js';

// Register test repo
registerRepo('test-repo', '/path/to/test/repo');

// Run test task
const result = await runAgent({
    taskId: 'test-001',
    taskPrompt: 'List all TypeScript files',
    repoIds: ['test-repo'],
    permissionLevel: 'read_only',
    readOnly: true,
});

console.log(result.success);
console.log(result.output);
```

## Support

- Claude Agent SDK docs: https://docs.anthropic.com/en/docs/claude-code
- Bedrock integration: https://docs.aws.amazon.com/bedrock/
- Internal Specter docs: See docs/ folder
