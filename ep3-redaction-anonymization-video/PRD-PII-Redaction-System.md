# PRD: PII Redaction System for Agentic RAG Applications

**Version:** 1.0
**Status:** Reference Implementation Complete

---

## 1. Executive Summary

This document specifies a PII (Personally Identifiable Information) redaction system for agentic RAG (Retrieval-Augmented Generation) applications. The system ensures that real PII never reaches cloud LLM providers while maintaining a natural, transparent user experience. Users interact with real data; the system invisibly anonymizes data before it leaves the local environment and de-anonymizes responses before they reach the user.

The system uses Microsoft Presidio for NER-based PII detection, generates realistic surrogate values via Faker for reversible entity types, and applies irreversible hard-redaction placeholders for highly sensitive entity types (credit cards, SSNs, etc.). Entity resolution (clustering name variants like "Danny" and "Daniel Walsh" into the same identity) is handled via configurable strategies: algorithmic name-parsing, local LLM inference, or disabled entirely.

---

## 2. Goals & Non-Goals

### Goals

- **Privacy by default**: No real PII is sent to any cloud/remote LLM endpoint
- **Invisible to the user**: The redaction layer is transparent -- users see real names, real data, and never encounter surrogates or placeholders (except for hard-redacted entity types)
- **Conversation-scoped consistency**: Within a single conversation, the same real entity always maps to the same surrogate, preserving coherence across multi-turn dialogue
- **Entity resolution**: Name variants, nicknames, abbreviations, and titles referring to the same person are clustered and assigned consistent surrogate families
- **Hard redaction for sensitive types**: Highly sensitive entity types (credit cards, SSNs, bank numbers, etc.) are irreversibly replaced with `[ENTITY_TYPE]` placeholders
- **Full agent pipeline coverage**: Redaction applies across the entire agent pipeline -- main chat, sub-agents, tool calls, title generation, metadata extraction
- **Configurable**: Entity types, detection thresholds, resolution strategy, and scan behaviors are configurable via environment variables
- **Observability**: Full tracing of redaction operations via pluggable tracing providers

### Non-Goals

- **Multi-language support**: English only for this version
- **Filename/metadata PII**: PII in filenames and document metadata fields is not redacted (deferred)
- **Audit UI**: No user-facing redaction audit trail or transparency dashboard (deferred)
- **Custom entity types**: No support for domain-specific custom entity recognizers beyond Presidio's built-in set
- **User-facing toggle**: Redaction is always-on when configured; there is no per-user or per-conversation toggle

---

## 3. Architecture Overview

### 3.1 Core Principle

```
User sees REAL data <--[de-anonymize]--> Database stores REAL data --[anonymize on-the-fly]--> Cloud LLM sees FAKE data
```

- Documents are stored with their **original, unmodified content**
- Embeddings are generated locally (no PII sent to remote embedding services)
- PII anonymization happens **at chat time**, not at ingestion time
- De-anonymization happens before data reaches the user
- A per-conversation entity registry maintains consistent mappings throughout a conversation

### 3.2 Data Flow: Document Ingestion

```
Upload file --> extract text --> generate embeddings LOCALLY --> store raw content in database
```

Key points:
- No anonymization during ingestion -- documents are stored as-is
- Embeddings must be generated using a **local** embedding model to avoid sending PII to remote services
- Metadata extraction (title, summary, topics) should prefer a local LLM to avoid PII leakage

### 3.3 Data Flow: Chat (Main Agent)

```
1. User sends message (contains real PII)
2. Load conversation history from database (contains real PII)
3. ANONYMIZE: Run all messages through the redaction pipeline
   - Presidio detects PII entities
   - Hard-redact entities --> [ENTITY_TYPE] placeholders
   - Surrogate entities --> Faker-generated realistic fake values
   - Entity resolution clusters name variants
   - Registry updated with new mappings
4. Retrieve relevant document chunks for context
   - ANONYMIZE retrieved chunk content before including in LLM prompt
5. Send anonymized messages + anonymized context to cloud LLM
6. BUFFER complete LLM response (do not stream raw surrogates to user)
7. DE-ANONYMIZE: Replace surrogates back to real values
   - Exact match replacement from registry
   - Fuzzy/algorithmic matching for name variants the LLM may have reformatted
8. Stream de-anonymized response to user via SSE
9. Store de-anonymized (real) response in conversation history
```

### 3.4 Data Flow: Tool Calls

| Tool Type | Input Handling | Output Handling |
|-----------|---------------|-----------------|
| **Document search** | LLM generates query with surrogates; search runs against real content, results anonymized before returning to LLM | Results anonymized before LLM sees them |
| **SQL queries** | De-anonymize the SQL query (replace surrogates with real values) before execution | Anonymize query results before returning to LLM |
| **Text search/grep** | De-anonymize search pattern before execution | Results anonymized in the next chat loop iteration |
| **Sub-agents** | Anonymize all content sent to the sub-agent's LLM | De-anonymize the sub-agent's complete response before displaying to user |

### 3.5 Data Flow: Sub-Agents & Multi-Agent Pipelines

Sub-agents (e.g., document analyzers, knowledge base explorers) receive the same redaction service instance as the parent agent:

- **Input anonymization**: Document content and user queries are anonymized before the sub-agent's cloud LLM call
- **Output de-anonymization**: The sub-agent's complete response is de-anonymized before display
- **Streaming behavior**: When redaction is active, sub-agent reasoning events are buffered (not streamed in real-time) to prevent surrogate leakage. The full de-anonymized result is emitted once complete
- **Nested agents**: If an explorer agent invokes a sub-agent, the redaction service is threaded through. Tool arguments from the explorer LLM are de-anonymized before execution (so searches find real content), and tool results are re-anonymized before going back to the explorer's LLM
- **When redaction is off**: All behavior reverts to normal streaming with no performance overhead

### 3.6 Data Flow: Auxiliary LLM Calls

Any LLM call that processes raw user/document content should prefer a local LLM:

| Operation | Approach |
|-----------|----------|
| **Thread/conversation title generation** | Use local LLM; fall back to cloud LLM if unavailable |
| **Document metadata extraction** | Use local LLM; fall back to cloud LLM if unavailable |
| **Entity resolution** | Always local LLM (by design) |
| **Missed PII scan** | Always local LLM (by design) |

---

## 4. Functional Requirements

### FR-1: PII Detection

**FR-1.1**: The system MUST detect the following PII entity types using NER (Named Entity Recognition):
- Person names
- Email addresses
- Phone numbers
- Physical locations
- Dates and times
- URLs
- IP addresses
- Credit card numbers
- Social Security Numbers (SSN)
- Individual Taxpayer Identification Numbers (ITIN)
- Bank account / routing numbers
- IBAN codes
- Passport numbers
- Driver's license numbers
- Medical license numbers
- Cryptocurrency addresses

**FR-1.2**: Detection MUST use a two-pass approach with different confidence thresholds:
- **High threshold (default 0.7)** for surrogate entity types -- reduces false positives that would create unnecessary Faker surrogates
- **Low threshold (default 0.3)** for hard-redact entity types -- false positives are cheap (just a `[TYPE]` placeholder) while false negatives leak real sensitive data

**FR-1.3**: Entity types MUST be configurable into two groups via environment variables:
- **Surrogate entities**: Get reversible Faker-generated replacements (default: PERSON, EMAIL, PHONE, LOCATION, DATE_TIME, URL, IP_ADDRESS)
- **Hard-redact entities**: Get irreversible `[ENTITY_TYPE]` placeholders (default: CREDIT_CARD, SSN, ITIN, BANK_NUMBER, IBAN, CRYPTO, PASSPORT, DRIVER_LICENSE, MEDICAL_LICENSE)

**FR-1.4**: Detection MUST NOT produce false positives on UUID strings. UUID segments (e.g., hex patterns like `b8a3` within `d301a04a-9ca5-4e92-b8a3-40a769d6ceac`) MUST be excluded from detection results. This is critical because document/record IDs are typically UUIDs and corrupting them breaks tool call pipelines.

**FR-1.5**: Detection thresholds MUST be configurable via environment variables.

### FR-2: Anonymization (Real --> Surrogate)

**FR-2.1**: Surrogate entity types MUST be replaced with realistic fake values generated by Faker (e.g., real name "John Smith" becomes fake name "Marcus Rodriguez").

**FR-2.2**: Hard-redact entity types MUST be replaced with `[ENTITY_TYPE]` format placeholders (e.g., `4532-1234-5678-9012` becomes `[CREDIT_CARD]`). These are never reversed.

**FR-2.3**: Surrogate generation MUST be collision-free: a generated surrogate MUST NOT match any existing real value or existing surrogate in the registry. Retry with a new Faker value (up to a reasonable limit) before falling back to a non-realistic identifier.

**FR-2.4**: Surrogate names MUST be gender-matched using a gender detection library. If the original name is detected as female, the surrogate must be female (and vice versa). Ambiguous or unknown names fall back to random gender.

**FR-2.5**: Surrogate surname generation MUST NOT reuse surname or first name components that appear in any real person entity within the same conversation. This prevents the fuzzy de-anonymization pass from corrupting real names (see FR-5.4).

**FR-2.6**: The replacement operation MUST be a programmatic string replacement, not LLM-generated text reproduction. The LLM is never asked to rewrite text with surrogates; the system performs surgical find-and-replace.

### FR-3: Conversation-Scoped Entity Registry

**FR-3.1**: Each conversation/thread MUST maintain its own entity registry mapping real values to surrogates. Registries are NOT shared across conversations.

**FR-3.2**: The registry MUST be persisted to the database and loaded when a conversation is resumed.

**FR-3.3**: Lookups MUST be case-insensitive (e.g., "john smith" and "John Smith" map to the same surrogate).

**FR-3.4**: Within a conversation, the same real entity MUST always produce the same surrogate (consistency guarantee).

**FR-3.5**: Hard-redacted entities are NOT stored in the registry (they are irreversible `[TYPE]` placeholders with no reverse mapping).

### FR-4: Entity Resolution

**FR-4.1**: The system MUST support three entity resolution modes, configurable via environment variable:

| Mode | Description | Dependencies |
|------|-------------|-------------|
| **Algorithmic** | Name parsing + clustering algorithms, no API calls | Name parser, nickname dictionary, string similarity library |
| **LLM** | Local LLM inference for entity clustering | Local LLM endpoint (e.g., LM Studio, Ollama) |
| **None** | No clustering; each unique string gets its own surrogate | None |

**FR-4.2**: **Algorithmic mode** MUST implement:
- Name parsing into components (title, first, middle, last, suffix)
- Nickname/diminutive resolution (e.g., "Danny" --> canonical "Daniel")
- Union-Find clustering with three merge rules:
  - Same canonical first + last name --> merge (high confidence)
  - Last-name-only entity merges with a cluster if exactly one cluster has that last name (medium confidence)
  - First-name-only entity merges with a cluster if exactly one cluster has that canonical first name (low confidence)
- Sub-surrogate derivation: if "Daniel Walsh" --> "Marcus Smith", then "Danny" --> "Marcus", "Mr. Walsh" --> "Mr. Smith", "Walsh" --> "Smith"

**FR-4.3**: **LLM mode** MUST:
- Send only PERSON entities (not emails, phones, etc.) to the local LLM
- NOT send the full source text -- only the names and existing registry
- Return a flat mapping of original names to surrogates
- Fall back to direct candidate mapping if the local LLM is unavailable or returns invalid output

**FR-4.4**: Entity resolution MUST only apply to PERSON entities. Other entity types (emails, phones, URLs, etc.) use exact-match normalization:
- Emails: lowercase
- Phone numbers: strip non-digits, normalize country prefix
- URLs: lowercase, strip trailing slash and www prefix
- IBANs: strip spaces, uppercase
- Others: lowercase + strip whitespace

### FR-5: De-Anonymization (Surrogate --> Real)

**FR-5.1**: The system MUST replace surrogates back to real values before displaying to the user.

**FR-5.2**: De-anonymization MUST use case-insensitive matching (to handle LLMs outputting surrogates in different cases, e.g., "MARCUS SMITH" header formatting).

**FR-5.3**: De-anonymization MUST support a fuzzy matching pass for name variants the LLM may have introduced (e.g., the LLM receives "Marcus Smith" but outputs "Mr. Smith" or "M. Smith"). This pass can use:
- **Algorithmic mode**: Name parsing + string similarity scoring (e.g., Jaro-Winkler >= 0.85)
- **LLM mode**: Local LLM call to identify fuzzy references
- **None mode**: Exact match only (no fuzzy pass)

**FR-5.4**: De-anonymization MUST use a placeholder-based pipeline to prevent surname collision corruption:
1. **Phase 1**: Replace all known surrogates with unique placeholder tokens (e.g., `<<PH_0001>>`)
2. **Phase 2**: Run fuzzy matching on the placeholder'd text (real names are hidden, so fuzzy matching cannot corrupt them)
3. **Phase 3**: Resolve all placeholders to real values

This prevents the scenario where a Faker-generated surname component matches a real person's surname and the fuzzy pass incorrectly replaces it.

**FR-5.5**: Hard-redacted placeholders (`[CREDIT_CARD]`, `[US_SSN]`, etc.) MUST survive de-anonymization unchanged. They are never in the registry and are never reversed.

### FR-6: LLM Response Buffering

**FR-6.1**: When redaction is active, the cloud LLM's text response MUST be fully buffered before de-anonymization and delivery to the user. This is accuracy-over-latency: partial de-anonymization of streamed chunks risks leaking surrogates or corrupting names split across chunk boundaries.

**FR-6.2**: The system SHOULD emit SSE status events to indicate progress:
- `{"type": "redaction_status", "stage": "anonymizing"}` -- before the LLM call
- `{"type": "redaction_status", "stage": "deanonymizing"}` -- before de-anonymization
- The frontend should display a subtle loading indicator during these phases

**FR-6.3**: For sub-agents, reasoning events MUST be suppressed during generation when redaction is active (they would contain surrogates). The full de-anonymized result is emitted as a single batch when complete.

### FR-7: LLM System Prompt Guidance

**FR-7.1**: The main agent's system prompt MUST include instructions telling the LLM that content may contain anonymized placeholder values, and that it MUST reproduce names, emails, phone numbers, locations, dates, times, and URLs in the **exact format** they appear in the source material. This prevents the LLM from abbreviating or reformatting surrogates (e.g., turning "Marcus Smith" into "M. Smith"), which would cause de-anonymization failures.

### FR-8: Secondary Missed-PII Scan

**FR-8.1**: The system SHOULD support an optional secondary LLM-based scan that checks anonymized text for hard-redact entities that the primary NER engine missed (e.g., credit card numbers in unusual formats). This is controlled via an environment variable toggle.

**FR-8.2**: The missed-PII scan MUST run independently of the entity resolution mode. It fires for all modes (algorithmic, LLM, none) when enabled.

**FR-8.3**: Any missed entities found MUST be validated against the configured hard-redact entity set. Invalid entity types are discarded.

**FR-8.4**: If missed entities are found and replaced, the primary NER engine SHOULD re-run on the modified text to recalculate surrogate entity positions (since the text has changed).

### FR-9: Local LLM Configuration

**FR-9.1**: The system MUST support configuring a local LLM endpoint (base URL + model name) for entity resolution, missed-PII scanning, title generation, and metadata extraction.

**FR-9.2**: Configuration MUST be available through both environment variables and a settings API/UI.

**FR-9.3**: The local LLM endpoint MUST use an OpenAI-compatible API (supporting `/v1/chat/completions`). This is compatible with LM Studio, Ollama, and similar local inference servers.

**FR-9.4**: No API key is required for local LLM calls (local model).

**FR-9.5**: All LLM calls that process raw PII (title generation, metadata extraction) MUST prefer the local LLM and fall back to the cloud LLM only if the local LLM is not configured or fails.

---

## 5. Non-Functional Requirements

### NFR-1: Performance

- Presidio NER engine MUST be loaded once at application startup and shared across all requests (it is expensive to initialize)
- Per-conversation entity registries are loaded from the database on first use and kept in memory for the duration of the request
- The gender detection model and nickname dictionary MUST be lazy-loaded singletons
- Anonymization adds latency proportional to the amount of text and number of detected entities. For typical chat messages (< 2000 tokens), anonymization should complete in < 500ms
- The LLM response buffering strategy trades latency for accuracy. Users will perceive a delay before seeing the response (mitigated by status events)

### NFR-2: Security

- The entity registry is a system-level table. It MUST NOT be directly accessible to end users via the API. Access is controlled at the application layer
- Row-level security on the registry table should restrict access to the service role only
- Real PII remains in the database (chat history, documents). Database-level security (encryption at rest, access controls) remains the responsibility of the infrastructure layer
- The redaction system protects against PII leakage to **cloud LLM providers**, not against database compromise

### NFR-3: Reliability

- If the local LLM is unavailable, entity resolution MUST gracefully degrade:
  - LLM mode falls back to direct 1:1 candidate mapping (no clustering)
  - Missed-PII scan is skipped
  - Title generation and metadata extraction fall back to cloud LLM
- If Presidio fails to initialize (e.g., missing NER model), the application SHOULD still start but log a clear error. Redaction calls should fail gracefully rather than crash the application
- Concurrent requests writing to the same conversation registry MUST be serialized (async lock) to prevent race conditions

### NFR-4: Observability

- All redaction operations SHOULD be traced via the application's observability provider
- The tracing system MUST support pluggable providers (e.g., LangSmith, Langfuse) switchable via environment variable
- Debug-level logging SHOULD capture: entities detected, surrogates assigned, fuzzy matches found, missed PII scan results, UUID filter drops

---

## 6. Configuration Reference

| Variable | Default | Description |
|----------|---------|-------------|
| `ENTITY_RESOLUTION_MODE` | `llm` | Entity resolution strategy: `llm`, `algorithmic`, or `none` |
| `PII_SURROGATE_ENTITIES` | `PERSON,EMAIL_ADDRESS,PHONE_NUMBER,LOCATION,DATE_TIME,URL,IP_ADDRESS` | Comma-separated entity types that get reversible Faker surrogates |
| `PII_REDACT_ENTITIES` | `CREDIT_CARD,US_SSN,US_ITIN,US_BANK_NUMBER,IBAN_CODE,CRYPTO,US_PASSPORT,US_DRIVER_LICENSE,MEDICAL_LICENSE` | Comma-separated entity types that get irreversible `[TYPE]` placeholders |
| `PII_SURROGATE_SCORE_THRESHOLD` | `0.7` | Presidio confidence threshold for surrogate entities |
| `PII_REDACT_SCORE_THRESHOLD` | `0.3` | Presidio confidence threshold for hard-redact entities (lower is safer) |
| `PII_MISSED_SCAN_ENABLED` | `true` | Whether to run the secondary LLM-based missed-PII scan |
| `LOCAL_LLM_BASE_URL` | *(none)* | OpenAI-compatible endpoint for local LLM (e.g., `http://localhost:1234/v1`) |
| `LOCAL_LLM_MODEL` | *(none)* | Model name for local LLM |
| `TRACING_PROVIDER` | *(none)* | Observability provider: `langsmith`, `langfuse`, or empty (disabled) |

---

## 7. Technical Approach: Key Decisions

### 7.1 Why Chat-Time Anonymization (Not Ingestion-Time)

Early iterations attempted anonymizing documents at ingestion time and storing both original and anonymized versions. This was abandoned because:

- **Double-anonymization chains**: Faker-generated surrogates are realistic names that the NER engine re-detects on subsequent passes, creating cascading anonymization
- **No entity resolution at ingestion**: Without conversation context, there's no way to know that "M. Thompson" and "Margaret Thompson" are the same person. Each mention gets a different surrogate
- **Chunk boundary alignment**: Anonymized text has different lengths than original (fake names differ in length), causing chunk boundary misalignment
- **Pipeline complexity**: Every downstream component needed both original and anonymized variants of content

The chat-time approach is simpler: documents are stored raw, embeddings are generated locally, and anonymization only happens when content is about to leave the local environment for a cloud LLM.

### 7.2 Why Conversation-Scoped Registries (Not Global)

A global vault (mapping real --> surrogate for all users and conversations) has scaling and isolation problems:

- Cross-conversation surrogate leakage: if "John Smith" is always "Marcus Rodriguez" globally, an attacker could potentially reverse-engineer mappings
- Vault growth: unbounded growth as more entities are encountered
- No conversation context for entity resolution

Conversation-scoped registries keep mappings isolated. Each conversation starts fresh, and entity resolution has full conversation context for clustering decisions.

### 7.3 Why Buffer-and-De-anonymize (Not Stream-and-De-anonymize)

Early iterations used a streaming de-anonymizer that buffered SSE chunks and detected surrogate boundaries in real-time. This was replaced with full response buffering because:

- Surrogates split across SSE chunk boundaries created edge cases
- Fuzzy de-anonymization (matching "Mr. Smith" to "Marcus Smith") requires seeing the complete response
- The latency trade-off is acceptable when paired with progress status events

### 7.4 Why Two-Pass Presidio Detection

A single confidence threshold creates a dilemma:
- High threshold (0.7): catches person names reliably but misses context-dependent entities like bank numbers and passport numbers that Presidio scores at 0.4-0.65
- Low threshold (0.3): catches everything but creates excessive false positives for person names

Two passes solve this: high threshold for surrogate entities (where false positives create bogus Faker surrogates that confuse the conversation) and low threshold for hard-redact entities (where false positives just produce harmless `[TYPE]` placeholders).

### 7.5 Why Placeholder-Based De-anonymization

A naive de-anonymization approach (replace surrogates with real values, then run fuzzy matching) is vulnerable to surname collision corruption:

1. Faker generates "Aaron Thompson DDS" as surrogate for "Maria Vasquez"
2. Exact match replaces "Brittney Ferguson" with "Margaret Thompson" (introduces real "Thompson" into text)
3. Fuzzy pass finds "Thompson" and replaces it with "Vasquez" (because "Thompson" is a surname component of surrogate "Aaron Thompson DDS")
4. Result: "Margaret Vasquez" -- corrupted

The placeholder approach prevents this by never having real names and surrogates co-exist in the text during fuzzy matching.

---

## 8. Dependencies

| Component | Purpose | Notes |
|-----------|---------|-------|
| **Microsoft Presidio** (analyzer + anonymizer) | NER-based PII detection | Core detection engine |
| **spaCy** (transformer model, e.g., `en_core_web_trf`) | NER model backend for Presidio | Loaded at startup, shared across requests |
| **Faker** | Realistic surrogate value generation | Used for all surrogate entity types |
| **nameparser** | Parse person names into components | Used by algorithmic entity resolution |
| **nicknames** (or equivalent diminutive dictionary) | Resolve nicknames to canonical names | Used by algorithmic entity resolution |
| **rapidfuzz** (or equivalent string similarity) | Jaro-Winkler similarity for fuzzy name matching | Used by algorithmic de-anonymization |
| **gender-guesser** | Detect likely gender from first names | Used for gender-matched surrogate generation |
| **Local LLM server** (LM Studio, Ollama, etc.) | OpenAI-compatible local inference | Used for LLM-mode entity resolution, missed-PII scan, title generation, metadata extraction |
| **Tracing provider** (Langfuse, LangSmith, etc.) | Observability and debugging | Optional but recommended |

---

## 9. Risks & Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| NER false negatives (PII not detected) | Real PII sent to cloud LLM | Two-pass detection with low threshold for sensitive types; optional LLM-based secondary scan |
| NER false positives (non-PII detected as PII) | Unnecessary surrogates confuse conversation | Higher threshold for surrogate entities; UUID post-filter |
| Faker surrogate collides with real name | De-anonymization corruption | Collision check against all known originals before accepting surrogate; surname component cross-check |
| LLM reformats surrogates | De-anonymization fails, user sees surrogates | System prompt instructions to preserve exact formatting; fuzzy de-anonymization as safety net |
| Local LLM unavailable | No entity resolution, no missed-PII scan | Graceful degradation to direct mapping; cloud LLM fallback for title/metadata |
| Response buffering latency | Perceived delay before user sees response | SSE status events for progress indication |
| Large entity registry | Memory pressure in long conversations | Acceptable for typical conversations; consider LRU eviction for extreme cases |

---

## 10. Future Considerations

- **Multi-language support**: Extend Presidio configuration to support additional languages
- **Filename/metadata PII**: Redact PII in document filenames, titles, and metadata fields
- **Audit trail UI**: Show users what was redacted, with ability to review anonymization decisions
- **Custom entity types**: Allow domain-specific entity recognizers (e.g., internal employee IDs, project codes)
- **Per-user redaction preferences**: Allow users to configure which entity types to redact
- **Cross-conversation entity consistency**: Optional global registry for users who prefer consistent surrogates across conversations
- **Registry encryption at rest**: Encrypt entity registry mappings in the database so that real PII values are not stored in plaintext. This would protect against database compromise scenarios where an attacker gains read access to the registry and can reverse all surrogate mappings
- **Streaming de-anonymization**: Revisit streaming approach if response buffering latency becomes unacceptable (would require more sophisticated chunk boundary handling)
