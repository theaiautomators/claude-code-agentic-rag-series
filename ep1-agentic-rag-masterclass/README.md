# Episode 1: Claude Code Agentic RAG Masterclass

This episode follows along with **The AI Automators' Claude Code Agentic RAG Masterclass**.

## Source Repository

**GitHub:** [theaiautomators/claude-code-agentic-rag-masterclass](https://github.com/theaiautomators/claude-code-agentic-rag-masterclass)

## What It Is

A hands-on course where you collaborate with Claude Code to build a full-featured RAG system. You're not the one writing code—Claude is. Your job is to guide it, understand what you're building, and course-correct when needed.

**You don't need to know how to code.** You do need to be technically minded and willing to learn about APIs, databases, and system architecture.

## What You'll Build

- **Chat interface** with threaded conversations, streaming, tool calls, and subagent reasoning
- **Document ingestion** with drag-and-drop upload and processing status
- **Full RAG pipeline**: chunking, embedding, hybrid search, reranking
- **Agentic patterns**: text-to-SQL, web search, subagents with isolated context

## Tech Stack

| Layer          | Tech                                            |
| -------------- | ----------------------------------------------- |
| Frontend       | React, TypeScript, Tailwind, shadcn/ui, Vite    |
| Backend        | Python, FastAPI                                 |
| Database       | Supabase (Postgres + pgvector + Auth + Storage) |
| Doc Processing | Docling                                         |
| AI Models      | Local (LM Studio) or Cloud (OpenAI, OpenRouter) |
| Observability  | LangSmith                                       |

## The 8 Modules

1. **App Shell** — Auth, chat UI, managed RAG with OpenAI Responses API
2. **BYO Retrieval + Memory** — Ingestion, pgvector, switch to generic completions API
3. **Record Manager** — Content hashing, deduplication
4. **Metadata Extraction** — LLM-extracted metadata, filtered retrieval
5. **Multi-Format Support** — PDF, DOCX, HTML, Markdown via Docling
6. **Hybrid Search & Reranking** — Keyword + vector search, RRF, reranking
7. **Additional Tools** — Text-to-SQL, web search fallback
8. **Subagents** — Isolated context, document analysis delegation

## Getting Started

1. Clone the [source repo](https://github.com/theaiautomators/claude-code-agentic-rag-masterclass)
2. Install Claude Code
3. Open in your IDE (Cursor, VS Code, etc.)
4. Run `claude` in the terminal
5. Use the `/onboard` command to get started

## Community

Join [The AI Automators community](https://www.theaiautomators.com/) to connect with other builders creating production-grade AI and RAG systems.
