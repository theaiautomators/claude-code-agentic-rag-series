# Episode 4 — PRD: Agent Skills & Code Execution

## Overview

Episode 4 transforms the Agentic RAG chat app into a customizable AI agent platform. Users can create reusable "skills" that give the AI specialized behaviors, attach resource files to those skills, execute code in a sandboxed environment, and import/export skills using an open standard format. The AI gains persistent memory of tool results across conversation turns.

---

## Feature 1: Agent Skills

### What It Does

Skills are reusable, named behavior units that extend what the AI can do. Each skill has a name, a short description (used for discovery), and full instructions (loaded on demand). Users create skills to teach the AI how to handle specific domains — legal review, sales analysis, report generation, etc.

### How Discovery Works

The system uses a **progressive disclosure** pattern inspired by Claude's agent skills architecture:

1. **Lightweight catalog in system prompt** — Every enabled skill's name and short description is injected into the LLM's system prompt as a scannable table. This is cheap on tokens since descriptions are capped at ~1-2 sentences.
2. **On-demand loading** — When the user's query matches a skill's description, the LLM calls a `load_skill` tool to fetch the full instructions. This avoids stuffing all skill instructions into every request.
3. **Anti-speculation guardrail** — The system prompt explicitly tells the LLM to only load skills when there's a clear match, preventing unnecessary tool calls.

### Skill Properties

| Property | Description |
|----------|-------------|
| Name | Lowercase, hyphenated identifier (e.g., `analyzing-sales-data`). Max 64 chars. Must not start/end with hyphens or contain consecutive hyphens. |
| Description | 1-2 sentences describing what the skill does AND when to use it. Written in third person. Min 20 chars, max 1024 chars. |
| Instructions | Full markdown instructions loaded when the skill is activated. Should only contain context the LLM doesn't already know. |
| Enabled | Toggle controlling whether the skill appears in the LLM's discovery catalog. Disabled skills remain manageable in the UI but are invisible to the AI. |
| License | Optional license identifier (e.g., MIT, Apache-2.0) for portability. |
| Compatibility | Optional compatibility descriptor for cross-tool interop. |
| Metadata | Optional key-value pairs for extensibility. |

### Ownership Model

- **Private skills** — Owned by a user, visible only to them.
- **Global skills** — No owner (`user_id = NULL`), visible to all users. Users cannot edit or delete global skills.
- **Sharing** — Users can toggle their private skills to global and back.

### Skill Creation — Three Paths

1. **Create with AI** — Navigates to the chat interface with a pre-populated message that triggers the built-in `skill-creator` skill. The AI walks the user through naming, describing, and writing instructions conversationally, then persists the skill via a `save_skill` tool call.
2. **Create Manually** — Opens a form dialog with validated fields (name, description, instructions). Straightforward CRUD.
3. **Import from File** — Opens a file picker for `.zip` files in the Agent Skills open standard format (see Feature 4). Supports single and bulk imports.

The **skill-creator** is itself a global skill seeded during setup. It follows best practices for naming conventions, description quality, and instruction conciseness.

### UI

- **Navigation** — A third top-level tab joins Chat and Documents: `Chat | Documents | Skills`.
- **Skills Page** — Two-column layout mirroring the Documents page. Left sidebar shows a searchable skill list with a "New Skill" dropdown (Create with AI / Create Manually / Import from File). Right side shows the skill editor or an empty state.
- **Skill List** — Shows all visible skills (own + global). Global skills get a badge. Disabled skills render dimmed with a "Disabled" badge.
- **Skill Editor** — Form with name, description (with authoring guidance and character counter), instructions (monospace textarea with markdown hint), building-block files section, enabled/disabled toggle, and action buttons: Save, Delete, Share, Export, Try in Chat.
- **Try in Chat** — Navigates to the chat interface with a message designed to trigger the skill, so users can immediately test it.

### Backend API

Standard REST endpoints following existing patterns:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/skills` | POST | Create a skill |
| `/skills` | GET | List visible skills (own + global) |
| `/skills/{id}` | GET | Get a single skill |
| `/skills/{id}` | PATCH | Update skill properties |
| `/skills/{id}` | DELETE | Delete a skill (own only) |
| `/skills/{id}/share` | PATCH | Toggle global/private |
| `/skills/{id}/export` | GET | Export skill as ZIP in open standard format |
| `/skills/import` | POST | Import skills from ZIP |
| `/skills/{id}/files` | POST | Upload a building-block file |
| `/skills/{id}/files` | GET | List building-block files |
| `/skills/{id}/files/{file_id}` | DELETE | Delete a building-block file |
| `/skills/{id}/files/{file_id}/content` | GET | Get file content or download URL |

### LLM Tools

| Tool | Purpose |
|------|---------|
| `load_skill` | Fetch full instructions for a skill when the user's request matches |
| `save_skill` | Persist a new skill after the AI-guided creation flow |

---

## Feature 2: Skill Building-Block Files

### What It Does

Skills can have files attached to them — Python scripts, brand guidelines, templates, data files, fonts, etc. These are the skill's own resources ("building blocks"), distinct from knowledge base documents. They're stored in dedicated storage and loaded on demand, not indexed for RAG search. If a skill needs to search the knowledge base, that behavior is described in the skill's instructions.

### How It Works

1. **Upload** — Users upload files to a skill from the skill editor UI. Files are stored in a private storage bucket at `{user_id}/{skill_id}/{filename}`.
2. **Discovery** — When `load_skill` runs, the response includes a table listing all attached files (filename, size, type).
3. **On-demand reading** — The LLM uses a `read_skill_file` tool to fetch specific file contents. Text files are returned inline; binary files return metadata with a note that content can't be displayed.
4. **File preview** — Clicking a file in the skill editor opens a side panel showing the file content (for text files) with copy and download buttons.

### LLM Tool

| Tool | Purpose |
|------|---------|
| `read_skill_file` | Read the content of a file attached to a skill |

### UI

- **Building Block Files section** in the skill editor for managing attached files.
- **File list** showing icon, filename, size, and delete button.
- **Upload button** triggering a file picker (only shown for owned skills).
- **File preview panel** — Slides in from the right when a file is clicked. Shows text content in a monospace `<pre>` block, or a "binary file" message with download button. Header has copy, download, and close buttons.

---

## Feature 3: Code Execution Sandbox

### What It Does

Gives the AI the ability to execute Python code in a secure, sandboxed Docker container. This enables skills that generate files (PowerPoint, Excel, CSV, charts), process data, run computations, or perform web scraping.

### Technical Approach

- **Runtime** — Docker containers using the `llm-sandbox` library. Docker is already running locally for Supabase.
- **Session persistence** — Interactive Python sessions (IPython kernel) where variables survive across `run()` calls within a conversation thread. Keyed by thread ID with a configurable TTL (default 30 min) and automatic cleanup.
- **Streaming output** — stdout/stderr are streamed to the frontend via SSE events as the code runs, giving users real-time feedback.
- **File output** — Generated files in `/sandbox/output/` are uploaded to Supabase Storage and served via signed URLs.
- **Feature flag** — `SANDBOX_ENABLED=false` by default. The `execute_code` tool only appears when enabled.
- **Security** — A security policy blocks dangerous operations: `subprocess`, `os.system`, `socket`, `ctypes`, filesystem access outside the sandbox. Network access and filesystem manipulation are restricted.

### Custom Docker Image

Rather than using a vanilla `python:3.11-slim` image (which requires installing libraries on every new session), a custom image pre-installs commonly needed packages:

| Category | Packages |
|----------|----------|
| Document generation | python-pptx, python-docx, openpyxl, fpdf2 |
| Data processing | pandas, numpy |
| Visualization | matplotlib, pillow |
| Templating & formats | jinja2, pyyaml, tabulate |
| Web/utility | requests, beautifulsoup4 |

Users can still install additional packages on-demand via a `libraries` parameter. Installed packages persist for the session lifetime.

### SSE Events

| Event | When |
|-------|------|
| `code_execution_start` | Code begins executing |
| `code_stdout` | Each line of stdout (streaming) |
| `code_stderr` | Each line of stderr (streaming) |
| `code_execution_complete` | Execution finished — includes exit code, timing, generated files |
| `code_execution_error` | Execution failed |

### Database

- **`code_executions`** table — Audit log of all executions (code, stdout, stderr, exit code, timing, status).
- **`sandbox_files`** table — Metadata for files generated by executions.
- **`sandbox-outputs`** storage bucket — Private bucket for generated files, scoped by user.

### Backend API

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/sandbox/files/{file_id}/download` | GET | Get a fresh signed download URL |
| `/sandbox/executions/{execution_id}/files` | GET | List files for an execution |

### LLM Tool

| Tool | Purpose |
|------|---------|
| `execute_code` | Run Python code in a sandboxed container. Accepts code, optional libraries to install, and expected output filenames. |

### UI — Code Execution Panel

An inline component in the chat (similar to the sub-agent panel):

- **Header** — Python badge, status indicator (spinner/check/error), execution time.
- **Code preview** — Collapsible monospace block showing the executed code.
- **Output** — Terminal-style container with dark background. Green text for stdout, red for stderr.
- **Files** — Download cards showing filename, size, and a download link.
- **Error state** — Red alert with error message.

### Lifecycle

- Sessions are created on first code execution in a thread.
- Sessions are cleaned up when their TTL expires (background task runs every 60s).
- Sessions are explicitly closed when a thread is deleted.
- The session manager starts/stops with the application lifespan.

---

## Feature 4: Skills Open Standard (Import/Export)

### What It Does

Bridges the app's database-backed skill model with the Agent Skills open standard (agentskills.io) — an open format adopted by 30+ agent products that defines skills as filesystem directories with a `SKILL.md` file. This makes skills portable between tools.

### Export

Exports a skill as a `.zip` file containing:

```
skill-name/
  SKILL.md          # YAML frontmatter (name, description, metadata) + markdown body (instructions)
  scripts/          # Code files (.py, .js, .sh, etc.)
  references/       # Text/document files (.md, .txt, .pdf, etc.)
  assets/           # Binary/media files (.png, .jpg, .zip, etc.)
```

The `SKILL.md` is generated from the skill's database fields. Attached building-block files are categorized into subdirectories based on MIME type/extension.

### Import

Accepts a `.zip` file upload (max 50 MB) and creates skills in the database:

- Supports single-skill ZIPs (one `SKILL.md` at root or in a named directory) and bulk imports (multiple directories each with their own `SKILL.md`).
- Parses YAML frontmatter from `SKILL.md` to extract skill properties.
- Validates name/description against the app's rules.
- Uploads attached files, stripping the category subdirectory prefix (stored flat in the app's storage).
- Reports per-skill results — naming conflicts are reported as errors without blocking other skills in a bulk import.

### SKILL.md Format

```yaml
---
name: analyzing-sales-data
description: Analyzes sales data and generates reports...
license: MIT
compatibility: agentskills-v1
metadata:
  author: jane
  version: "1.0"
---

# Instructions

Full markdown instructions here...
```

### Backend

A utility module handles parsing and generating `SKILL.md` files and categorizing attached files into the standard directory structure (scripts/references/assets) based on MIME type and extension.

---

## Feature 5: Persistent Tool Memory

### What It Does

Tool call results (search results, file trees, UUIDs, etc.) are persisted across conversation turns. Previously, only user and assistant text messages were loaded from the database — all tool context was lost between turns. This meant the AI couldn't reference data from previous tool calls without re-executing them.

### How It Works

1. **Save results** — After each tool execution, the full result string is stored alongside the existing tool call metadata (name, arguments, status, summary) in the message's JSONB `tool_calls` column. Results are capped at a reasonable size to avoid database bloat.
2. **Rebuild on load** — When loading conversation history for a new turn, tool call messages are reconstructed in the LLM's expected format: assistant message with tool calls, followed by tool result messages, followed by the assistant's text response.
3. **No schema migration** — The `tool_calls` column is already JSONB, so storing the additional `result` and `tool_call_id` fields requires no database changes.

### User-Facing Impact

- The AI can reference UUIDs, search results, file listings, and other data from previous turns without re-executing tools.
- Follow-up questions like "what's the UUID of that file?" work naturally.
- Reduces redundant tool calls, making conversations faster and cheaper.

---

## Cross-Cutting Concerns

### Row-Level Security

All new tables (skills, skill_files, code_executions, sandbox_files) enforce RLS. Users can only see and modify their own data, with the exception that global skills (user_id = NULL) are visible to all authenticated users.

### SSE Streaming

New SSE event types are added for skill activation (`skill_activated`) and code execution (`code_*` events). These follow the existing SSE patterns used by sub-agents and tool calls.

### Feature Flags

Code execution is gated behind `SANDBOX_ENABLED`. When disabled, the `execute_code` tool is not registered and doesn't appear in the system prompt.

### Observability

Code execution uses the existing LangSmith tracing with `@traceable` decorators.

---

## Summary of New LLM Tools

| Tool | Feature | Purpose |
|------|---------|---------|
| `load_skill` | Skills | Load full instructions when a query matches a skill |
| `save_skill` | Skills | Persist a new skill from AI-guided creation |
| `read_skill_file` | Skill Files | Read content of a file attached to a skill |
| `execute_code` | Sandbox | Run Python code in a Docker container |

---