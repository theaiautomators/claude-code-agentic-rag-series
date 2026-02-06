# Episode 2: Build AI Agents that Explore Like Claude Code

> **Knowledge Base Explorer — GSD Planning Files**
> 
> *Part of the Agentic RAG with Claude Code series*

This folder contains **GSD (Get Shit Done)** planning artifacts generated from the `/gsd:new-project` command. These files provide a structured approach to building out the Knowledge Base Explorer — giving your AI agent filesystem-like tools (`ls`, `tree`, `grep`, `glob`, `read`) to navigate and search a hierarchical knowledge base, just like Claude Code explores codebases.

## Install GSD First

Before using these files, you need to install **Get Shit Done** — a meta-prompting and spec-driven development system for Claude Code.

**Repository:** [github.com/glittercowboy/get-shit-done](https://github.com/glittercowboy/get-shit-done)

```bash
# Install globally (recommended)
npx get-shit-done-cc --global

# Or install locally in your project
npx get-shit-done-cc --local
```

After installation, restart Claude Code to load the slash commands.

## What's Inside

```
.planning/
├── PROJECT.md      # High-level project definition, requirements, context & decisions
├── REQUIREMENTS.md # Detailed requirements with traceability to roadmap phases
├── ROADMAP.md      # Phased development plan with success criteria
└── STATE.md        # Current progress and session continuity
```

## How to Use These Files

### 1. Copy to Your Project

Bring these files into a `.planning` folder at the root of your existing codebase:

```bash
# From your project root
mkdir -p .planning
cp -r path/to/knowledgebase-video/.planning/* .planning/
```

### 2. Map Your Existing Codebase

Run the map command to have GSD analyze your current codebase structure:

```
/gsd:map-codebase
```

This creates a codebase map that GSD uses to understand your existing architecture, conventions, and patterns.

### 3. Start Planning or Discussing

Once your codebase is mapped, you can either:

- **Discuss a phase** to refine requirements and approach:
  ```
  /gsd:discuss-phase 1
  ```

- **Plan a phase** to generate detailed implementation plans:
  ```
  /gsd:plan-phase 1
  ```

## Prerequisites

### Existing Codebase (Episode 1)

This build picks up where Episode 1 left off. You have two options:

1. **Build along with Episode 1** — Follow the [Episode 1 video](https://github.com/theaiautomators/claude-code-agentic-rag-masterclass) to build the foundation yourself
2. **Skip ahead** — The complete Episode 1 codebase is available exclusively to members of [The AI Automators](https://www.theaiautomators.com) community

The foundation includes:

- React/Vite frontend + FastAPI backend
- Supabase (Postgres with pgvector)
- Document ingestion via Docling
- Hybrid search with RRF fusion
- Tool calling framework

The GSD system will guide you through adding the Knowledge Base Explorer layer on top of this existing foundation.

## GSD Workflow Overview

```
/gsd:new-project     →  Creates PROJECT, REQUIREMENTS, ROADMAP, STATE files
/gsd:map-codebase    →  Analyzes existing code to inform planning
/gsd:discuss-phase N →  Interactive refinement of a phase
/gsd:plan-phase N    →  Generates detailed implementation plans
/gsd:execute-plan    →  Implements a specific plan
/gsd:progress        →  Check where you are and what's next
/gsd:help            →  Show all commands and usage guide
```

For the full list of commands and configuration options, see the [GSD documentation](https://github.com/glittercowboy/get-shit-done).

---

## Join the Community

Connect with hundreds of builders creating production-grade AI and RAG systems at [**The AI Automators**](https://www.theaiautomators.com). Get help when you're stuck, share your progress, and see what others are building.

---

**Agentic RAG with Claude Code Series**
- [Episode 1: Build the RAG foundation](https://github.com/theaiautomators/claude-code-agentic-rag-masterclass) (React + FastAPI + Supabase)
- Episode 2: Build AI Agents that Explore Like Claude Code ← *You are here*
