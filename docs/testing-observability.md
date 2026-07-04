# Testing & Observability

> Overview of CartGenie's testing strategy, error handling patterns, and observability infrastructure.

---

## Error Handling Philosophy

CartGenie follows a **"never go silent"** principle. Every code path must produce a user-facing response, even in failure scenarios.

### Fallback Chain Pattern

The system uses layered fallback chains throughout:

```
Primary Path → Retry → Fallback → Graceful Error Message
```

| Component | Primary | Fallback | Error State |
|---|---|---|---|
| **Intent Parsing** | Groq LLM API | Rule-based regex parser | "I'm not sure how to handle that" |
| **Product Scraping** | Full Playwright scrape | Direct affiliate link | Error code + "try again" |
| **Gift Brainstorming** | LLM-generated categories | Hardcoded context-aware categories | Generic gift suggestions |
| **Search Modifiers** | LLM-generated attributes | Category-specific hardcoded lists | Generic defaults |
| **Web Research** | Yahoo Search scrape | Skip context, LLM uses parameters only | Fallback categories |

### Error Code Traceability

When an unexpected error occurs, the user receives a message like:

> ❌ An unexpected error occurred. Please try again. If the problem persists, quote this error code: `a1b2c3d4`

The 8-character hex code is generated from a UUID and maps to the full stack trace in the server logs, enabling rapid debugging without exposing internals.

---

## Logging Architecture

### Logger Hierarchy
Each Python module creates its own named logger:

```
cartgenie.bot          — Core bot logic, message routing, state transitions
cartgenie.scraper      — Playwright operations, page navigation, data extraction
cartgenie.llm_parser   — LLM API calls, intent classification, fallback triggers
cartgenie.database     — SQLite operations, schema migrations
cartgenie.web_server   — HTTP request handling, mock object lifecycle
cartgenie.web_search   — External search operations
cartgenie.config       — Configuration loading, validation
```

### Output Configuration
- **File handler** — Persistent log file for post-incident analysis
- **Stream handler** — stdout for real-time Docker log monitoring
- **Log level** — Configurable via environment variable (default: INFO)
- **Noise suppression** — Playwright, httpx, and asyncio loggers suppressed to WARNING unless DEBUG mode

### What Gets Logged
| Event | Level | Example |
|---|---|---|
| User message received | INFO | `User 12345 sent message: 'best laptop under 50k'` |
| Intent classification | INFO | `Parsed intent: search, budget=50000, query='laptop'` |
| Scraper navigation | INFO | `Navigating to Amazon product page: /dp/B0GF1YGC2P` |
| API rate limit | WARNING | `Groq API returned 429. Retrying after 1.5s...` |
| Scraper failure | ERROR | `Exception during Amazon scrape: TimeoutError` |
| Unhandled exception | ERROR | Full stack trace with error code |

### What Never Gets Logged
- Bot tokens or API keys
- Database file paths in user-facing contexts
- Full user message content at DEBUG level only (configurable)
- Internal state machine details in user replies

---

## Graceful Degradation

### Scraper Resilience
When Amazon blocks a scraping attempt:
1. The error is logged with full context
2. The user receives a clear error message
3. A direct affiliate link to the product is included so the user can still access it
4. The conversation state is cleanly reset

### API Resilience
When the Groq API is unavailable:
1. First retry after 1.5 seconds (for 429 rate limits)
2. If still failing, fall back to rule-based parser
3. Log the fallback trigger for monitoring
4. Continue processing with reduced accuracy but full functionality

### State Machine Safety
- In-progress states are kept in temporary memory
- States automatically clear on cancellation, completion, or bot restart
- If a user sends a text message while the bot awaits a button tap, the pending flow is cancelled and the new message is processed fresh

---

## Input Validation

| Input | Validation | On Failure |
|---|---|---|
| **Bot token** | Regex: `^\d{6,}:[A-Za-z0-9_-]{20,}$` | Startup crash with clear error |
| **Amazon URLs** | Regex extraction + trailing punctuation stripping | Ignored if invalid |
| **Target price** | Numeric validation (positive number) | Re-prompt with error message |
| **Budget** | Regex with currency/multiplier support (k, lakh) | Treated as no budget |
| **Callback data** | Pattern matching against registered handlers | Ignored if unrecognised |

---

## Performance Considerations

- **Parallel scraping** — Comparison and search flows scrape multiple products via `asyncio.gather()`
- **Semaphore bounding** — Background price checks use configurable concurrency limits
- **Connection reuse** — `httpx.AsyncClient` for LLM API calls with connection pooling
- **Lazy loading** — Reviews are extracted via programmatic scrolling rather than separate page loads
- **Session hashing** — Web chat sessions use MD5 hashing for O(1) user ID derivation
