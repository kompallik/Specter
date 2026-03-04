# Output Formats Guide

Guide to defining and using structured output formats in Specter agents.

---

## Overview

Output formats define the expected structure of agent results using JSON Schema. This enables:
- ✅ Structured, parseable outputs
- ✅ Validation against schema
- ✅ Type-safe result handling
- ✅ Consistent agent behavior

---

## Output Format Interface

```typescript
interface OutputFormat {
  name: string;          // Unique identifier
  description: string;   // Human-readable description
  schema: JSONSchema;    // JSON Schema definition
}
```

---

## Pre-defined Output Formats

Phoenix includes several built-in formats:

```typescript
import { OutputFormats } from './phoenix/PhoenixConfig.js';

// Available formats:
OutputFormats.DEFECT_ANALYSIS
OutputFormats.EXPLANATION
OutputFormats.CONFIG_LOOKUP
OutputFormats.SECURITY_SCAN
OutputFormats.REFACTORING_PLAN
OutputFormats.TEST_GENERATION
OutputFormats.DOCUMENTATION
```

---

## Creating Custom Output Formats

### Basic Example

```typescript
const MY_OUTPUT_FORMAT: OutputFormat = {
  name: 'my_custom_format',
  description: 'Description of what this represents',
  schema: {
    type: 'object',
    properties: {
      field1: {
        type: 'string',
        description: 'What this field represents',
      },
      field2: {
        type: 'number',
        description: 'Numeric value',
      },
    },
    required: ['field1'], // Which fields are mandatory
  },
};
```

### Using in Agent

```typescript
const phoenix = new Phoenix({
  // ... other options
  outputFormat: MY_OUTPUT_FORMAT,
});

const result = await phoenix.run();

// Access structured output
const data = result.structuredOutput as {
  field1: string;
  field2?: number;
};
```

---

## JSON Schema Types

### String

```typescript
{
  type: 'string',
  description: 'A text value',
  minLength: 1,
  maxLength: 100,
  pattern: '^[A-Z]',  // Regex pattern
}
```

### Number

```typescript
{
  type: 'number',
  description: 'A numeric value',
  minimum: 0,
  maximum: 100,
}
```

### Boolean

```typescript
{
  type: 'boolean',
  description: 'True or false value',
}
```

### Array

```typescript
{
  type: 'array',
  description: 'List of items',
  items: {
    type: 'string', // Array of strings
  },
  minItems: 1,
  maxItems: 100,
}
```

### Object

```typescript
{
  type: 'object',
  description: 'Nested object',
  properties: {
    nestedField: { type: 'string' },
  },
  required: ['nestedField'],
}
```

### Enum

```typescript
{
  type: 'string',
  enum: ['CRITICAL', 'HIGH', 'MEDIUM', 'LOW'],
  description: 'Severity level',
}
```

---

## Real-World Examples

### Security Scan Output

```typescript
const SECURITY_SCAN_FORMAT: OutputFormat = {
  name: 'security_scan',
  description: 'Security vulnerability scan results',
  schema: {
    type: 'object',
    properties: {
      vulnerabilities: {
        type: 'array',
        description: 'List of found vulnerabilities',
        items: {
          type: 'object',
          properties: {
            type: {
              type: 'string',
              description: 'Vulnerability type (e.g., SQL Injection)',
            },
            severity: {
              type: 'string',
              enum: ['CRITICAL', 'HIGH', 'MEDIUM', 'LOW'],
              description: 'Severity level',
            },
            file: {
              type: 'string',
              description: 'File path',
            },
            line: {
              type: 'number',
              description: 'Line number',
            },
            code: {
              type: 'string',
              description: 'Vulnerable code snippet',
            },
            description: {
              type: 'string',
              description: 'What the vulnerability is',
            },
            fix: {
              type: 'string',
              description: 'How to fix it',
            },
          },
          required: ['type', 'severity', 'file', 'description'],
        },
      },
      summary: {
        type: 'object',
        description: 'Summary statistics',
        properties: {
          critical: { type: 'number' },
          high: { type: 'number' },
          medium: { type: 'number' },
          low: { type: 'number' },
        },
        required: ['critical', 'high', 'medium', 'low'],
      },
    },
    required: ['vulnerabilities', 'summary'],
  },
};
```

### Test Generation Output

```typescript
const TEST_GENERATION_FORMAT: OutputFormat = {
  name: 'test_generation',
  description: 'Generated test information',
  schema: {
    type: 'object',
    properties: {
      sourceFile: {
        type: 'string',
        description: 'Original source file path',
      },
      testFile: {
        type: 'string',
        description: 'Generated test file path',
      },
      testCount: {
        type: 'number',
        description: 'Number of tests generated',
      },
      coverage: {
        type: 'object',
        properties: {
          functions: {
            type: 'array',
            items: { type: 'string' },
            description: 'Functions covered by tests',
          },
          edgeCases: {
            type: 'array',
            items: { type: 'string' },
            description: 'Edge cases tested',
          },
        },
      },
    },
    required: ['sourceFile', 'testFile', 'testCount'],
  },
};
```

### Code Analysis Output

```typescript
const CODE_ANALYSIS_FORMAT: OutputFormat = {
  name: 'code_analysis',
  description: 'Code quality analysis results',
  schema: {
    type: 'object',
    properties: {
      metrics: {
        type: 'object',
        properties: {
          linesOfCode: { type: 'number' },
          complexity: { type: 'number' },
          maintainability: { type: 'number', minimum: 0, maximum: 100 },
        },
      },
      issues: {
        type: 'array',
        items: {
          type: 'object',
          properties: {
            type: { type: 'string' },
            severity: { type: 'string' },
            message: { type: 'string' },
            file: { type: 'string' },
            line: { type: 'number' },
          },
        },
      },
      recommendations: {
        type: 'array',
        items: {
          type: 'object',
          properties: {
            category: { type: 'string' },
            suggestion: { type: 'string' },
            priority: { type: 'string', enum: ['HIGH', 'MEDIUM', 'LOW'] },
          },
        },
      },
    },
    required: ['metrics', 'issues'],
  },
};
```

---

## System Prompt Integration

**Critical**: Your system prompt MUST include the exact output format.

### Good Example

```typescript
const SYSTEM_PROMPT = `You are a security scanner.

Process:
1. Search for vulnerabilities
2. Analyze each finding
3. Output structured results

Output format:
\`\`\`json
{
  "vulnerabilities": [
    {
      "type": "SQL Injection",
      "severity": "HIGH",
      "file": "db.ts",
      "line": 45,
      "description": "...",
      "fix": "..."
    }
  ],
  "summary": {
    "critical": 0,
    "high": 1,
    "medium": 0,
    "low": 0
  }
}
\`\`\`
`;

const outputFormat = {
  name: 'security_scan',
  schema: {
    type: 'object',
    properties: {
      vulnerabilities: { type: 'array' },
      summary: { type: 'object' },
    },
    required: ['vulnerabilities', 'summary'],
  },
};

// Prompt and schema MUST match!
```

---

## Validation

Phoenix automatically validates output against the schema:

```typescript
const result = await phoenix.run();

if (result.success && result.structuredOutput) {
  // Output matched schema
  const data = result.structuredOutput;
} else if (result.success && !result.structuredOutput) {
  // Output didn't match schema (validation failed)
  // result.output contains raw text
} else {
  // Execution failed
  console.error(result.error);
}
```

---

## Best Practices

### 1. Keep Schemas Simple

```typescript
// ❌ Too complex
schema: {
  type: 'object',
  properties: {
    level1: {
      type: 'object',
      properties: {
        level2: {
          type: 'object',
          properties: {
            level3: { /* ... */ }
          }
        }
      }
    }
  }
}

// ✅ Flat and simple
schema: {
  type: 'object',
  properties: {
    field1: { type: 'string' },
    field2: { type: 'number' },
    items: { type: 'array', items: { type: 'string' } }
  }
}
```

### 2. Don't Require Everything

```typescript
// ❌ Too strict
required: ['field1', 'field2', 'field3', 'field4', 'field5']

// ✅ Only require essentials
required: ['field1', 'field2']
// field3, field4, field5 are optional
```

### 3. Include Descriptions

```typescript
// ❌ No descriptions
properties: {
  f1: { type: 'string' },
  f2: { type: 'number' },
}

// ✅ With descriptions
properties: {
  fileName: {
    type: 'string',
    description: 'Path to the file being analyzed'
  },
  lineCount: {
    type: 'number',
    description: 'Total number of lines in file'
  },
}
```

### 4. Use Enums for Fixed Values

```typescript
// ✅ Good
severity: {
  type: 'string',
  enum: ['CRITICAL', 'HIGH', 'MEDIUM', 'LOW'],
}

// ❌ Loose
severity: {
  type: 'string', // Could be anything
}
```

### 5. Provide Examples in Prompt

```typescript
const PROMPT = `...
Output format:
\`\`\`json
{
  "field": "example value",
  "number": 42,
  "list": ["item1", "item2"]
}
\`\`\`

Example output:
\`\`\`json
{
  "field": "Found 3 issues",
  "number": 3,
  "list": ["Issue 1", "Issue 2", "Issue 3"]
}
\`\`\`
`;
```

---

## Fallback Handling

If validation fails, handle gracefully:

```typescript
const result = await phoenix.run();

if (result.structuredOutput) {
  // Use structured output
  return result.structuredOutput;
} else {
  // Fallback to parsing text output
  console.warn('Structured output not available, parsing text');

  // Try to extract JSON from text
  const jsonMatch = result.output.match(/```json\n(.*?)\n```/s);
  if (jsonMatch) {
    try {
      return JSON.parse(jsonMatch[1]);
    } catch (e) {
      console.error('Failed to parse JSON from output');
    }
  }

  // Last resort: return raw text
  return { rawText: result.output };
}
```

---

## TypeScript Types

Define TypeScript types matching your schema:

```typescript
// Define schema
const SECURITY_SCAN_FORMAT = { /* ... */ };

// Define matching TypeScript type
interface SecurityScanOutput {
  vulnerabilities: Array<{
    type: string;
    severity: 'CRITICAL' | 'HIGH' | 'MEDIUM' | 'LOW';
    file: string;
    line: number;
    description: string;
    fix: string;
  }>;
  summary: {
    critical: number;
    high: number;
    medium: number;
    low: number;
  };
}

// Use in agent
const result = await phoenix.run();
const data = result.structuredOutput as SecurityScanOutput;

// Type-safe access
data.vulnerabilities.forEach(vuln => {
  console.log(`${vuln.severity}: ${vuln.type}`);
});
```

---

## Common Patterns

### List of Items

```typescript
{
  type: 'array',
  items: {
    type: 'object',
    properties: {
      id: { type: 'string' },
      name: { type: 'string' },
    },
  },
}
```

### Status with Optional Error

```typescript
{
  type: 'object',
  properties: {
    success: { type: 'boolean' },
    message: { type: 'string' },
    error: { type: 'string' }, // Optional
  },
  required: ['success', 'message'],
}
```

### Categorized Results

```typescript
{
  type: 'object',
  properties: {
    byCategory: {
      type: 'object',
      additionalProperties: {
        type: 'array',
        items: { type: 'string' },
      },
    },
  },
}

// Example output:
{
  "byCategory": {
    "errors": ["Error 1", "Error 2"],
    "warnings": ["Warning 1"],
    "info": ["Info 1", "Info 2", "Info 3"]
  }
}
```

---

**Next**: [08-BEST-PRACTICES.md](./08-BEST-PRACTICES.md)
