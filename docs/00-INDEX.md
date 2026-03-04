# Specter Documentation Index

**Welcome to Specter** - An AI Agent Cluster powered by the Phoenix autonomous agent engine.

This documentation is designed to help coding agents (both AI and human developers) quickly understand and start building specialized agents.

---

## Documentation Structure

### 🚀 Getting Started
- **[01-GETTING-STARTED.md](./01-GETTING-STARTED.md)** - Quick start guide to build your first agent in 10 minutes

### 🏗️ Architecture & Concepts
- **[02-ARCHITECTURE.md](./02-ARCHITECTURE.md)** - Deep dive into Specter and Phoenix architecture
- **[04-PHOENIX-CONFIGURATION.md](./04-PHOENIX-CONFIGURATION.md)** - Complete Phoenix configuration reference

### 📖 Development Guides
- **[03-AGENT-DEVELOPMENT-GUIDE.md](./03-AGENT-DEVELOPMENT-GUIDE.md)** - Step-by-step agent creation guide
- **[05-TOOL-REGISTRY.md](./05-TOOL-REGISTRY.md)** - Available tools and usage patterns
- **[06-MCP-INTEGRATION.md](./06-MCP-INTEGRATION.md)** - Integrating external tools via MCP
- **[07-OUTPUT-FORMATS.md](./07-OUTPUT-FORMATS.md)** - Defining structured outputs

### 💡 Best Practices & Patterns
- **[08-BEST-PRACTICES.md](./08-BEST-PRACTICES.md)** - Optimization strategies, patterns, and anti-patterns
- **[10-EXAMPLES.md](./10-EXAMPLES.md)** - Real-world agent examples

### 📚 Reference
- **[09-API-REFERENCE.md](./09-API-REFERENCE.md)** - Complete API documentation
- **[11-TROUBLESHOOTING.md](./11-TROUBLESHOOTING.md)** - Common issues and solutions
- **[12-CONTRIBUTING.md](./12-CONTRIBUTING.md)** - Guidelines for contributing agents

---

## Quick Navigation by Task

| I want to... | Go to... |
|--------------|----------|
| Build my first agent | [Getting Started](./01-GETTING-STARTED.md) |
| Understand Phoenix architecture | [Architecture](./02-ARCHITECTURE.md) |
| Learn agent development patterns | [Agent Development Guide](./03-AGENT-DEVELOPMENT-GUIDE.md) |
| Configure Phoenix options | [Phoenix Configuration](./04-PHOENIX-CONFIGURATION.md) |
| Use file system tools | [Tool Registry](./05-TOOL-REGISTRY.md) |
| Integrate Jira/GitHub/Slack | [MCP Integration](./06-MCP-INTEGRATION.md) |
| Define custom output formats | [Output Formats](./07-OUTPUT-FORMATS.md) |
| Optimize agent performance | [Best Practices](./08-BEST-PRACTICES.md) |
| See example agents | [Examples](./10-EXAMPLES.md) |
| Look up API details | [API Reference](./09-API-REFERENCE.md) |
| Debug issues | [Troubleshooting](./11-TROUBLESHOOTING.md) |

---

## Core Concepts at a Glance

### What is Specter?
Specter is a **cluster of specialized AI agents**, each powered by the Phoenix autonomous agent engine. Each agent is optimized for a specific task type (defect analysis, code explanation, configuration lookup, etc.).

### What is Phoenix?
Phoenix is a **general-purpose autonomous agent loop** with 12 integrated optimizations:
1. Context Management
2. Tool Result Caching
3. Parallel Tool Execution
4. Checkpointing
5. Self-Reflection
6. Error Recovery
7. Adaptive Iterations
8. Token Tracking
9. Tool Usage Guidance
10. Semantic Deduplication
11. Multi-Model Verification
12. Output Validation

### Agent Development Pattern
```typescript
// 1. Define your system prompt
const MY_AGENT_PROMPT = `You are a specialized agent...`;

// 2. Create agent class
export class MyAgent {
  async execute(task: string): Promise<Result> {
    const phoenix = new Phoenix({
      taskDescription: task,
      systemPrompt: MY_AGENT_PROMPT,
      toolSources: [registryTools(['read_file', 'search_files'])],
      outputFormat: OutputFormats.MY_FORMAT,
      maxIterations: 20,
      // Enable optimizations as needed
    });
    return await phoenix.run();
  }
}
```

---

## For AI Coding Agents

If you're an AI agent building a new Specter agent:

1. **Start here**: [01-GETTING-STARTED.md](./01-GETTING-STARTED.md)
2. **Copy a pattern**: [10-EXAMPLES.md](./10-EXAMPLES.md)
3. **Understand tools**: [05-TOOL-REGISTRY.md](./05-TOOL-REGISTRY.md)
4. **Configure Phoenix**: [04-PHOENIX-CONFIGURATION.md](./04-PHOENIX-CONFIGURATION.md)
5. **Follow best practices**: [08-BEST-PRACTICES.md](./08-BEST-PRACTICES.md)

---

## For Human Developers

If you're a human developer:

1. **Read architecture**: [02-ARCHITECTURE.md](./02-ARCHITECTURE.md)
2. **Follow development guide**: [03-AGENT-DEVELOPMENT-GUIDE.md](./03-AGENT-DEVELOPMENT-GUIDE.md)
3. **Check API reference**: [09-API-REFERENCE.md](./09-API-REFERENCE.md)
4. **Deploy**: See README.md in project root

---

## Documentation Conventions

- **Code blocks** are fully executable TypeScript
- **File paths** use the format `src/agents/MyAgent.ts:123`
- **Tool names** are in `backticks` like `read_file`
- **Configuration options** are documented with types and defaults
- **Examples** are complete, not partial snippets

---

## Getting Help

- **Troubleshooting**: [11-TROUBLESHOOTING.md](./11-TROUBLESHOOTING.md)
- **GitHub Issues**: Report bugs or request features
- **Code Examples**: [10-EXAMPLES.md](./10-EXAMPLES.md) has working examples

---

**Ready to build?** Start with [Getting Started](./01-GETTING-STARTED.md) →
