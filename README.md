# Taco Bell UK — AI Chatbot (n8n)

An AI-powered customer service chatbot for Taco Bell UK, built on n8n. Customers chat via a public web widget and get instant, accurate answers about the menu, nutrition, and allergens — with every conversation logged to Google Sheets.

---

## Architecture

```
Customer (Chat Widget)
  └─▶ When Chat Message Received  (n8n Chat Trigger — public URL)
        └─▶ Restaurant AI Agent   (OpenAI GPT-5-mini + LangChain)
              ├─ Conversation Memory    (last 10 messages)
              ├─ Menu Items Tool        (Google Sheets → "UK Menu Items" tab)
              ├─ Nutrition Info Tool    (Google Sheets → "Nutrition (UK)" tab)
              └─ Knowledge Retriever   (in-memory vector store)
        └─▶ Extract Customer Data (Set node)
              └─▶ Save to Customer Interactions (Google Sheets log)
```

| Component | Detail |
|---|---|
| AI Model | OpenAI GPT-5-mini |
| Embeddings | OpenAI (text-embedding) |
| Memory | Buffer window — 10 messages |
| Menu data | Google Sheets (TacoBell_UK_Menu) |
| Conversation log | Google Sheets (customer respnse) |
| Trigger | n8n Chat Widget (public, embeddable) |

---

## Prerequisites

- [n8n](https://n8n.io) account (cloud or self-hosted)
- OpenAI API key
- Google account with Sheets access
- Two Google Sheets set up (see below)

---

## Google Sheets Setup

### Sheet 1 — Menu & Nutrition Data
Create a Google Sheet named **TacoBell_UK_Menu** with these tabs:

| Tab name | Contents |
|---|---|
| `UK Menu Items` | Item name, category, price, description, dietary flags |
| `Nutrition (UK)` | Item name, calories, protein, fat, carbs, allergens |

### Sheet 2 — Conversation Log
Create a Google Sheet named **customer respnse** (or any name) with a tab `Sheet1` containing these columns:

| Data and time | sessionID | customer message | AI response |
|---|---|---|---|

---

## Import & Configure Workflow

### 1. Import
1. Open your n8n instance
2. Go to **Workflows → Import**
3. Upload `workflow.json` from this repo

### 2. Set up Credentials

**OpenAI**
1. Go to **Credentials → New → OpenAI**
2. Paste your API key
3. Assign it to both `OpenAI Chat Model` and `OpenAI Embeddings` nodes

**Google Sheets**
1. Go to **Credentials → New → Google Sheets OAuth2**
2. Complete the OAuth flow
3. Assign the credential to: `Menu Items Tool`, `Nutrition Information Tool`, `Save to Customer Interactions`

### 3. Link Google Sheets
In each of these nodes, re-select your actual sheet:
- `Menu Items Tool` → select your TacoBell_UK_Menu sheet, tab: `UK Menu Items`
- `Nutrition Information Tool` → same sheet, tab: `Nutrition (UK)`
- `Save to Customer Interactions` → select your conversation log sheet, tab: `Sheet1`

### 4. Load the Knowledge Base (Optional)
To use the vector store for additional context, create a separate loader workflow:
1. Use a **Google Sheets** node to read your menu data
2. Connect to a **Simple Vector Store** node in INSERT mode
3. Set the memory key to: `restaurant_menu_data`
4. Run it once to populate the store

### 5. Activate
Toggle the **Active** switch in the top-right of the workflow editor.

---

## Accessing the Chat Widget

Once active, the chat widget is available at:
```
https://your-instance.app.n8n.cloud/webhook/YOUR_WEBHOOK_ID/chat
```

You can embed it in any website using an `<iframe>`.

---

## AI System Prompt

The full system prompt is in [`prompts/system_prompt.md`](prompts/system_prompt.md). Edit it in n8n via:

`Restaurant AI Agent` node → **Options** → **System Message**

Key rules:
- Always use the Menu Items Tool before answering menu questions
- Always use the Nutrition Information Tool for allergen/calorie queries
- Keep responses brief and conversational
- Never fabricate menu information

---

## Files

```
taco bell AI agent/
├── workflow.json          # Full n8n workflow export (import this)
├── prompts/
│   └── system_prompt.md   # AI system prompt (source of truth)
├── .env.example           # Required environment variable keys
├── .gitignore
└── README.md
```

---

## Updating Menu Data

When the menu, prices, or nutrition info changes:
1. Update the relevant tab in your **TacoBell_UK_Menu** Google Sheet
2. The workflow reads live from Google Sheets — no redeployment needed
3. Re-run your vector store loader workflow if you use the knowledge base

---

## Conversation Logging

Every chat exchange is logged to your Google Sheet with:
- Timestamp (ISO 8601)
- Session ID
- Customer message
- AI response

Use this data to identify common questions and improve the system prompt over time.
