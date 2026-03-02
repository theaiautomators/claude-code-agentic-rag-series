# Episode 4: Agent Skills & Code Execution Sandbox

This episode transforms the Agentic RAG system into a **customizable AI agent platform** with reusable skills, file attachments, and sandboxed code execution.

## What It Is

A skills system that lets users teach the AI specialized behaviors—legal review, sales analysis, report generation—plus a secure Python sandbox for generating files, processing data, and running computations.

## What You'll Build

- **Agent Skills** — Reusable behavior units with name, description, and full instructions using progressive disclosure (lightweight catalog in system prompt, full instructions loaded on demand)
- **Skill Building-Block Files** — Attach Python scripts, templates, data files, and other resources to skills
- **Code Execution Sandbox** — Run Python in secure Docker containers with streaming output and file generation
- **Skills Open Standard** — Import/export skills using the agentskills.io format for cross-tool portability
- **Persistent Tool Memory** — Tool results survive across conversation turns

## Key Features

| Feature | Description |
|---------|-------------|
| **Skill Discovery** | LLM scans a lightweight catalog, loads full instructions on demand |
| **Three Creation Paths** | Create with AI, create manually, or import from file |
| **Private & Global Skills** | Own your skills or share them with all users |
| **Session Persistence** | Python variables survive across code executions within a thread |
| **File Generation** | Create PowerPoint, Excel, CSV, charts—served via signed URLs |

## New LLM Tools

| Tool | Purpose |
|------|---------|
| `load_skill` | Fetch full instructions when a query matches a skill |
| `save_skill` | Persist a new skill from AI-guided creation |
| `read_skill_file` | Read content of a file attached to a skill |
| `execute_code` | Run Python in a sandboxed Docker container |

## Tech Stack Additions

| Component | Tech |
|-----------|------|
| Code Sandbox | Docker + llm-sandbox library |
| Document Generation | python-pptx, python-docx, openpyxl, fpdf2 |
| Data Processing | pandas, numpy, matplotlib |

## Prerequisites

- Completed Episodes 1-3
- Docker running locally

## Documentation

See the [PRD](./PRD-Skills-Sandbox.md) for complete technical specifications.

## Community

Join [The AI Automators community](https://www.theaiautomators.com/) to connect with other builders creating production-grade AI and RAG systems.
