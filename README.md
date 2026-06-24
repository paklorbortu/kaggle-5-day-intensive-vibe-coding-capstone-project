# Enterprise Data Agent

### A Multi-Agent AI Pipeline That Turns Natural Language Questions Into Executive Reports, Live on Google Cloud Run

**Live Demo:** https://enterprise-data-agent-4au4slh6fa-uc.a.run.app/

> Built for the [Kaggle 5-Day Gen AI Intensive Vibe Coding Capstone](https://www.kaggle.com/competitions/vibecoding-agents-capstone-project) — Track: Agents for Business

---

## The Problem

Small and medium-sized businesses sit on enormous amounts of data in warehouses like BigQuery, but accessing insights from that data typically requires a data analyst, SQL knowledge, and manual effort to produce a report. A sales manager who wants to know "which product categories drove the most profit in Q3?" has to file a request, wait for a query, interpret raw results, and produce a document — a workflow that can take hours or days.

The Enterprise Data Agent collapses that entire workflow into a single natural language question answered in under two minutes, delivered as a formatted executive report in Google Docs with a chart embedded.

---

## What It Does

A user types a business question into the web interface. The system autonomously:

1. Generates and executes SQL against a BigQuery sales dataset
2. Structures the query results into typed, validated data
3. Selects the appropriate chart type and renders it using the AntV MCP Server
4. Writes a complete executive report to Google Docs with the chart embedded
5. Streams live phase progress to the UI and returns the Google Doc link with chart preview

The entire pipeline completes in roughly 60 to 90 seconds.

---

## Architecture

### Multi-Agent Graph Pipeline

The system is built on **Google ADK 2.0** using a custom graph-based orchestration pattern. Rather than a single monolithic agent, the pipeline is decomposed into six specialized nodes that each do one thing:

```
┌─────────────────────────┐
│   data_analyst_agent    │  Queries BigQuery via SQL, produces plain-text summary
└──────────┬──────────────┘
           │
┌──────────▼──────────────┐
│  data_formatter_agent   │  Structures summary → validated DataQueryResult (Pydantic)
└──────────┬──────────────┘
           │
┌──────────▼──────────────────────┐
│  charting_specialist_agent      │  Renders chart via AntV MCP Server
└──────────┬──────────────────────┘
           │
┌──────────▼──────────────────────┐
│  charting_formatter_agent       │  Structures output → ChartingResult (Pydantic)
└──────────┬──────────────────────┘
           │
┌──────────▼──────────────┐
│    reporting_agent      │  Creates Google Doc report with chart embedded
└──────────┬──────────────┘
           │
┌──────────▼──────────────────────┐
│  reporting_formatter_agent      │  Structures output → ReportGenerationResult (Pydantic)
└─────────────────────────────────┘
```

Each node runs as an independent ADK `Runner`. The graph runner fires async progress callbacks at each node transition, which the FastAPI SSE endpoint streams to the frontend in real time.

### Why a Graph — Not a Workflow

The original implementation used ADK's built-in `Workflow` class. Migrating to an explicit node graph gave three key advantages:

- **Reliable SSE streaming**: phase events are fired explicitly by the graph runner rather than inferred from `event.author` mid-stream
- **Testability**: each node is a plain async Python function that can be unit tested independently
- **Graceful failure handling**: empty data or tool failures can be caught and routed per node

### Tool + Output Schema Split Pattern

Combining MCP tools with a structured `output_schema` on the same agent causes Gemini to either skip tool-calling entirely or exceed the schema complexity limit. Each major pipeline step is therefore split into:

- A **tool agent** (calls MCP tools, produces plain-text output)
- A **formatter agent** (no tools, structured `output_schema`, validates output into a Pydantic model)

This prevents hallucinated or malformed data from propagating between stages.

### MCP Tool Integration

Three MCP servers are wired into the pipeline via `stdio` subprocesses:

| Server | Package | Purpose |
|---|---|---|
| BigQuery | `mcp-server-bigquery` via `uvx` | SQL execution against `empyrean-verve-401907.sales.sales_table` |
| Chart Generator | `@antv/mcp-server-chart` via `npx` | Renders bar, line, column, pie, dual-axis charts; returns public image URL |
| Google Docs | Custom `gdocs-mcp` (TypeScript) | Creates, formats, and populates Google Docs via the Docs and Drive APIs |

All three MCP servers are managed by `AsyncExitStack` for clean teardown on shutdown.

### SSE Streaming Architecture

```
Browser  ──POST /sessions/{id}/ask──►  FastAPI
                                           │
                                    asyncio.Queue
                                           │
                                    graph_task (background)
                                           │
                              ┌────────────▼────────────┐
                              │   for node in GRAPH_NODES│
                              │     on_progress(phase)   │──► queue.put(SSE event)
                              │     await node(state)    │
                              └─────────────────────────-┘
                                           │
Browser  ◄──SSE phase/result events───  queue consumer
```

### Pydantic Output Schemas

```python
class DataQueryResult(BaseModel):
    dimensions: list[str]   # e.g. ["Category", "Year"]
    metrics: list[str]      # e.g. ["TotalSales", "Profit"]
    data: list[dict]        # actual query result rows

class ChartingResult(BaseModel):
    chart_image_url: str
    chart_type: str
    dimensions_visualized: list[str]
    metrics_visualized: list[str]

class ReportGenerationResult(BaseModel):
    document_url: str       # canonical https://docs.google.com/document/d/{id}/edit
    document_id: str
    title: str
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Agent Framework | Google ADK 2.0 |
| LLM | Gemini 2.5 Flash via Vertex AI |
| Data Warehouse | Google BigQuery |
| Chart Generation | AntV MCP Server (`@antv/mcp-server-chart`) |
| Document Creation | Custom gdocs-mcp (TypeScript, Google Docs + Drive API) |
| Backend | FastAPI + uvicorn, Python 3.12 |
| Frontend | Vanilla HTML/CSS/JS with SSE streaming |
| Infrastructure | Google Cloud Run, Container Registry, Cloud Build |
| Auth | Google OAuth2 (gdocs-mcp), Application Default Credentials (BigQuery + Vertex AI) |
| Secrets | Google Secret Manager |

---

## Project Structure

```
enterprise-data-agent/
├── app/
│   ├── agents/
│   │   ├── data_analyst.py          # BigQuery agent + DataQueryResult schema
│   │   ├── charting_specialist.py   # Chart agent + ChartingResult schema
│   │   ├── reporting.py             # Google Docs agent + ReportGenerationResult schema
│   │   └── worker.py                # Shared worker utilities
│   ├── api/
│   │   └── server.py                # FastAPI server, SSE streaming, session management
│   ├── config/
│   │   ├── settings.py              # Env vars, MCP server command configs
│   │   └── mcp_config.py            # McpToolset initializer + AsyncExitStack
│   ├── orchestrator/
│   │   ├── graph.py                 # Graph-based 6-node pipeline orchestrator
│   │   └── supervisor.py            # Original sequential pipeline (reference)
│   └── app_utils/
│       ├── telemetry.py             # OpenTelemetry tracing
│       └── typing.py                # Shared type definitions
├── frontend/
│   └── index.html                   # Single-page chat UI with SSE phase tracking
├── gdocs-mcp/                       # Custom Google Docs MCP server (TypeScript)
│   ├── src/
│   │   ├── server.ts                # MCP server entry point
│   │   ├── auth.ts                  # Google OAuth2 with env-var-configurable paths
│   │   └── tools/                   # Docs CRUD, content insertion, formatting tools
│   └── build/                       # Compiled JS output
├── tests/
│   ├── unit/
│   └── integration/
├── Dockerfile                       # Cloud Run container
├── cloudbuild.yaml                  # Cloud Build CI/CD
├── setup-gcp.ps1                    # One-time GCP project setup script
├── update-token.ps1                 # OAuth token refresh helper
└── pyproject.toml
```

---

## Local Setup

### Prerequisites

- Python 3.12+
- Node.js 20+
- `uv` package manager: `pip install uv`
- Google Cloud SDK with a project that has BigQuery, Vertex AI, and Docs API enabled
- A BigQuery dataset with a [`sales_table`](https://www.kaggle.com/datasets/shantanugarg274/sales-dataset/data) table

### 1. Clone and install

```bash
git clone https://github.com/paklorbortu/kaggle-5-day-intensive-vibe-coding-capstone-project.git
cd kaggle-5-day-intensive-vibe-coding-capstone-project
uv sync
```

### 2. Configure environment

Copy `.env.example` to `.env` and fill in your values:

```dotenv
GOOGLE_CLOUD_PROJECT=your-project-id
GOOGLE_CLOUD_LOCATION=us-central1
BIGQUERY_PROJECT_ID=your-project-id
BIGQUERY_DATASET_ID=sales
BIGQUERY_LOCATION=asia-east1
GOOGLE_APPLICATION_CREDENTIALS=/path/to/your/service-account.json
GOOGLE_DOCS_CREDENTIALS_PATH=/path/to/your/oauth-credentials.json
TARGET_DRIVE_FOLDER_ID=your-drive-folder-id
DEFAULT_MODEL=gemini-2.5-flash
```

### 3. Set up Google Docs OAuth

```bash
cd gdocs-mcp
npm install
npm run build
# First run will open a browser for OAuth consent
node build/server.js
cd ..
```

### 4. Run the server

```bash
uv run uvicorn app.api.server:app --host 0.0.0.0 --port 8000 --reload
```

Open http://localhost:8000 in your browser.

---

## Cloud Run Deployment

### One-time GCP setup

Run the setup script to create the service account, grant IAM roles, and upload OAuth secrets to Secret Manager:

```powershell
.\setup-gcp.ps1
```

This script:
- Enables required GCP APIs (Cloud Run, Cloud Build, Secret Manager, BigQuery, Container Registry)
- Creates a dedicated service account with BigQuery, Vertex AI, and Secret Manager roles
- Uploads your `credentials.json` and `token.json` to Secret Manager as `gdocs-oauth-credentials` and `gdocs-oauth-token`

### Build and deploy

```bash
# Build gdocs-mcp locally first (pre-compiled JS is copied into the container)
cd gdocs-mcp && npm run build && cd ..

# Submit to Cloud Build (builds image, pushes to GCR, deploys to Cloud Run)
gcloud builds submit --project your-project-id --config cloudbuild.yaml .
```

### How secrets are handled

- `credentials.json` (OAuth client ID) is baked into the container image — it is not a secret token
- `token.json` (OAuth access/refresh token) is stored in Secret Manager and mounted at `/app/secrets/token.json` at runtime
- `auth.ts` reads the token path from `GDOCS_TOKEN_PATH` env var, enabling the writable copy pattern that works around Cloud Run's read-only secret mounts

### Grant public access

```bash
gcloud beta run services add-iam-policy-binding \
  --region=us-central1 \
  --member=allUsers \
  --role=roles/run.invoker \
  enterprise-data-agent
```

---

## Key Technical Challenges Solved

**1. Tools + output_schema suppression in Gemini**
Combining MCP tools with `output_schema` on the same agent suppresses tool-calling. Solved by splitting each step into a tool agent (plain text) and a formatter agent (structured output, no tools).

**2. ADK 2.3 mode validation**
ADK 2.3 enforces `mode='chat'` on any agent passed directly to `Runner`. `mode='single_turn'` is only valid inside `Workflow` nodes. Resolved by removing the mode declaration.

**3. Read-only secret mounts on Cloud Run**
Cloud Run mounts secrets as read-only files, but OAuth token refresh writes back to `token.json`. Solved by mounting the secret outside the gdocs-mcp directory (`/app/secrets/token.json`) and pointing `auth.ts` there via `GDOCS_TOKEN_PATH`.

**4. Secret mount directory collision**
Cloud Run cannot mount two secrets into the same directory. Each secret must have its own unique parent directory.

**5. SSE streaming with long-running async pipeline**
Used `asyncio.Queue` with a `None` sentinel: the graph runs as a background `asyncio.create_task`, posting SSE events to the queue at each node transition, while the FastAPI generator yields until the sentinel arrives.

**6. Cross-platform MCP subprocess config**
`npx.cmd` is Windows-only. On Linux (Cloud Run), `npx` is used. Detected via `sys.platform == "win32"` in `settings.py`.

---

## Environment Variables Reference

| Variable | Required | Description |
|---|---|---|
| `GOOGLE_CLOUD_PROJECT` | Yes | GCP project ID |
| `GOOGLE_CLOUD_LOCATION` | Yes | Vertex AI region (e.g. `us-central1`) |
| `BIGQUERY_PROJECT_ID` | Yes | BigQuery project ID |
| `BIGQUERY_DATASET_ID` | Yes | BigQuery dataset name |
| `BIGQUERY_LOCATION` | Yes | BigQuery dataset location |
| `DEFAULT_MODEL` | Yes | Gemini model string (e.g. `gemini-2.5-flash`) |
| `TARGET_DRIVE_FOLDER_ID` | Yes | Google Drive folder ID for created docs |
| `GOOGLE_APPLICATION_CREDENTIALS` | Local only | Path to service account JSON (not needed on Cloud Run) |
| `GDOCS_TOKEN_PATH` | Cloud Run | Writable path for OAuth token (set automatically by Dockerfile) |
| `GDOCS_CREDENTIALS_PATH` | Cloud Run | Path to OAuth credentials JSON in container |

---

## Security Notes

- No API keys or secrets are stored in code or committed to the repository
- OAuth credentials and tokens are managed via Google Secret Manager on Cloud Run
- Each MCP server subprocess receives only the environment variables it needs (isolated env copies)
- BigQuery access uses the Cloud Run service account's ADC (Application Default Credentials) — no key file required in production
