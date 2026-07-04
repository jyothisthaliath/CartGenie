# Changelog

All notable changes to CartGenie are documented here. This is a sanitised, public-facing summary.

---

## [2.0] — 2026-07-02

### 🎨 UI Overhaul
- **Dark/Light Theme Toggle** — Added a theme switcher with persistent preference storage, supporting a premium dark mode and a clean light theme across all UI components.
- **Responsive Mobile Layout** — Migrated the sidebar into a collapsible hamburger drawer menu for screens under 768px, ensuring full usability on mobile devices.
- **Visual Polish** — Refined glassmorphism card styles, gradient message bubbles, custom scrollbar theming, and smooth transition animations.

### 🌐 Web Chat Enhancements
- Interactive `/command` links rendered as stylish outline buttons within chat messages.
- Product recommendation photo cards with contained image display and caption overlay.

---

## [1.0] — 2026-06-29

### 🚀 Core Platform Launch
- **Product Comparison** — Side-by-side analysis for 2–3 Amazon product links with mathematical recommendation scoring (logarithmic price scaling, trust modifiers, sentiment modifiers).
- **Review Summarisation** — Sentence-level sentiment analysis with theme-based pro/con extraction and purchase verdict scoring.
- **Price Tracking & Alerts** — Background price monitoring with target-price notifications and wishlist drop alerts via Telegram push messages.
- **Search & Discover** — Natural language product search with LLM-powered intent parsing, budget extraction, accessory filtering, and proximity-weighted value ranking.
- **Gift Finder** — Multi-step gift curation flow: slot filling → live web research → LLM brainstorming → targeted Amazon search.
- **Web Chat Interface** — Browser-based chat bridge mirroring full Telegram bot functionality via mock Telegram object translation.

### 🏗️ Infrastructure
- **Docker Deployment** — Containerised with Playwright base image for consistent browser automation.
- **Caddy Reverse Proxy** — Automatic HTTPS with Let's Encrypt TLS certificates.
- **Single-Instance Guard** — File-lock mechanism preventing duplicate polling sessions.

### 🧠 AI & NLP
- **Groq LLM Integration** — Llama 3.3 70B for intent classification, budget/modifier extraction, and gift brainstorming.
- **Rule-Based Fallback** — Complete fallback parser ensuring zero-downtime intent routing when the LLM API is unavailable.
- **Category Anchoring** — Baseline price floors and accessory keyword filtering per product category to prevent search pollution.

### 💰 Monetisation
- **Affiliate Link Engine** — Centralised URL sanitiser that strips existing tracking parameters and injects affiliate tags into every outgoing product link.

---

## Pre-Release

### Architecture & Design
- Conversation flow modelling with state machine design.
- Database schema for users, trackers, and wishlist items.
- Scraper stealth mechanisms: cookie persistence, rotating user agents, WebDriver flag removal.
- Structured logging with per-module loggers and error code traceability.
