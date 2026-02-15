# Episode 3: PII Redaction & Anonymization for Agentic RAG

This episode extends the Agentic RAG system with a **privacy-first PII redaction layer** that ensures sensitive data never reaches cloud LLM providers.

## What It Is

A PII redaction system that sits between your RAG application and cloud LLMs. Users interact with real data—the system invisibly anonymizes content before it leaves your local environment and de-anonymizes responses before they reach the user.

**The user never sees fake data. The LLM never sees real data.**

```
User sees REAL data <--[de-anonymize]--> Database stores REAL data --[anonymize on-the-fly]--> Cloud LLM sees FAKE data
```

## What You'll Build

- **NER-based PII detection** using Microsoft Presidio
- **Reversible anonymization** with Faker-generated realistic surrogates
- **Hard redaction** for highly sensitive types (SSN, credit cards, bank numbers)
- **Entity resolution** to cluster name variants ("Danny", "Daniel Walsh", "Mr. Walsh")
- **Full pipeline coverage** across chat, sub-agents, and tool calls

## Tech Stack Additions

| Component | Tech |
|-----------|------|
| PII Detection | Microsoft Presidio + spaCy |
| Surrogate Generation | Faker |
| Name Parsing | nameparser, rapidfuzz, gender-guesser |
| Local LLM | LM Studio / Ollama |

## Prerequisites

- Completed Episodes 1-2 (base RAG system)
- Local LLM server for entity resolution

## Documentation

See the [PRD](./PRD-PII-Redaction-System.md) for complete technical specifications, configuration options, and architecture details.

## Community

Join [The AI Automators community](https://www.theaiautomators.com/) to connect with other builders creating production-grade AI and RAG systems.
