# Restaurant AI Agent — n8n Workflow

An AI-powered restaurant chatbot built with n8n that handles customer queries about the Taco Bell UK menu via **Telegram**. Customers can ask about menu items, nutrition, allergens, ingredients, and general restaurant information and receive natural, conversational replies from an AI persona named **Shiva**.

## Overview

- Triggered by incoming Telegram messages
- Powered by OpenAI `gpt-4o-mini` via a LangChain agent
- Reads live menu data from a Google Spreadsheet (5 sheets)
- Answers general restaurant questions (hours, locations, delivery) from a **Google Docs FAQ document**
- Remembers conversation history per user using **Postgres** (100-message window)
- Replies back to the same Telegram chat automatically

## Architecture

```
Telegram Trigger
        │
        ▼
AI Agent ("Shiva") — gpt-4o-mini
    ├── Postgres Chat Memory     (100-msg persistent history per chat.id)
    ├── menu_lookup              → Google Sheets: UK Menu Items
    ├── nutrition_lookup         → Google Sheets: Nutrition (UK)
    ├── allergen_check           → Google Sheets: Allergens (UK)
    ├── ingredients_lookup       → Google Sheets: Ingredients (UK)
    ├── summary_lookup           → Google Sheets: Summary
    └── FAQ_lookup               → Google Docs: Restaurant FAQ document
        │
        ▼
Send a text message  (Telegram reply to same chat)
```

## Prerequisites

- n8n instance (cloud or self-hosted, v1.0+)
- Telegram Bot (created via [@BotFather](https://t.me/botfather))
- OpenAI API key
- Google account with Sheets and Docs access
- PostgreSQL database (for persistent chat memory)

## Google Spreadsheet Structure

One spreadsheet with **5 sheets** required:

| Sheet name | Purpose |
|---|---|
| `UK Menu Items` | Item name, description, price, category, dietary flags (vegetarian/vegan/gluten-free) |
| `Nutrition (UK)` | Calories (kcal/kJ), fat, sat fat, carbs, sugars, fibre, protein, salt per item |
| `Allergens (UK)` | 14 EU/UK allergens per item — YES / MAY CONTAIN / FREE |
| `Ingredients (UK)` | Full ingredient declaration per item |
| `Summary` | Category-level summaries for filter queries |

## Google Docs FAQ Document

A single Google Doc containing general restaurant information:

- Opening hours
- Locations
- Delivery options and partners
- Policies and services
- Any other non-menu information you want the bot to answer

The agent reads this document via the `FAQ_lookup` tool whenever a customer asks a general question about the restaurant.

## Setup

### 1. Create a Telegram Bot

1. Open Telegram and message [@BotFather](https://t.me/botfather)
2. Send `/newbot` and follow the prompts
3. Copy the **bot token** — you'll use it as your Telegram credential in n8n

### 2. Set Up PostgreSQL

Create a database and table for chat memory:

```sql
CREATE TABLE restaurent_AI_agent_chat_histories (
    id SERIAL PRIMARY KEY,
    session_id TEXT NOT NULL,
    message JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);
```

> Note: The table name contains an intentional spelling quirk (`restaurent`) — keep it exactly as shown to match the workflow config.

### 3. Create Google Spreadsheet

Create a spreadsheet with the 5 sheets listed above and note the spreadsheet ID from the URL:
`https://docs.google.com/spreadsheets/d/YOUR_SPREADSHEET_ID/edit`

### 4. Create Google Docs FAQ Document

Create a Google Doc with your restaurant FAQ content and note the document URL:
`https://docs.google.com/document/d/YOUR_DOCUMENT_ID/edit`

### 5. Configure n8n Credentials

In n8n go to **Settings → Credentials** and create:

| Credential type | Used by |
|---|---|
| Telegram API | Telegram Trigger, Send a text message |
| OpenAI API | OpenAI Chat Model |
| Google Sheets OAuth2 API | All 5 Google Sheets tool nodes |
| Google Docs OAuth2 API | FAQ_lookup |
| Postgres | Postgres Chat Memory |

### 6. Import the Workflow

1. In n8n go to **Workflows → Import from file**
2. Select `workflow.json` from this repo
3. Open each node that shows a credential warning and select your credential
4. Update the `documentId` in each of the 5 Google Sheets tool nodes with your actual spreadsheet ID
5. Update the `documentURL` in the `FAQ_lookup` node with your Google Doc URL

### 7. Activate & Test

1. Click **Activate** to publish the workflow
2. Open Telegram and send your bot a message — e.g. "hi" or "what tacos do you have?" or "what time do you open?"

## Environment Variables

Copy `.env.example` to `.env` and fill in your values. The `.env` file is gitignored and must never be committed.

```bash
cp .env.example .env
```

## Workflow Nodes

| Node | Type | Purpose |
|---|---|---|
| Telegram Trigger | Telegram Trigger | Entry point — fires on every incoming message |
| AI Agent | LangChain Agent | Orchestrates tools and generates responses as "Shiva" |
| OpenAI Chat Model | LLM | Powers the agent (`gpt-4o-mini`) |
| Postgres Chat Memory | Memory | Persistent 100-message history per Telegram chat ID |
| menu_lookup | Google Sheets Tool | Reads item names, prices, categories, dietary flags |
| nutrition_lookup | Google Sheets Tool | Reads calories, macros, energy values |
| allergen_check | Google Sheets Tool | Reads 14 allergens per item (YES/MAY/FREE) |
| ingredients_lookup | Google Sheets Tool | Reads full ingredient declarations |
| summary_lookup | Google Sheets Tool | Reads category summaries for filter queries |
| FAQ_lookup | Google Docs Tool | Reads FAQ document for general restaurant questions |
| Send a text message | Telegram | Sends the agent's reply back to the customer |

## System Prompt

The full agent system prompt is in [prompts/system_prompt.md](prompts/system_prompt.md).

The agent persona is **Shiva** — warm, friendly, uses UK English spelling, emoji-driven tone. The prompt contains 8 strict rules covering greetings, tool routing, response format, allergen handling, and what the agent must never do.

## Tool Routing

The agent decides which tool(s) to call based on the customer's question:

| Question type | Tool(s) called |
|---|---|
| Item name, price, availability | `menu_lookup` |
| Ingredients | `ingredients_lookup` |
| Allergen / allergy question | `allergen_check` |
| Calories, macros, nutrition | `nutrition_lookup` |
| Category browse ("what burritos?") | `menu_lookup` + `summary_lookup` |
| Filter ("vegetarian options?") | `menu_lookup` + `summary_lookup` + `allergen_check` |
| General restaurant info (hours, location, delivery…) | `FAQ_lookup` |
| "Tell me everything" about an item | All four: menu + ingredients + allergen + nutrition |

## Notes

- Chat memory is stored in Postgres and persists across n8n restarts (unlike in-memory buffers)
- Each Telegram chat ID gets its own isolated conversation thread
- All tool nodes use manual descriptions — the agent knows exactly when to call each tool
- The FAQ_lookup tool reads the entire Google Doc each time — keep the document concise for performance
- The chat interface is Telegram-only — no web widget in this version
