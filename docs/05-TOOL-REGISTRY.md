# Tool Registry Reference

Complete reference for all built-in Phoenix tools.

---

## Overview

Phoenix provides a registry of built-in tools for common operations:

- **File Operations**: read_file, write_file, list_files
- **Code Search**: search_files
- **Git Operations**: git_analysis
- **Command Execution**: execute_command

All tools follow a consistent interface and include automatic error handling, retry logic, and result caching.

---

## Tool Interface

Every tool implements this interface:

```typescript
interface Tool {
  name: string;
  description: string;
  input_schema: JSONSchema;
  execute: (params: any, context: ToolContext) => Promise<ToolResult>;
}

interface ToolContext {
  repositoryPath: string;
  taskId: string;
  logger: Logger;
}

interface ToolResult {
  success: boolean;
  content?: any;
  error?: string;
  metadata?: Record<string, any>;
}
```

---

## File Operations

### read_file

**Description**: Read contents of a file with optional line range.

**Input Schema**:
```typescript
{
  path: string;           // File path (absolute or relative to repo)
  start_line?: number;    // Optional: start reading from this line (1-indexed)
  end_line?: number;      // Optional: stop reading at this line
}
```

**Output**:
```typescript
{
  success: true,
  content: string,        // File contents (with line numbers if partial)
  metadata: {
    path: string,         // Resolved absolute path
    lines: number,        // Total lines in file
    size: number,         // File size in bytes
    start_line?: number,  // If range used
    end_line?: number,    // If range used
  }
}
```

**Examples**:

```typescript
// Read entire file
await read_file({ path: 'src/auth.ts' })

// Read specific lines
await read_file({
  path: 'src/auth.ts',
  start_line: 100,
  end_line: 150
})

// Read relative path
await read_file({ path: './package.json' })

// Read absolute path
await read_file({ path: '/Users/name/project/README.md' })
```

**Best Practices**:
- Use line ranges for large files (>1000 lines)
- Paths are relative to `repositoryPath` unless absolute
- Tool returns line numbers in output (format: `  123→ content`)
- Cached automatically (same file read multiple times returns cached)

**Error Cases**:
- File not found → `{ success: false, error: 'File not found: ...' }`
- Permission denied → `{ success: false, error: 'Permission denied: ...' }`
- Binary file → Returns hex preview with warning

---

### write_file

**Description**: Create or overwrite a file with new content.

**⚠️ Warning**: Destructive operation! Enable `enableCheckpointing: true` when using.

**Input Schema**:
```typescript
{
  path: string;          // File path (absolute or relative)
  content: string;       // New file contents
  create_dirs?: boolean; // Create parent directories if needed (default: true)
}
```

**Output**:
```typescript
{
  success: true,
  metadata: {
    path: string,        // Absolute path written
    size: number,        // Bytes written
    created: boolean,    // true if new file, false if overwritten
  }
}
```

**Examples**:

```typescript
// Create new file
await write_file({
  path: 'src/config/new-config.ts',
  content: 'export const config = { ... };'
})

// Overwrite existing file
await write_file({
  path: 'README.md',
  content: '# Updated README\n\n...'
})

// Write without creating directories
await write_file({
  path: 'existing/path/file.ts',
  content: '...',
  create_dirs: false
})
```

**Best Practices**:
- Always read file first to understand current content
- Enable checkpointing when using write operations
- Consider git backup before writes
- Validate content before writing
- Use atomic writes (complete content in one call)

**Error Cases**:
- Permission denied → Error
- Disk full → Error
- Parent directory missing (and create_dirs: false) → Error

---

### list_files

**Description**: List files and directories in a given path.

**Input Schema**:
```typescript
{
  path: string;           // Directory path
  recursive?: boolean;    // List subdirectories recursively (default: false)
  extensions?: string[];  // Filter by file extensions (e.g., ['.ts', '.js'])
  max_depth?: number;     // Maximum recursion depth (default: 10)
}
```

**Output**:
```typescript
{
  success: true,
  content: {
    files: string[],      // List of file paths
    directories: string[], // List of directory paths
    total: number,        // Total items found
  },
  metadata: {
    path: string,
    recursive: boolean,
    depth_reached: number,
  }
}
```

**Examples**:

```typescript
// List current directory
await list_files({ path: '.' })

// List recursively
await list_files({
  path: 'src',
  recursive: true
})

// Filter by extension
await list_files({
  path: 'src',
  recursive: true,
  extensions: ['.ts', '.tsx']
})

// Limit depth
await list_files({
  path: 'node_modules',
  recursive: true,
  max_depth: 2
})
```

**Best Practices**:
- Use extensions filter to reduce results
- Set max_depth for large directories
- Non-recursive for quick directory overview
- Combine with search_files for targeted searches

**Error Cases**:
- Directory not found → Error
- Permission denied → Error
- Too many files (>10000) → Warning, truncates results

---

## Code Search

### search_files

**Description**: Search codebase for pattern using ripgrep (fast regex search).

**Input Schema**:
```typescript
{
  pattern: string;        // Regex pattern or literal string
  path?: string;          // Path to search in (default: repository root)
  extensions?: string[];  // File extensions to search (e.g., ['.ts', '.js'])
  case_sensitive?: boolean; // Case-sensitive search (default: false)
  whole_word?: boolean;   // Match whole words only (default: false)
  context_lines?: number; // Show N lines of context around matches (default: 0)
  max_results?: number;   // Maximum results to return (default: 100)
}
```

**Output**:
```typescript
{
  success: true,
  content: {
    matches: Array<{
      file: string,       // File path
      line: number,       // Line number
      column: number,     // Column number
      text: string,       // Matched line
      context_before?: string[], // Lines before (if context_lines > 0)
      context_after?: string[],  // Lines after
    }>,
    total_matches: number,
    files_searched: number,
    truncated: boolean,   // true if max_results exceeded
  }
}
```

**Examples**:

```typescript
// Simple search
await search_files({ pattern: 'authentication' })

// Search in specific path
await search_files({
  pattern: 'function.*authenticate',
  path: 'src/auth'
})

// Filter by extension
await search_files({
  pattern: 'TODO',
  extensions: ['.ts', '.tsx']
})

// Case-sensitive search
await search_files({
  pattern: 'ErrorCode',
  case_sensitive: true
})

// Whole word match
await search_files({
  pattern: 'user',
  whole_word: true  // Matches 'user' but not 'username'
})

// With context
await search_files({
  pattern: 'export class.*Agent',
  context_lines: 3  // Show 3 lines before and after
})

// Regex patterns
await search_files({ pattern: 'interface \\w+Options' })
await search_files({ pattern: 'async function .+\\(' })
await search_files({ pattern: 'import .* from .*.ts' })
```

**Best Practices**:
- Use specific patterns to reduce noise
- Filter by extensions for faster searches
- Use context_lines to understand matches
- Combine with read_file to get full context
- Escape regex special characters: `\\ ( ) [ ] . * + ? ^ $ | {}`

**Pattern Examples**:

| Goal | Pattern |
|------|---------|
| Find function definitions | `function\s+\w+` |
| Find class definitions | `class\s+\w+` |
| Find imports | `import.*from` |
| Find TODO comments | `TODO:` |
| Find console.log | `console\.log` |
| Find async functions | `async\s+function` |
| Find specific API calls | `fetch\(` |

**Error Cases**:
- Invalid regex → Error with details
- Path not found → Error
- No matches → `{ success: true, content: { matches: [], total_matches: 0 } }`

---

## Git Operations

### git_analysis

**Description**: Execute git commands to analyze repository history.

**⚠️ Note**: Requires git repository, relatively slow (shell execution).

**Input Schema**:
```typescript
{
  command: string;  // Git command to execute (without 'git' prefix)
}
```

**Output**:
```typescript
{
  success: true,
  content: string,  // Command output
  metadata: {
    command: string,
    exit_code: number,
    duration_ms: number,
  }
}
```

**Common Commands**:

```typescript
// View recent commits
await git_analysis({ command: 'log --oneline -n 20' })

// View commit details
await git_analysis({ command: 'show abc123' })

// Blame (who changed each line)
await git_analysis({ command: 'blame src/auth.ts' })

// File history
await git_analysis({ command: 'log --oneline -- src/auth.ts' })

// Diff between commits
await git_analysis({ command: 'diff abc123 def456' })

// Changed files in commit
await git_analysis({ command: 'diff-tree --no-commit-id --name-only -r abc123' })

// Search commit messages
await git_analysis({ command: 'log --grep="fix login"' })

// Show changes in commit
await git_analysis({ command: 'show abc123:src/auth.ts' })

// Find when line was changed
await git_analysis({ command: 'log -S "authenticate" -- src/auth.ts' })

// List branches
await git_analysis({ command: 'branch -a' })

// Show file at specific commit
await git_analysis({ command: 'show commit-hash:path/to/file.ts' })
```

**Best Practices**:
- Use `-n` flag to limit output (e.g., `log -n 20`)
- Specify file paths to narrow scope
- Use `--oneline` for compact output
- Combine commands: `log --oneline --grep="bug" -- src/`
- Check if file exists before running blame

**Common Patterns**:

| Goal | Command |
|------|---------|
| Recent changes to file | `log --oneline -n 10 -- file.ts` |
| Who modified line X | `blame file.ts` |
| Commit that introduced bug | `log -S "buggy code" -- file.ts` |
| Changes in last week | `log --since="1 week ago"` |
| Commits by author | `log --author="John"` |
| File renames | `log --follow -- file.ts` |

**Error Cases**:
- Not a git repo → Error
- Invalid command → Error with git output
- File not in repo → Warning, empty output

---

## Command Execution

### execute_command

**Description**: Execute arbitrary shell commands.

**⚠️ Security Warning**: High risk! Validate commands carefully. Consider disabling in production.

**Input Schema**:
```typescript
{
  command: string;       // Shell command to execute
  timeout_ms?: number;   // Execution timeout (default: 30000)
  cwd?: string;         // Working directory (default: repositoryPath)
}
```

**Output**:
```typescript
{
  success: true,
  content: {
    stdout: string,      // Standard output
    stderr: string,      // Standard error
    exit_code: number,   // Process exit code
  },
  metadata: {
    command: string,
    duration_ms: number,
    cwd: string,
  }
}
```

**Examples**:

```typescript
// Run tests
await execute_command({ command: 'npm test' })

// Check dependencies
await execute_command({ command: 'npm outdated' })

// Run linter
await execute_command({ command: 'eslint src/' })

// Build project
await execute_command({ command: 'npm run build' })

// Custom timeout
await execute_command({
  command: 'npm test',
  timeout_ms: 120000  // 2 minutes
})

// Different working directory
await execute_command({
  command: 'ls -la',
  cwd: '/path/to/dir'
})
```

**Security Best Practices**:
- ⚠️ Never pass user input directly to commands
- Whitelist allowed commands
- Sanitize all inputs
- Use absolute paths
- Set strict timeout
- Consider sandboxing
- Log all executions
- Disable in production if not needed

**Validation Example**:
```typescript
const ALLOWED_COMMANDS = ['npm test', 'npm run build', 'npm outdated'];

function isCommandAllowed(cmd: string): boolean {
  return ALLOWED_COMMANDS.some(allowed => cmd.startsWith(allowed));
}

if (!isCommandAllowed(command)) {
  throw new Error('Command not allowed');
}
```

**Error Cases**:
- Command not found → Error
- Timeout exceeded → Error, process killed
- Non-zero exit code → Success with exit_code in metadata

---

## Tool Selection Guide

### By Task Type

| Task | Recommended Tools |
|------|-------------------|
| **Code Reading** | `read_file`, `search_files`, `list_files` |
| **Code Explanation** | `read_file`, `search_files` |
| **Code Modification** | `read_file`, `write_file`, `search_files` |
| **Defect Analysis** | `read_file`, `search_files`, `git_analysis` |
| **Config Lookup** | `read_file`, `search_files` |
| **Security Scan** | `read_file`, `search_files` |
| **Test Execution** | `execute_command` (with caution) |
| **Documentation** | `read_file`, `search_files`, `list_files` |

### By Agent Pattern

**Analysis Agents** (read-only):
```typescript
registryTools(['read_file', 'search_files', 'list_files'])
```

**Modification Agents**:
```typescript
registryTools(['read_file', 'search_files', 'write_file'])
```

**Historical Agents**:
```typescript
registryTools(['read_file', 'search_files', 'git_analysis'])
```

**Lookup Agents** (fast):
```typescript
registryTools(['read_file', 'search_files'])
```

---

## Tool Performance Characteristics

| Tool | Speed | Cacheability | Cost Impact |
|------|-------|--------------|-------------|
| **read_file** | Fast (10-50ms) | High | Low |
| **write_file** | Fast (10-50ms) | None | Low |
| **list_files** | Fast (10-100ms) | Medium | Low |
| **search_files** | Medium (100-500ms) | High | Medium |
| **git_analysis** | Slow (500-5000ms) | Low | High |
| **execute_command** | Variable | None | Variable |

---

## Tool Result Caching

Phoenix automatically caches tool results based on:
- Tool name
- Input parameters (JSON serialized)
- Repository state (for file operations)

**Cache Invalidation**:
- Automatic after write operations
- Manual via `clearCache()` method
- Disabled with `enableCaching: false`

**Example**:
```typescript
// First call: executes
await read_file({ path: 'auth.ts' })  // 50ms

// Second call: cached
await read_file({ path: 'auth.ts' })  // <1ms

// Different parameters: not cached
await read_file({ path: 'auth.ts', start_line: 100 })  // 50ms
```

---

## Error Handling

All tools use consistent error handling:

```typescript
try {
  const result = await tool.execute(params, context);
  if (!result.success) {
    // Handle tool-specific error
    console.error('Tool failed:', result.error);
  }
} catch (error) {
  // Handle unexpected error
  console.error('Unexpected error:', error);
}
```

Phoenix automatically:
- Retries failed tools (if `enableErrorRecovery: true`)
- Logs errors
- Returns error message to agent
- Continues execution (doesn't crash)

---

## Custom Tools

To add custom tools to the registry:

```typescript
import { toolRegistry } from './phoenix/tools/registry';

const my_custom_tool: Tool = {
  name: 'my_custom_tool',
  description: 'Does something custom',
  input_schema: {
    type: 'object',
    properties: {
      param1: { type: 'string' },
    },
    required: ['param1'],
  },
  execute: async (params: any, context: ToolContext) => {
    // Implementation
    return {
      success: true,
      content: 'result',
    };
  },
};

toolRegistry.register(my_custom_tool);

// Use in agent
toolSources: [registryTools(['my_custom_tool'])]
```

---

## Tool Usage Patterns

### Pattern 1: Search → Read
```typescript
// 1. Find relevant files
const searchResult = await search_files({
  pattern: 'authenticate',
  extensions: ['.ts']
});

// 2. Read each file for details
for (const match of searchResult.matches) {
  await read_file({ path: match.file });
}
```

### Pattern 2: List → Filter → Read
```typescript
// 1. List all files
const listResult = await list_files({
  path: 'src',
  recursive: true,
  extensions: ['.ts']
});

// 2. Filter by criteria
const configFiles = listResult.files.filter(f => f.includes('config'));

// 3. Read each
for (const file of configFiles) {
  await read_file({ path: file });
}
```

### Pattern 3: Git History → Read → Write
```typescript
// 1. Find when issue introduced
const gitLog = await git_analysis({
  command: 'log -S "buggy code" -- auth.ts'
});

// 2. Read current file
const current = await read_file({ path: 'auth.ts' });

// 3. Write fix
await write_file({
  path: 'auth.ts',
  content: fixedContent
});
```

---

**Next**: [06-MCP-INTEGRATION.md](./06-MCP-INTEGRATION.md)
