# ⚙️ CartGenie — System Overview

This document provides a high-level overview of how CartGenie is built, how it is deployed, and the core engineering decisions made during its design.

---

## 🏗️ System Architecture

CartGenie is designed to be **async-first and lightweight**, routing traffic from both Telegram and a web interface into a single, unified bot application core.

### Request Routing Block Diagram

```mermaid
graph TD
    TG[📱 Telegram User] -->|Telegram Bot API| BotCore[🤖 Unified Bot Application]
    Web[🌐 Web Browser User] -->|Web API Bridge| BotCore
    
    BotCore -->|1. Understand Intent| LLM[🧠 Groq LLM / Fallback Parser]
    BotCore -->|2. Scrape Products| Scraper[🕷️ Playwright Stealth Scraper]
    BotCore -->|3. Save Trackers| DB[(💾 SQLite Database)]
```

### Dynamic Web-to-Bot Bridge

Instead of writing separate application flows for Telegram and Web chat channels, the server translates browser HTTP requests into mock Telegram event objects. 

This design choice ensures:
- **100% Feature Parity:** Every feature updates simultaneously on both platforms.
- **Zero Code Duplication:** The web frontend interacts with the exact same state machine and modules as the Telegram bot.

---

## 💬 Conversation State Flowchart

CartGenie manages multi-step interactions (such as collecting gift details or configuring a target price alert) using a structured conversation state machine.

```mermaid
graph TD
    Msg([User Message]) --> URLCheck{Contains Amazon URL?}
    
    URLCheck -->|Yes| RouteURL[Route directly to Comparison / Review Summary]
    URLCheck -->|No| StateCheck{Awaiting Slot Input?}
    
    StateCheck -->|Yes| SaveSlot[Save input into conversation context]
    StateCheck -->|No| LLMParse[Parse Intent via LLM / Fallback Parser]
    
    SaveSlot --> Action[Execute Action]
    LLMParse --> Action
    
    Action --> Response([Send Response & Reset/Update State])
```

- **Escape Hatch:** If a user types a new command (or clicks "Cancel") while a conversational flow is in progress, the active state is instantly cleared, and the new input takes priority.

---

## 🚀 VPS Deployment & Infrastructure

CartGenie is deployed on a Linux VPS using a containerized, self-hosting stack aimed at keeping operations entirely free of service fees.

```mermaid
graph LR
    User[Client] -->|HTTPS :443| Caddy[Caddy Reverse Proxy]
    Caddy -->|Proxy :5000| Docker[Docker Container]
    Docker --> DB[(SQLite Database File)]
```

- **Docker Container:** Bundles the application along with all Chromium dependencies required for Playwright scraping.
- **Caddy Reverse Proxy:** Handles SSL/TLS termination automatically, routing traffic securely to the web interface.
- **SQLite Persistence:** Mounts a local database file to preserve user alerts and wishlists between container restarts.
- **Single-Instance Guard:** An automatic process lock ensures only one instance of the bot runs at a time, preventing Telegram API conflicts.

---

## 💡 Core Design Decisions & Tradeoffs

### 1. Custom Stealth Scraping vs. Paid APIs
- **Tradeoff:** Commercial product data APIs are highly reliable but charge monthly fees.
- **Decision:** To maintain a $0 budget, CartGenie uses a custom Playwright configuration utilizing user-agent cycling, persistent browser context sharing, and stealth headers to query Amazon directly. If blocked, the bot gracefully falls back to sharing clean affiliate links directly.

### 2. LLM Parsing with Local Fallbacks
- **Tradeoff:** Relying solely on external LLM APIs (like Groq) risks downtime due to API rate limits or network issues.
- **Decision:** All user queries are processed by the LLM for high accuracy, but are supported by a local regex-based keyword parser. If the API limits are hit, the local fallback immediately routes key actions so the bot remains functional.

### 3. Sentence-level Sentiment Tagging
- **Tradeoff:** Full LLM review processing is slow and expensive for long review text.
- **Decision:** Reviews are filtered using a custom keyword lexicon that tags sentences into positive and negative themes (such as build quality, battery life, or shipping speed) and sums votes to deliver a general recommendation verdict.
