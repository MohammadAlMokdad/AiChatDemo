# AiChatDemo

A tiny, zero-dependency Node.js app that demonstrates the two common ways to put AI on a website: a public visitor chatbot and an admin assistant that edits site content through function calling. Powered by the Google Gemini free tier.

## What it does

| Page | Endpoint | What it shows |
|---|---|---|
| `http://localhost:3000/` | `POST /api/chat` | Public **visitor chatbot**: no tools, rate-limited per IP, answers only from `site-data.json` |
| `http://localhost:3000/admin` | `POST /api/admin/chat` | Token-protected **admin assistant**: changes `site-data.json` through Gemini **function calling** (tool use) |

- The visitor page renders a small studio website from `site-data.json` and adds a chat bubble in the bottom-right corner.
- The admin page shows the live site data next to a chat. Ask for a change in plain English and the assistant applies it with one of four tools: `update_field`, `add_service`, `update_service`, `remove_service`. Each applied change is shown as a tag under the reply.
- `site-data.json` plays the role of a database. Edit something on the admin page, refresh the visitor page, and the change is live.
- The Gemini API key lives only on the server. The browser never sees it.
- If the configured model name returns a 404, the error message lists the models your key can use.

## Tech stack

- Node.js 18+ (built-in `http` and `fetch`, no npm packages)
- Plain HTML, CSS and JavaScript in the browser
- Google Gemini API (`generateContent` over REST, with `functionDeclarations` for tool use)

## Prerequisites

- [Node.js](https://nodejs.org/) 18 or newer
- A free Gemini API key from [Google AI Studio](https://aistudio.google.com/apikey) (no credit card needed)

## Getting started

```bash
git clone https://github.com/MohammadAlMokdad/AiChatDemo.git
cd AiChatDemo
```

There is nothing to install. Set your key and start the server.

**PowerShell**

```powershell
$env:GEMINI_API_KEY = "your-key-here"
npm start
```

**macOS / Linux**

```bash
GEMINI_API_KEY=your-key-here npm start
```

**Using a `.env` file (Node 20.6+)**

```bash
cp .env.example .env   # then put your key in .env
node --env-file=.env server.js
```

Then:

1. Open http://localhost:3000 and click the chat bubble in the bottom-right corner.
2. Open http://localhost:3000/admin. The default admin token is `letmein`. Try:
   - "change the tagline to something bolder"
   - "add a photography service"
   - "rename the studio to Octopus Lab and update the contact email to hi@octopus.example"

## Configuration

All settings are environment variables. See [`.env.example`](.env.example).

| Variable | Default | Purpose |
|---|---|---|
| `GEMINI_API_KEY` | *(required)* | Free key from AI Studio. Without it, the chat endpoints return `503`. |
| `GEMINI_MODEL` | `gemini-flash-latest` | Rolling alias for the newest Flash model (pinned names like `gemini-2.5-flash` get retired for new keys). Pin a specific model if you want stable behavior; on a 404 the error lists the models your key can use. |
| `ADMIN_TOKEN` | `letmein` | Token the admin page sends in the `x-admin-token` header. Change it before showing anyone else. |
| `PORT` | `3000` | HTTP port. |

## API

| Method | Path | Body | Response |
|---|---|---|---|
| `GET` | `/api/site` | | Current contents of `site-data.json` |
| `POST` | `/api/chat` | `{ "messages": [...] }` | `{ "reply": "..." }` |
| `POST` | `/api/admin/chat` | `{ "messages": [...] }`, header `x-admin-token` | `{ "reply": "...", "applied": ["..."] }` |

`messages` is the full conversation so far: an array of `{ "role": "user" | "model", "text": "..." }`, up to 30 messages of up to 4,000 characters each.

## How it works

- **`server.js` → `visitorChat()`** injects the current site data into the system prompt, forwards the conversation, and returns text. It has no tools, so a malicious visitor can talk *at* it but can't change anything.
- **`server.js` → `adminChat()`** declares the four tools as Gemini `functionDeclarations`. When the model responds with a `functionCall` instead of text, the server runs it against `site-data.json`, sends a `functionResponse` back, and loops (up to 5 rounds) until the model answers in plain text.
- **Rate limiter.** The free-tier quota (roughly 10 to 15 requests per minute for the whole key) is shared by *all* visitors, so `/api/chat` is capped at 10 messages per minute per IP. The admin endpoint is protected by the `x-admin-token` header instead.

## Project structure

```
AiChatDemo/
├── server.js          HTTP server, Gemini calls, tool execution, rate limiter
├── site-data.json     Demo "database" read by both chats and edited by the admin assistant
├── public/
│   ├── index.html     Visitor site with the chat widget
│   └── admin.html     Admin assistant with the live data panel
├── .env.example       Environment variable template
└── package.json
```

Note that the admin assistant rewrites `site-data.json` in place, so any edits you make through it show up as changes to that file.

## Before real use

This demo deliberately skips a few things:

- Real authentication for the admin endpoint (use your app's login, not a static token)
- Streaming responses (nicer UX for long answers)
- Persistent per-IP limits that survive restarts, and correct client IPs behind a proxy (`X-Forwarded-For`)
- A confirmation step before destructive tool calls such as `remove_service`

Google may use prompts sent on the **free tier** for training, so don't send anything sensitive. The paid tier and Vertex AI don't have that clause.
