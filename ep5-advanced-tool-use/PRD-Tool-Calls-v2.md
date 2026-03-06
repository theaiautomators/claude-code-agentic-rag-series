# Episode 5 — PRD: Advanced Tool Calling & Agent Intelligence

## Overview

This episode transforms the platform's tool infrastructure from a static, hardcoded system into a dynamic, scalable agent architecture. The LLM gains the ability to discover tools on demand, execute multi-step workflows in a single sandbox round-trip, and connect to external tool servers via MCP. Alongside these capabilities, the chat experience is improved with context window visibility and persistent rich history rendering.

---

## Feature 1: Context Window Usage Indicator

### What It Does

Displays a cumulative token usage progress bar in the chat UI, showing how much of the LLM's context window has been consumed. As the context fills, the bar changes color from green to yellow to red — giving users a visual signal to start a new thread before output quality degrades.

### How It Works

1. **Backend captures usage** — The OpenAI-compatible streaming API is called with `stream_options: {"include_usage": true}`, which returns token counts on the final chunk of each stream. In a multi-round tool-calling loop, the system tracks the last round's `prompt_tokens` (the most accurate measure since it includes full history + all tool results) and accumulates `completion_tokens` across rounds.

2. **SSE delivery** — A `usage` event is emitted just before the `done` event after each message exchange, containing `prompt_tokens`, `completion_tokens`, and `total_tokens`.

3. **Frontend rendering** — The frontend parses usage events and renders a slim progress bar above the chat input: `45k / 128k (35%)`. The bar resets when switching threads.

### Configuration

The context window limit is configurable via a `LLM_CONTEXT_WINDOW` environment variable (default: 128,000). The value is exposed to the frontend via a new `GET /settings/public` endpoint (no auth required) so it stays in sync with the backend without hardcoding.

### Color Thresholds

| Usage | Color |
|-------|-------|
| 0–59% | Green |
| 60–79% | Yellow/Amber |
| 80–100% | Red |

### UI

- **Location** — Slim bar inside the input footer, above the text input, within the existing `max-w-3xl` container.
- **Visibility** — Only appears after the first message exchange in a thread (when usage data is available). Disappears on thread switch.
- **Format** — Colored fill bar with text label: `Xk / Yk (Z%)`.

### Backend API

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/settings/public` | GET | Returns `{"context_window": N}`. No auth required. |

### SSE Events

| Event | When |
|-------|------|
| `usage` | After each LLM response completes, before `done`. Contains `prompt_tokens`, `completion_tokens`, `total_tokens`. |

### Design Decisions

- **No database persistence** — Token counts are ephemeral, held in frontend state per session. Thread reload resets to zero. Acceptable for v1.
- **Provider compatibility** — Some OpenAI-compatible providers may ignore `stream_options` silently. The system handles this gracefully by only emitting usage events when data is present — no errors occur.
- **Manual context limit** — Auto-detecting limits from provider APIs (OpenRouter `/models`, Ollama `/api/show`) is a future enhancement. Users set the env var based on their model.

---

## Feature 2: Chat History Interleaved Rendering

### What It Does

Preserves the progressive, interleaved rendering of chat conversations across page reloads and thread re-entry. During live streaming, the UI renders messages in the correct order: `LLM text → tool call → LLM text → sub-agent panel → LLM text`. Previously, on reload, all tool calls were grouped together followed by all text — losing the natural sequence. Additionally, sub-agent analysis panels and code execution panels were ephemeral (streaming-only) and disappeared on reload.

### How It Works

1. **Per-round message persistence** — Instead of saving one giant assistant message per exchange, the backend saves one database row per agentic round. Each round contains its own tool calls and text. Natural ordering is preserved via `created_at` timestamps.

2. **Rich state persistence** — Sub-agent state (document analysis results, explorer findings) and code execution state (code, stdout/stderr, files, timing) are stored alongside tool call records in the existing `tool_calls` JSONB column. No schema migration needed — JSONB is schema-flexible.

3. **Faithful reconstruction** — On history load, the frontend processes each message row in order, producing interleaved `ConversationItem[]` entries: regular tool calls → sub-agent panels → code execution panels → text response. Multiple rows (rounds) are processed sequentially, producing the complete interleaved sequence.

### What Gets Persisted

| Data | Storage Location | Reconstructed UI |
|------|-----------------|------------------|
| Tool call name, args, result, status | `tool_calls` JSONB array | StepsPanel (collapsible tool list) |
| Sub-agent mode, document, reasoning, explorer tool calls | `tool_calls[n].sub_agent_state` | SubAgentPanel (analysis/explorer view) |
| Code, stdout, stderr, exit code, timing, output files | `tool_calls[n].code_execution_state` | CodeExecutionPanel (terminal + file downloads) |

### User-Facing Impact

- Reloading a page or revisiting a thread now shows the full rich conversation — sub-agent analysis panels, code execution terminals with output, and tool calls all appear in the correct interleaved order.
- Follow-up messages work correctly because the LLM context is properly reconstructed from the multi-row format.
- Live streaming behavior is unchanged — it already worked correctly.

---

## Feature 3: Unified Tool Registry + Search

### What It Does

Replaces the hardcoded, static tool injection with a dynamic registry that supports on-demand tool discovery. Instead of sending all tools to the LLM every time (~7K tokens for 14 tools), the system provides a compact catalog (name + one-liner per tool) and a `tool_search` meta-tool for pulling full schemas when needed. This is the foundation for scaling to hundreds of tools across native, skill, and MCP sources.

### How Discovery Works

Inspired by Docker MCP Gateway's "primordial tools" pattern:

1. **Compact catalog in system prompt** — Every registered tool's name and one-line description appears in a scannable table. Capped at 50 entries (~500 tokens). The LLM knows what exists without knowing the details.

2. **`tool_search` meta-tool** — When the LLM needs a specific tool, it calls `tool_search` with a keyword or regex query. The registry returns matching tools with their full OpenAI function-calling schemas. Matched tools are added to the active set for the current conversation turn.

3. **Dynamic injection** — Before each LLM call in the agentic loop, the tools array is built from: always-on native tools + tools loaded via `tool_search` in previous rounds. Active tools are ephemeral — reset per request, not persisted across conversations.

### Tool Registration

| Source | When Registered | Loading |
|--------|----------------|---------|
| Native tools (14 existing) | App startup | Immediate (always available) |
| Skills | On database load | Deferred (loaded via `tool_search`) |
| MCP tools | On server connection | Deferred (loaded via `tool_search`) |

### Registry Internals

- **Backing store** — Single `dict[str, ToolDefinition]` with a `source` discriminator (native/skill/mcp).
- **Registration** — `register()` accepts name, description, OpenAI schema, source, loading type, and executor callable.
- **Search** — Regex matching on name + description. Returns matching tools with full schemas.
- **Feature flag** — `TOOL_REGISTRY_ENABLED=false` by default. When disabled, the system uses the original hardcoded `build_rag_tools()` with zero behavioral change.

### LLM Tool

| Tool | Purpose |
|------|---------|
| `tool_search` | Search the tool catalog by keyword/regex. Returns names, descriptions, and full schemas for matches. Matched tools become available for direct calling. |

---

## Feature 4: Code Mode via Sandbox HTTP Bridge

### What It Does

Extends the existing code execution sandbox so that LLM-generated Python code can call platform tools programmatically. Instead of the LLM making N sequential tool calls (N round-trips), it writes a Python script that makes all calls in one sandbox execution — reducing token consumption by up to 98% for data-heavy workflows.

### Technical Approach

Adapted from Cloudflare's "Code Mode" pattern for Docker/llm-sandbox:

1. **HTTP bridge endpoint** — A FastAPI router (`/bridge`) on the host accepts tool execution requests from the sandbox container. The bridge validates session tokens and dispatches calls through the tool registry.

2. **Pre-baked bridge client** — The custom sandbox Docker image includes a Python `ToolClient` class that reads `BRIDGE_URL` and `BRIDGE_TOKEN` from environment variables. LLM-generated code calls `tool_client.call("search_documents", query="revenue")` as naturally as any Python function.

3. **Typed stubs** — For each active tool, a typed Python function stub is generated and injected into the sandbox at runtime. The stubs wrap `ToolClient.call()` with proper parameter names and types, giving the LLM IDE-quality API hints.

4. **Credential isolation** — Service-role Supabase client, API keys, and MCP connections live on the host. Sandbox code sees only the response, never the credentials. The container's network is limited to the bridge endpoint only.

### Single Sandbox Model

Every sandbox session gets bridge access by default — no separate "Code Mode" vs "simple execution" paths. If LLM-generated code doesn't call any tools, the bridge sits unused with zero overhead. This eliminates the need for a separate tool and keeps the mental model simple.

### Security

| Layer | Protection |
|-------|-----------|
| Session tokens | Ephemeral UUIDs, generated per sandbox session, expire with session TTL |
| Bridge auth | Validates session token on every call, cross-checks user ownership |
| Network isolation | Docker container can only reach the bridge endpoint (`host.docker.internal:PORT`) |
| Code scanning | Existing security policy blocks dangerous imports (`subprocess`, `os.system`, `socket`, etc.) |
| Error handling | Bridge client stubs catch errors and return structured error dicts — no exceptions leak |

### Backend API

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/bridge/call` | POST | Execute a tool call from sandbox. Body: `{tool_name, arguments, session_token}` |
| `/bridge/catalog` | GET | List available tools for a session (with session token) |
| `/bridge/health` | GET | Bridge health check |

### SSE Events

| Event | When |
|-------|------|
| `code_mode_start` | Code execution begins with bridge stubs available. Lists available tools. |

### Custom Docker Image

The sandbox Dockerfile is extended with the bridge client module pre-installed. At runtime, generated Python stubs for active tools are injected via `execute_command`. The stubs use `urllib.request` (stdlib) to avoid extra dependencies.

---

## Feature 5: MCP Client Integration

### What It Does

Connects to external MCP (Model Context Protocol) servers, discovers their tools, and registers them in the unified tool registry. This makes any MCP-compatible tool server (GitHub, Slack, databases, etc.) available to the AI through the same `tool_search` → call mechanism as native tools.

### How It Works

1. **Configuration** — MCP servers are defined in the `MCP_SERVERS` env var (format: `name:command:args`, e.g., `github:npx:-y @modelcontextprotocol/server-github`).

2. **Connection** — On app startup, the `MCPClientManager` spawns each configured server via stdio transport using the `mcp` Python SDK. It calls `list_tools()` to discover available tools.

3. **Registration** — Each discovered tool is eagerly converted to three formats: (1) OpenAI function-calling schema, (2) system prompt catalog entry, (3) Python stub definition. All three are stored in the registry as `source="mcp"`, `loading="deferred"`.

4. **Execution** — When an MCP tool is called (either directly by the LLM or via sandbox bridge), the registry's executor callable routes through `mcp_client.call_tool(server_name, tool_name, args)`.

5. **Resilience** — If an MCP server disconnects, the client logs the error, marks the server's tools as unavailable, and attempts reconnection with backoff.

### Schema Conversion

MCP tools use JSON Schema for `inputSchema` — this maps nearly 1:1 to OpenAI's `parameters` format. Edge cases handled: tools without descriptions, complex nested schemas, optional parameters. Conversion is eager (at connect time) to validate schemas early and fail fast on incompatible tools.

### LLM Visibility

MCP tools appear in the system prompt catalog alongside native tools — same compact `name + one-liner` format. They're discoverable via `tool_search` and callable via the bridge from sandbox code. From the LLM's perspective, there's no difference between native and MCP tools.

---

## Cross-Cutting Concerns

### Feature Flags

| Flag | Default | Controls |
|------|---------|----------|
| `TOOL_REGISTRY_ENABLED` | `false` | Entire tool registry system (Features 3, 4, 5). When disabled, falls back to hardcoded tools. |
| `SANDBOX_ENABLED` | `false` | Code execution and bridge (Feature 4). |
| `LLM_CONTEXT_WINDOW` | `128000` | Context window limit for usage indicator (Feature 1). |

### SSE Streaming

New SSE event types follow the existing pattern of JSON objects with a `type` discriminator:

| Event | Feature | Purpose |
|-------|---------|---------|
| `usage` | Context Indicator | Token counts after each exchange |
| `tool_search_results` | Registry | Tools found by a search query |
| `code_mode_start` | Bridge | Sandbox execution with tool access |

### Observability

Registry operations and bridge calls use `@traceable` decorators for LangSmith visibility. Tool search queries, matched tools, bridge call latency, and MCP server status are all traced.

### Backward Compatibility

All changes are gated behind feature flags. With flags disabled:
- The 14 hardcoded tools work identically to Episode 4
- Code execution works identically (no bridge, no stubs)
- No MCP connections are attempted
- Token usage events are simply not emitted if the provider doesn't support `stream_options`
- Chat history rendering improvements work unconditionally (no flag needed — purely better)

### Dependencies

| Package | Feature | Notes |
|---------|---------|-------|
| `mcp` (Python MCP SDK) | MCP Client (Feature 5) | New. `pip install mcp`. |
| All other dependencies | — | Existing. No new npm or pip packages for Features 1–4. |

### Infrastructure Requirements

- Docker must allow container → host networking (`host.docker.internal` — standard Docker Desktop behavior)
- Port 8002 (configurable) available for bridge endpoint
- `npx` available for MCP stdio transport servers (Feature 5)

---

## Summary of New LLM Tools

| Tool | Feature | Purpose |
|------|---------|---------|
| `tool_search` | Registry | Search the tool catalog by keyword/regex, load full schemas for matches |

### Modified LLM Tools

| Tool | Feature | Change |
|------|---------|--------|
| `execute_code` | Bridge | Gains bridge access — sandbox code can call platform tools via typed Python stubs |

---

## Phased Delivery

The tool calling features (3, 4, 5) are built in dependency order:

| Phase | Feature | Prerequisite |
|-------|---------|-------------|
| Phase 1 | Tool Registry + Search | None |
| Phase 2 | Sandbox HTTP Bridge | Phase 1 (registry for tool dispatch) |
| Phase 3 | MCP Client | Phase 1 (registry for tool registration) |

Features 1 (Context Indicator) and 2 (History Rendering) are independent and can ship in any order.
