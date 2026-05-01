# RithwikBairu_QuantitativeEngineer
# StreamVerse Insights Assistant

An AI-powered internal analytics assistant for StreamVerse Entertainment. Ask business questions in plain English — the system queries your database, searches internal PDF reports, and delivers synthesized answers with charts.

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Browser (React)                          │
│  ┌──────────────┐  ┌──────────────────────┐  ┌──────────────┐  │
│  │  Sidebar     │  │    Chat Panel        │  │  Insights    │  │
│  │  Quick Qs    │  │  Messages + Charts   │  │  Panel       │  │
│  │  History     │  │  Tool Trace          │  │  Bar/Pie     │  │
│  └──────────────┘  └──────────────────────┘  └──────────────┘  │
└──────────────────────────────┬──────────────────────────────────┘
                               │ HTTP  /api/chat
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│                     FastAPI Backend                             │
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              AI Orchestrator (orchestrator.py)           │  │
│  │                                                          │  │
│  │  1. Build system prompt with DB schema                   │  │
│  │  2. Send to Groq-hosted LLM with tool definitions        │  │
│  │  3. Execute tool calls the model requests                │  │
│  │  4. Loop until the model returns final answer            │  │
│  │  5. Return answer + tool trace + chart data              │  │
│  └──────────────────────────────────────────────────────────┘  │
│           │                 │                  │                │
│           ▼                 ▼                  ▼                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  sql_tool    │  │  pdf_tool    │  │  csv_tool            │  │
│  │              │  │              │  │                      │  │
│  │  SELECT-only │  │  Keyword     │  │  Pandas analytics    │  │
│  │  SQLite      │  │  search over │  │  chart-ready output  │  │
│  │  queries     │  │  PDF chunks  │  │  8 metric types      │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────────────────┘  │
│         │                 │                                     │
│         ▼                 ▼                                     │
│  ┌──────────────┐  ┌──────────────┐                            │
│  │  SQLite DB   │  │  PDF Files   │                            │
│  │  6 tables    │  │  5 reports   │                            │
│  │  from CSVs   │  │  reportlab   │                            │
│  └──────────────┘  └──────────────┘                            │
└─────────────────────────────────────────────────────────────────┘
                               │
                        Groq API
        meta-llama/llama-4-scout-17b-16e-instruct
```

---

## Quick Start (Windows — No Docker)

### Prerequisites
- Python 3.11+
- Node.js 20+
- A Groq API key (free tier available at https://console.groq.com/keys)

### Step 1 — Clone / download the project
```
git clone <your-repo-url>
cd entertainment-assistant
```

### Step 2 — Backend setup
```cmd
cd backend

:: Create virtual environment
python -m venv venv
venv\Scripts\activate

:: Install dependencies
pip install -r requirements.txt

:: Set your API key
copy .env.example .env
::: In .env, set GROQ_API_KEY to your Groq key (free tier)
:: Now open .env in Notepad and replace "your_anthropic_api_key_here" with your real key

:: Generate all fake data (CSVs + PDFs)
python scripts/seed_data.py

:: Start the backend
uvicorn main:app --reload --port 8000
```

You should see: `StreamVerse backend ready ✓`
Test it: open http://localhost:8000/health in your browser — you should see `{"status":"ok"}`

### Step 3 — Frontend setup (new terminal)
```cmd
cd frontend
npm install
npm run dev
```

Open http://localhost:5173 — the app is running!

---

## Running with Docker (Recommended for Submission)

```cmd
:: From the project root
:: Create a .env file with your API key
echo GROQ_API_KEY=your_key_here > .env

docker-compose up --build
```

- Frontend: http://localhost:3000
- Backend API: http://localhost:8000
- API docs: http://localhost:8000/docs

---

## Project Structure

```
entertainment-assistant/
├── backend/
│   ├── main.py                    # FastAPI entry point
│   ├── requirements.txt
│   ├── .env.example               # Copy → .env and add API key
│   ├── routers/
│   │   ├── chat.py                # POST /api/chat
│   │   ├── analytics.py           # POST /api/analytics
│   │   └── ingest.py              # POST /api/ingest/reload
│   ├── tools/
│   │   ├── sql_tool.py            # SELECT-only DB queries
│   │   ├── pdf_tool.py            # Keyword search over PDFs
│   │   └── csv_tool.py            # Pandas analytics + charting
│   ├── ai/
│   │   └── orchestrator.py        # Groq tool-calling agentic loop
│   ├── db/
│   │   └── database.py            # SQLite setup + safe execute
│   ├── scripts/
│   │   └── seed_data.py           # Generates all CSV + PDF data
│   └── data/
│       ├── csvs/                  # 6 generated CSV files
│       └── pdfs/                  # 5 generated PDF reports
└── frontend/
    └── src/
        ├── App.jsx                # Main shell (sidebar+chat+insights)
        ├── components/
        │   ├── ChartView.jsx      # Recharts bar/pie/table renderer
        │   └── InsightsPanel.jsx  # Right panel with live analytics
        └── hooks/
            └── useApi.js          # Fetch wrappers
```

---

## API Endpoints

| Method | Endpoint             | Description                          |
|--------|----------------------|--------------------------------------|
| GET    | /health              | Health check                         |
| POST   | /api/chat            | Main chat endpoint (AI + tools)      |
| POST   | /api/analytics       | Direct analytics (no AI)             |
| GET    | /api/analytics/metrics | List available metrics             |
| POST   | /api/ingest/reload   | Force reload data from CSVs          |
| GET    | /api/ingest/status   | Show row counts per table            |
| GET    | /docs                | Auto-generated Swagger UI            |

### Chat request example
```json
POST /api/chat
{
  "message": "Which titles performed best in 2025?",
  "history": []
}
```

### Chat response example
```json
{
  "answer": "Based on the database, the top performing titles in 2025 by revenue are...",
  "tool_trace": [
    { "tool": "query_database", "inputs": {"sql": "SELECT ..."}, "result_summary": "10 rows returned" },
    { "tool": "search_documents", "inputs": {"query": "top titles 2025"}, "result_summary": "3 document chunks found" }
  ],
  "chart_data": { "chart_type": "bar", "title": "Top Titles by Revenue", "data": [...] },
  "sources": ["CSV Analytics", "PDF Documents", "SQL Database"]
}
```

---

## Data Sources

### CSV / SQL Tables (loaded into SQLite)
| Table | Rows | Description |
|-------|------|-------------|
| movies | 15 | Title metadata, budget, revenue, ratings |
| viewers | 2,000 | Demographics, city, subscription tier |
| watch_activity | 8,000 | Per-viewer watch events |
| reviews | 3,000 | Sentiment, rating, review text |
| marketing_spend | 90 | Channel spend + CTR per title |
| regional_performance | 450 | City-level engagement + views |

### PDF Documents
- `quarterly_executive_report.pdf` — Q1 2025 business summary
- `campaign_performance_summary.pdf` — Marketing analysis
- `content_roadmap_2025.pdf` — Upcoming projects
- `content_policy_guidelines.pdf` — Internal policies
- `audience_behavior_report.pdf` — Viewer analysis

---

## Security Design

1. **SELECT-only SQL** — The `execute_query()` function validates that every query starts with SELECT and blocks DROP, DELETE, INSERT, UPDATE, ALTER, ATTACH. Parameterized queries prevent injection.

2. **Tool-based access** — The model never touches raw data directly. It calls named tools that enforce their own access rules and cannot bypass these by rewriting a tool.

3. **No PII exposure** — The system prompt explicitly instructs the model not to surface individual viewer data. Tools cap result sets at 100 rows.

4. **API key security** — Keys stored in `.env` (gitignored). The `.env.example` file is committed with placeholder values only.

5. **CORS whitelist** — Only `localhost:5173` and `localhost:3000` are allowed in development.

6. **Input validation** — All request bodies are validated via Pydantic models with length limits.

---

## Assumptions & Tradeoffs

| Decision | Rationale |
|----------|-----------|
| SQLite over PostgreSQL | Zero infrastructure to spin up; fine for demo data at this scale |
| Keyword PDF search vs vector embeddings | No external vector DB needed; good enough for 5 small docs |
| In-process tool calling vs microservices | Simpler deployment; tools are stateless functions, easy to extract later |
| Fake data with deterministic seed | Reproducible results; all example questions work predictably |
| Groq-hosted Llama 3.3 70B | Free-tier friendly setup with tool-calling support |
| React + Recharts | Lightweight, no heavy chart library needed |
| Single SQLite file | Adequate for read-heavy analytics workload; persistent with Docker volume |

---

## Example Questions to Try

1. **"Which titles performed best in 2025?"** → Queries DB + retrieves executive report context
2. **"Why is Stellar Run trending recently?"** → Queries watch_activity + searches campaign PDF
3. **"Compare Dark Orbit vs Last Kingdom"** → Uses title_comparison analytics + DB reviews
4. **"Which city had the strongest engagement last month?"** → Regional performance query
5. **"What explains weak comedy performance?"** → Comedy analysis + searches content roadmap PDF
6. **"What recommendations would you give for leadership?"** → Synthesizes DB data + multiple PDFs

