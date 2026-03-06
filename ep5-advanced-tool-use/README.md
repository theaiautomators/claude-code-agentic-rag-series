# Episode 5: Advanced Tool Calling & Agent Intelligence

This episode transforms the platform's tool infrastructure from a **static, hardcoded system into a dynamic, scalable agent architecture** with on-demand tool discovery, multi-tool code execution, and MCP integration.

## What It Is

A dynamic tool registry that lets the LLM discover and load tools on demand, execute multi-step workflows in a single sandbox round-trip, and connect to external tool servers via MCP (Model Context Protocol).

## What You'll Build

- **Context Window Usage Indicator** — Visual progress bar showing token consumption with color-coded thresholds
- **Chat History Interleaved Rendering** — Preserve rich conversation state (sub-agent panels, code execution) across page reloads
- **Unified Tool Registry + Search** — Dynamic tool discovery replacing static injection (~7K tokens → ~500 tokens)
- **Code Mode via Sandbox Bridge** — LLM-generated Python can call platform tools programmatically (up to 98% token reduction)
- **MCP Client Integration** — Connect to external MCP servers (GitHub, Slack, databases, etc.)

## Key Concepts

| Feature | Before | After |
|---------|--------|-------|
| Tool injection | All 14 tools sent every request | Compact catalog + on-demand loading |
| Multi-tool workflows | N sequential tool calls (N round-trips) | Single Python script in sandbox |
| External tools | Hardcoded only | MCP servers auto-discovered |
| History reload | Tool calls grouped, panels lost | Full interleaved rendering preserved |

## New LLM Tools

| Tool | Purpose |
|------|---------|
| `tool_search` | Search the tool catalog by keyword/regex, load full schemas for matches |

## Modified Tools

| Tool | Change |
|------|--------|
| `execute_code` | Gains bridge access—sandbox code can call platform tools via typed Python stubs |

## Tech Stack Additions

| Component | Tech |
|-----------|------|
| MCP Client | `mcp` Python SDK |
| Tool Bridge | FastAPI router + typed Python stubs |

## Configuration

| Variable | Default | Description |
|----------|---------|-------------|
| `TOOL_REGISTRY_ENABLED` | `false` | Enable dynamic tool registry |
| `LLM_CONTEXT_WINDOW` | `128000` | Context window limit for usage indicator |
| `MCP_SERVERS` | — | MCP server definitions (`name:command:args`) |

## Prerequisites

- Completed Episodes 1-4
- Docker with container → host networking
- `npx` available for MCP stdio transport servers

## Documentation

See the [PRD](./PRD-Tool-Calls-v2.md) for complete technical specifications.

## Community

Join [The AI Automators community](https://www.theaiautomators.com/) to connect with other builders creating production-grade AI and RAG systems.
