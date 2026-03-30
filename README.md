# Taco Bell UK AI Chatbot — n8n Workflow

An AI-powered restaurant chatbot built with n8n that handles customer queries about the Taco Bell UK menu, nutrition, and allergens in real time.

## Overview

This workflow provides a public chat interface where customers can:
- Ask about menu items, prices, and availability
- Get dietary information (vegetarian, vegan, gluten-free)
- Look up calories, allergens, and ingredients
- Receive natural, conversational responses

All conversations are automatically logged to Google Sheets for analytics.

## Architecture

```
Chat Trigger (Public Widget)
        │
        ▼
Restaurant AI Agent (LangChain)
    ├── OpenAI Chat Model (gpt-4o-mini)
    ├── Conversation Memory (10-message window)
    ├── Menu Items Tool → Google Sheets (UK Menu Items)
    └── Nutrition Information Tool → Google Sheets (Nutrition UK)
        │
        ▼
Extract Customer Data (Set Node)
        │
        ▼
Save to Customer Interactions (Google Sheets)
```

A **Restaurant Knowledge Base** (in-memory vector store with OpenAI Embeddings) is also wired in for semantic retrieval via the Knowledge Retriever node.

## Prerequisites

- n8n instance (cloud or self-hosted, v1.0+)
- OpenAI API key
- Google account with Sheets access
- Two Google Spreadsheets (see Setup below)

## Setup

### 1. Create Google Spreadsheets

**Menu Spreadsheet** — two sheets required:

| Sheet name | Purpose |
|---|---|
| `UK Menu Items` | Item name, price, category, dietary flags |
| `Nutrition (UK)` | Calories, protein, fat, allergens, ingredients |

**Customer Interactions Spreadsheet** — one sheet with these columns:

| Column | Description |
|---|---|
| `Data and time` | ISO timestamp of the exchange |
| `sessionID` | Unique session identifier |
| `customer message` | What the customer asked |
| `AI response` | What the agent replied |

### 2. Configure n8n Credentials

Inside your n8n instance go to **Settings → Credentials** and add:

| Credential type | Used by |
|---|---|
| OpenAI API | OpenAI Chat Model, OpenAI Embeddings |
| Google Sheets OAuth2 API | Menu Items Tool, Nutrition Tool, Save to Customer Interactions |

### 3. Import the Workflow

1. In n8n, go to **Workflows → Import from file**
2. Select `workflow.json` from this repo
3. After import, open each node that references a credential and select your own credential
4. Update the `documentId` in these three nodes with your actual spreadsheet IDs:
   - **Menu Items Tool** → your Menu spreadsheet ID
   - **Nutrition Information Tool** → your Menu spreadsheet ID
   - **Save to Customer Interactions** → your Interactions spreadsheet ID

### 4. Activate & Test

1. Click **Activate** to enable the workflow
2. Open the **When Chat Message Received** node and copy the **Chat URL**
3. Open the URL in a browser and send a test message (e.g. "What tacos do you have?")

## Environment Variables

Copy `.env.example` to `.env` and fill in your values. The `.env` file is gitignored and must never be committed.

```bash
cp .env.example .env
```

See `.env.example` for all required keys.

## Workflow Nodes

| Node | Type | Purpose |
|---|---|---|
| When Chat Message Received | Chat Trigger | Public entry point, loads previous session |
| Conversation Memory | Memory Buffer | Retains last 10 messages |
| OpenAI Chat Model | LLM | Powers the agent (gpt-4o-mini) |
| Restaurant AI Agent | LangChain Agent | Orchestrates tools and generates responses |
| Menu Items Tool | Google Sheets Tool | Reads UK menu data |
| Nutrition Information Tool | Google Sheets Tool | Reads nutrition/allergen data |
| Restaurant Knowledge Base | Vector Store | In-memory semantic index (key: `restaurant_menu_data`) |
| OpenAI Embeddings | Embeddings | Creates vectors for the knowledge base |
| Knowledge Retriever | Retriever | Semantic search over the vector store |
| Extract Customer Data | Set | Formats data for logging |
| Save to Customer Interactions | Google Sheets | Appends each exchange to the log sheet |

## System Prompt

The full agent system prompt is in [prompts/system_prompt.md](prompts/system_prompt.md).

## Notes

- The in-memory vector store (`restaurant_menu_data`) resets on each n8n restart. To persist knowledge, populate it from a separate workflow on startup.
- The chat widget is public — no authentication is required to access it.
- All conversation data logged to Google Sheets should be handled in compliance with applicable privacy regulations (GDPR, etc.).
