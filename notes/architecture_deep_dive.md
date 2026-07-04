# Architecture Deep Dive

> This document provides a detailed technical overview of CartGenie's system architecture. The implementation resides in a private codebase; this is an architecture-level summary.

---

## System Overview

CartGenie is built as an **async-first Python application** that serves two client channels (Telegram and Web) through a unified bot core. The system is designed around four primary subsystems:

1. **Conversation Engine** — State machine that manages multi-step user flows
2. **Scraping Pipeline** — Headless browser automation for product data extraction
3. **Intelligence Layer** — LLM-powered NLP with rule-based fallback
4. **Data Persistence** — Lightweight relational storage for users, trackers, and wishlists

---

## Component Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        TG["📱 Telegram App"]
        WEB["🌐 Web Browser"]
    end

    subgraph "Entry Points"
        TGAPI["Telegram Bot API<br/><i>Long Polling</i>"]
        AIOHTTP["aiohttp Server<br/><i>Port 5000</i>"]
    end

    subgraph "Bot Core"
        SM["State Machine<br/><i>ConversationHandler</i>"]
        IR["Intent Router<br/><i>Message → Handler</i>"]
        AFF["Affiliate Engine<br/><i>URL Rewriter</i>"]
    end

    subgraph "Intelligence"
        LLM["LLM Parser<br/><i>Groq / Llama 3.3 70B</i>"]
        FB["Fallback Parser<br/><i>Rule-Based</i>"]
        SA["Sentiment Analyser<br/><i>Theme Voting</i>"]
    end

    subgraph "Data Access"
        SCRAPER["Playwright Scraper<br/><i>Product Details</i>"]
        SEARCH["Search Scraper<br/><i>Amazon Search Pages</i>"]
        WEBSEARCH["Web Research<br/><i>Yahoo Search</i>"]
    end

    subgraph "Storage"
        DB[("SQLite<br/><i>aiosqlite</i>")]
    end

    subgraph "External"
        AMAZON["Amazon India"]
        YAHOO["Yahoo Search"]
        GROQ["Groq API"]
    end

    TG <--> TGAPI
    WEB <--> AIOHTTP
    TGAPI --> SM
    AIOHTTP -->|Mock Update Objects| SM
    SM --> IR
    IR --> LLM
    LLM -.->|Fallback| FB
    IR --> SCRAPER
    IR --> SEARCH
    IR --> WEBSEARCH
    IR --> SA
    IR --> AFF
    SM --> DB
    LLM --> GROQ
    SCRAPER --> AMAZON
    SEARCH --> AMAZON
    WEBSEARCH --> YAHOO
```

---

## Data Flow: Message Processing

Every incoming message follows this pipeline:

```mermaid
sequenceDiagram
    participant U as User
    participant E as Entry Point
    participant SM as State Machine
    participant URL as URL Extractor
    participant NLP as NLP Engine
    participant S as Scraper
    participant SA as Sentiment
    participant DB as Database
    participant R as Reply

    U->>E: Send message
    E->>SM: Route to handler

    alt State machine is awaiting input
        SM->>SM: Consume as slot value
    else Fresh message
        SM->>URL: Extract Amazon URLs
        alt URLs found
            URL->>SM: Route by URL count
            Note over SM: 1 URL → Track/Review<br/>2-3 URLs → Compare
        else No URLs
            SM->>NLP: Parse intent via LLM
            NLP-->>SM: Intent + extracted data
        end
    end

    SM->>S: Scrape product data
    S-->>SM: Product details + reviews
    SM->>SA: Analyse sentiment
    SA-->>SM: Pros, cons, verdict
    SM->>DB: Save tracker/wishlist
    SM->>R: Format & send reply
    R->>U: Deliver response
```

---

## Concurrency Model

CartGenie runs on a **single asyncio event loop** with the following concurrent operations:

| Component | Concurrency Mechanism | Detail |
|---|---|---|
| **Message Handling** | asyncio coroutines | Each user message is processed as an async coroutine |
| **Product Scraping** | Parallel `asyncio.gather()` | Comparison scrapes run in parallel for 2-3 products |
| **Background Price Checks** | `asyncio.Semaphore` | Bounded concurrency (configurable, default: 2 concurrent checks) |
| **Price Check Loop** | `asyncio.sleep()` loop | Periodic background task (configurable interval, default: 1 hour) |
| **Web Server** | aiohttp `AppRunner` | Serves web chat alongside the bot on the same event loop |

### Instance Safety
A file-based lock (`fcntl.LOCK_EX | LOCK_NB`) ensures only one bot process can run at a time. This prevents the Telegram `Conflict` error that occurs when multiple processes poll the same bot token.

---

## Web Chat Bridge

The web interface achieves full feature parity with Telegram through an innovative **mock object translation layer**:

```mermaid
graph LR
    subgraph "Web Client"
        JS["app.js<br/><i>Session Management</i>"]
    end

    subgraph "Web Server"
        API["REST API<br/><i>/api/chat, /api/callback</i>"]
        MOCK["Mock Layer<br/><i>MockUpdate, MockMessage,<br/>MockCallbackQuery</i>"]
    end

    subgraph "Bot Core"
        PTB["python-telegram-bot<br/><i>Application.process_update()</i>"]
    end

    JS -->|POST JSON| API
    API --> MOCK
    MOCK -->|"Mock Telegram Objects"| PTB
    PTB -->|"Replies stored in mock.replies[]"| MOCK
    MOCK -->|Serialised JSON| API
    API -->|JSON Response| JS
```

**Key design decision:** Instead of building a separate handler pipeline for the web, we create mock `Update`, `Message`, and `CallbackQuery` objects that extend the real `python-telegram-bot` classes. These mock objects override reply methods (`reply_html`, `reply_text`, `reply_photo`) to capture responses into a list, which is then serialised and returned to the web client.

This approach means:
- **Zero code duplication** — Every feature, flow, and edge case works identically on both channels
- **Automatic feature parity** — New Telegram features automatically work on the web
- **Session isolation** — Web sessions are mapped to stable integer user IDs via MD5 hashing

---

## Database Schema

```mermaid
erDiagram
    USERS {
        int user_id PK
        text username
        text joined_date
    }
    TRACKERS {
        int id PK
        int user_id FK
        text asin
        text url
        text title
        real target_price
        real current_price
        text status
        bool is_wishlist
    }
    USERS ||--o{ TRACKERS : "has"
```

The database is intentionally lightweight — SQLite with async access via `aiosqlite`. It stores:
- **Users**: Telegram user IDs and usernames
- **Trackers**: Active price alerts and wishlist items with current/target prices and status

---

## Affiliate Monetisation Flow

Every product URL that leaves the system passes through a centralised sanitisation pipeline:

1. **Strip existing parameters** — Remove any pre-existing tracking, marketing, or affiliate tags
2. **Inject affiliate tag** — Append the configured Amazon Associate ID
3. **Return clean URL** — The final URL contains only the product path and affiliate tag

This applies uniformly to: comparison charts, review summaries, price alert notifications, search results, and gift recommendations.
