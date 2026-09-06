# Nexum — Backend

A conversational AI agent that searches, filters, and discovers files within a designated Google Drive folder. Built with FastAPI, LangGraph, and supports both Google Gemini and Ollama (local) as LLM providers.

---

## Tech Stack

| Layer | Technology |
|---|---|
| API Server | FastAPI + Uvicorn |
| Agent Framework | LangGraph |
| LLM (cloud) | Google Gemini (`gemini-2.0-flash-preview`) |
| LLM (local) | Ollama (`llama3.2`, `gemma3`) |
| Drive Integration | Google Drive API v3, Service Account auth |
| Config | Pydantic Settings + `.env` |
| Runtime | Python 3.11+, isolated virtualenv |

---

## Project Structure

```
backend/
├── main.py                  # FastAPI app, lifespan manager, /health + /chat routes
├── config.py                # Pydantic Settings, get_settings(), is_placeholder()
├── requirements.txt
├── .env.example             # Template — copy to .env and fill in real values
│
├── agent/
│   ├── graph.py             # LangGraph StateGraph — nodes wired, conditional routing
│   ├── nodes.py             # input_node, llm_node, tool_node, response_node
│   ├── state.py             # AgentState TypedDict
│   └── prompts.py           # SYSTEM_PROMPT — NL to Drive q parameter rules
│
├── services/
│   ├── llm_factory.py       # get_llm() + validate_llm() — Gemini or Ollama
│   └── drive_client.py      # get_drive_service(), list_files()
│
└── tools/
    └── drive_tool.py        # DriveSearchTool (LangChain BaseTool subclass)
```

---

## Agent Flow

```
User message
    │
    ▼
[input_node] → attach conversation history
    │
    ▼
[llm_node] → LLM translates NL to Drive q parameter string
    │
    ▼
[router] ──search──► [tool_node] → Drive API files.list(q=...)
    │                     │
    │chat                 ▼
    │              [response_node] → format markdown reply
    └──────────────────► ▲
```

The LLM always responds in JSON:

```json
{ "action": "search", "q": "mimeType = 'application/pdf'", "explanation": "..." }
{ "action": "chat",   "q": "", "explanation": "Hello! How can I help?" }
```

The `router` reads `action` and conditionally routes to `tool_node` (search) or directly to `response_node` (chat).

---

## Environment Variables

Copy `.env.example` to `.env` and fill in all values.

| Variable | Required | Description |
|---|---|---|
| `GEMINI_API_KEY` | Yes (if using Gemini) | From Google AI Studio |
| `GOOGLE_SA_JSON` | Yes | Full service account JSON as a single-line string |
| `DRIVE_FOLDER_ID` | Yes | ID from your Google Drive folder URL |
| `ACTIVE_LLM` | Yes | `gemini` or `ollama` |
| `GEMINI_MODEL` | No | Default: `gemini-2.0-flash-preview` |
| `OLLAMA_BASE_URL` | No | Default: `http://localhost:11434` |
| `OLLAMA_MODEL` | No | Default: `llama3.2` |

### GOOGLE_SA_JSON formatting (Windows)

The entire service account JSON must be on one line. Run this in PowerShell:

```powershell
$json = Get-Content "C:\path\to\service-account-key.json" -Raw
$oneline = $json -replace "`r`n", "" -replace "`n", ""
Write-Output $oneline
```

Paste the output as: `GOOGLE_SA_JSON={"type":"service_account",...}`

---

## Google Drive Setup

1. **Google Cloud Console**
   - Create a project → Enable **Google Drive API**
   - Go to IAM & Admin → Service Accounts → Create a service account
   - Under Keys tab → Add Key → JSON → download the file

2. **Share your Drive folder**
   - Create a folder in your normal Google Drive
   - Upload test files (PDFs, Docs, Sheets, images)
   - Right-click folder → Share → paste the `client_email` from the JSON key
   - Set role to **Viewer**

3. **Get the Folder ID**
   - Open the folder in Drive
   - Copy the ID from the URL: `drive.google.com/drive/folders/THIS_PART`

---

## Local Development

### First-time setup

```powershell
# From the backend/ folder
python -m venv venv
.\venv\Scripts\Activate.ps1
pip install -r requirements.txt
copy .env.example .env
# Fill in .env with real values
```

### Start the server

```powershell
.\venv\Scripts\Activate.ps1
uvicorn main:app --reload --port 8000
```

The server validates the configured LLM on startup:

```
[startup] active LLM: gemini
[startup] LLM 'gemini' validated OK
```

### Switching LLM providers

Change `ACTIVE_LLM` in `.env` and restart:

```env
ACTIVE_LLM=ollama        # use local Ollama
ACTIVE_LLM=gemini        # use Google Gemini
```

For Ollama, make sure `ollama serve` is running and the model is pulled:

```bash
ollama pull llama3.2
ollama serve
```

---

## API Reference

### GET /health

Returns backend status and active LLM info.

```json
{
  "status": "ok",
  "active_llm": "gemini",
  "ollama_url": "http://localhost:11434",
  "drive_folder_id": "1aBcDeFg..."
}
```

### POST /chat

Send a conversation turn and receive a reply with Drive search results.

**Request**

```json
{
  "messages": [
    { "role": "user", "content": "find all PDFs" }
  ],
  "session_id": "any-string"
}
```

**Response**

```json
{
  "reply": "Found **3** file(s):\n• [Report.pdf](https://...) — PDF, modified 2024-06-01",
  "sources": [
    {
      "id": "1xyz...",
      "name": "Report.pdf",
      "mimeType": "application/pdf",
      "modifiedTime": "2024-06-01T10:00:00Z",
      "webViewLink": "https://drive.google.com/file/d/..."
    }
  ]
}
```

**Note:** The frontend sends the full conversation history on every request. The backend is stateless — no database or session storage is used.

---

## Supported Drive Search Queries

The LLM is trained via the system prompt to produce these `q` strings:

| User says | Drive q parameter |
|---|---|
| find all PDFs | `mimeType = 'application/pdf'` |
| files named budget | `name contains 'budget'` |
| exact file name | `name = 'Q3 Report'` |
| Google Sheets | `mimeType = 'application/vnd.google-apps.spreadsheet'` |
| Google Docs | `mimeType = 'application/vnd.google-apps.document'` |
| Images | `mimeType contains 'image/'` |
| Modified after date | `modifiedTime > '2024-01-01T00:00:00'` |
| Combined | `name contains 'report' and mimeType = 'application/pdf'` |

The folder parent filter (`'{folder_id}' in parents`) is always added automatically by `drive_client.py` — the LLM never needs to include it.

---

## Testing the API (PowerShell)

Save a payload file and post it:

```powershell
# test_payload.json
{"messages":[{"role":"user","content":"find all PDFs"}],"session_id":"test-1"}

curl -X POST http://localhost:8000/chat `
  -H "Content-Type: application/json" `
  --data-binary @test_payload.json
```

---

## Known Limitations (MVP1)

- Full-text search inside documents is not supported (requires Drive content indexing)
- No pagination — results capped at 20 files per query
- Conversation memory is client-side only (session_state in Streamlit)
- No OAuth — uses a single shared Service Account for all Drive access

---
