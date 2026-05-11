# Agentic AI Security Scanner

> **Automated security testing with AI-assisted remediation.** Scan web targets, analyze findings, and generate developer-focused JSON and DOCX reports.

## Table of Contents

- [Overview](#overview)
- [What Agentic AI Means Here](#what-agentic-ai-means-here)
- [Features](#features)
- [Architecture](#architecture)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Prerequisites](#prerequisites)
- [Running the Application](#running-the-application)
- [API Reference](#api-reference)
- [Agent Details](#agent-details)
- [Database Schema](#database-schema)
- [Development](#development)
- [License](#license)

## Overview

Agentic AI Security Scanner is a web application security testing platform built around **OWASP ZAP**, a **FastAPI backend**, and a **React frontend**. It collects scan data, normalizes and enriches findings, runs structured AI analysis, and produces professional reports with actionable remediation guidance.

By default, the application uses a deterministic backend pipeline. A LangChain-based orchestration mode is available as an opt-in feature.

## What Agentic AI Means Here

In this project, AI is used in a controlled, bounded way:

- Vulnerabilities are normalized into structured objects before AI sees them.
- AI is asked to return JSON-only output with a fixed schema.
- Responses are validated before being merged into findings.
- Similar findings are grouped so AI analyzes representative items instead of every duplicate.
- If AI fails, the pipeline falls back to deterministic static remediation content.

For workflow diagrams and ideas, see [AGENTIC_AI_WORKFLOW.md](AGENTIC_AI_WORKFLOW.md).

## Features

- OWASP ZAP integration for spider, AJAX spider, and active scan workflows
- Fallback HTTP-based scanning path when ZAP is unavailable
- Structured normalization and deduplication of raw alerts
- OWASP Top 10 mapping from CWE-based findings
- AI-assisted vulnerability analysis using Gemini with Groq fallback
- DOCX report generation plus structured JSON report payloads
- Server-Sent Events streaming for pipeline progress, logs, findings, AI analysis, and reports
- Persistent scan history with PostgreSQL
- URL validation, SSRF guard rails, and rate limiting

## Architecture

### High-Level Pipeline

```mermaid
flowchart LR

A["1. Input Validation and Rate Limiting"]
--> B["2. Scanning<br/>ZAP Spider / AJAX / Active Scan<br/>Fallback: Basic HTTP Scan"]

B --> C["3. MCP Structuring<br/>Normalize and Deduplicate Alerts<br/>Create analysis_key"]

C --> D["4. OWASP Mapping<br/>CWE to OWASP Top 10"]

D --> E["5. AI Analysis<br/>Gemini / Groq<br/>Strict JSON Schema<br/>Representative Groups and Expansion"]

E --> F["6. Merge AI Into Findings<br/>Developer-Ready Fixes and Verification"]

F --> G["7. Report Generation<br/>DOCX and JSON Summary"]

G --> H["8. Persistence<br/>Save Scan and Vulnerabilities"]
```
### End-to-End Data Flow

```mermaid
sequenceDiagram
  participant UI as React UI
  participant API as FastAPI Backend
  participant ZAP as OWASP ZAP
  participant AI as Gemini/Groq
  participant DOCX as Report Agent

  UI->>API: POST /api/scan {url, scanType}
  API-->>UI: SSE /api/scan/{scanId}/stream
  API->>ZAP: Spider/AJAX/Active scan
  ZAP-->>API: Raw alerts
  API->>API: Normalize + map + score
  API->>AI: Structured remediation prompt
  AI-->>API: JSON analysis per analysis_key
  API->>DOCX: Build report
  DOCX-->>API: docx_b64 + report metadata
  API-->>UI: SSE findings, analysis, report, done
```

### System Context

```mermaid
flowchart TB
    classDef person fill:#08427B,stroke:#073B6F,color:#fff
    classDef system fill:#1168BD,stroke:#0B4884,color:#fff
    classDef external fill:#666666,stroke:#0B4884,color:#fff

    user["Security Engineer / Developer"]
    scanner["Agentic AI Scanner"]
    target["Target Web Application"]
    zap["OWASP ZAP"]
    gemini["Google Gemini / Groq"]
    postgres["PostgreSQL"]

    user -->|Uses web UI| scanner
    scanner -->|Scans| target
    scanner -->|API calls| zap
    scanner -->|AI analysis| gemini
    scanner -->|Stores data| postgres

    class user person
    class scanner system
    class target,zap,gemini,postgres external
```

### Container View

```mermaid
graph TB
    subgraph Frontend
        UI[React UI]
    end

    subgraph Backend
        API[FastAPI]
        Routes[Scan + Health Routes]
    end

    subgraph Agents
        Scanner[Scanner Agent]
        MCP[MCP Agent]
        Mapper[OWASP Mapper]
        Analysis[Analysis Agent]
        Report[Report Agent]
    end

    subgraph Data
        DB[(PostgreSQL)]
        Reports[Reports Directory]
    end

    subgraph External
        ZAP[OWASP ZAP]
        Gemini[Gemini]
        Groq[Groq]
    end

    UI --> API
    API --> Routes
    Routes --> Scanner
    Routes --> MCP
    Routes --> Mapper
    Routes --> Analysis
    Routes --> Report
    Scanner --> ZAP
    Analysis --> Gemini
    Analysis -.-> Groq
    API --> DB
    Report --> Reports
```

## Quick Start

### System Requirements

- Python 3.11+
- Node.js 18+ and npm
- PostgreSQL 13+
- OWASP ZAP running in daemon mode

### 1. Start OWASP ZAP

```bash
zap.bat -daemon -port 8080 -config api.key=YOUR_KEY
```

Download OWASP ZAP from [zaproxy.org](https://www.zaproxy.org/download/).

### 2. Clone and Configure

```bash
git clone <repository-url>
cd agentic-ai-scanner
```

Create `backend/.env` from `backend/.env.example`, then fill in your API keys and database settings.

### 3. Install Root Dependencies

```bash
npm install
```

### 4. Start the Application

```bash
npm start
```

This starts:

- Backend at `http://localhost:8000`
- Frontend at `http://localhost:3000`

## Configuration

Create `backend/.env` with values like:

```bash
# Application
APP_NAME=Agentic AI Security Scanner
VERSION=3.0

# Database
DATABASE_URL=postgresql+asyncpg://postgres:password@localhost:5432/agentic_scanner

# ZAP
ZAP_BASE_URL=http://127.0.0.1:8080
ZAP_API_KEY=your_zap_api_key

# AI
ENABLE_AI=true
GEMINI_API_KEY=your_gemini_api_key
GEMINI_MODEL=gemini-2.0-flash
GROQ_API_KEY=your_groq_api_key
GROQ_MODEL=llama-3.3-70b-versatile

# Optional LangChain orchestration
ENABLE_LANGCHAIN_AGENT=false

# Scan limits
MAX_SPIDER_MINUTES=1
MAX_ACTIVE_SCAN_MINUTES=2
MAX_URLS=50

# Reports
REPORTS_DIR=./backend/reports
```

### AI Provider Notes

#### Gemini

Gemini is the primary AI provider for vulnerability analysis.

| Variable | Default | Description |
|---------|---------|-------------|
| `GEMINI_API_KEY` | none | Google AI API key |
| `GEMINI_MODEL` | `gemini-2.0-flash` | Gemini model used for analysis |

#### Groq

Groq is used as the fallback provider when Gemini is unavailable.

| Variable | Default | Description |
|---------|---------|-------------|
| `GROQ_API_KEY` | none | Groq API key |
| `GROQ_MODEL` | `llama-3.3-70b-versatile` | Fallback model |

#### LangChain Agent Mode

Set `ENABLE_LANGCHAIN_AGENT=true` to use the optional LangChain orchestration path. By default it stays off so the manual pipeline remains the stable default.

#### Disable AI

```bash
ENABLE_AI=false
```

When disabled, the application uses static fallback remediation content instead of AI output.

## Prerequisites

### OWASP ZAP Setup

1. Install OWASP ZAP.
2. Start it in daemon mode.
3. Set `ZAP_BASE_URL` and `ZAP_API_KEY` in `backend/.env`.

Example:

```bash
zap.sh -daemon -port 8080 -config api.key=change-me-123
```

On Windows, use `zap.bat` instead of `zap.sh`.

### PostgreSQL Setup

Create a database named `agentic_scanner`, then point `DATABASE_URL` at it.

```sql
CREATE DATABASE agentic_scanner;
```

## Running the Application

### Development Mode

From the project root:

```bash
npm start
```

### Backend Only

```bash
cd backend
uvicorn main:app --reload --port 8000
```

Or:

```bash
npm run backend
```

### Frontend Only

```bash
cd frontend
npm start
```

Or:

```bash
npm run frontend
```

### Production Build

```bash
cd frontend
npm run build
```

The production build is written to `frontend/build/`.

## API Reference

### REST Endpoints

| Endpoint | Method | Description |
|---------|--------|-------------|
| `/api/scan` | POST | Launch a new security scan |
| `/api/scan/{scan_id}/stream` | GET | Stream real-time scan events via SSE |
| `/api/scan/{scan_id}/report` | GET | Get the generated report for a scan |
| `/api/scan/{scan_id}/vulnerabilities` | GET | Get vulnerabilities for a scan |
| `/api/scan/{scan_id}` | GET | Get scan details and event history |
| `/api/history` | GET | Get scan history |
| `/api/health` | GET | Health check with DB status |
| `/api/scans` | GET | Alternative scan history endpoint |
| `/api/scans/{scan_id}` | GET | Detailed scan payload with vulnerabilities and report |

### Scan Request Body

```json
{
  "url": "https://example.com",
  "scanType": "quick"
}
```

### Scan Types

- `quick`: faster scan path
- `full`: more comprehensive scan path

### SSE Event Types

| Event Type | Description |
|-----------|-------------|
| `pipeline` | Step status updates |
| `log` | INFO, WARN, and ERROR log lines |
| `vulnerabilities` | Structured finding list |
| `ai_analysis` | AI analysis and remediation content |
| `report` | Generated report metadata |
| `done` | Final score and verdict |
| `done_ack` | Client-close acknowledgment |

## Agent Details

### Validation and Guard Rails

The scan route validates incoming URLs, blocks unsafe internal targets, and enforces rate limits before scanning begins.

### Scanner Agent

- Connects to OWASP ZAP
- Runs spider and active scan workflows
- Supports fallback behavior when needed

### MCP Agent

- Normalizes raw alerts
- Deduplicates findings
- Produces stable structured vulnerability objects

### OWASP Mapper

- Maps CWE-based findings into OWASP Top 10 categories
- Supports reporting and prioritization

### Analysis Agent

- Uses Gemini first and Groq as fallback
- Produces structured remediation guidance
- Validates and normalizes AI output before merge

### Report Agent

- Builds DOCX output and JSON report payloads
- Produces score, verdict, summaries, and report artifacts

### Database Layer

- Persists scans and vulnerabilities
- Stores report metadata and file paths
- Performs lightweight schema compatibility updates on startup

## Database Schema

### `scan_records`

| Column | Type | Description |
|--------|------|-------------|
| `id` | String | Scan UUID |
| `url` | String | Target URL |
| `scan_type` | String | `quick` or `full` |
| `status` | String | Scan status |
| `verdict` | String | Final verdict |
| `score` | Integer | Numeric score |
| `total_vulns` | Integer | Total findings |
| `high_count` | Integer | High severity count |
| `medium_count` | Integer | Medium severity count |
| `low_count` | Integer | Low severity count |
| `info_count` | Integer | Informational count |
| `report_docx_b64` | Text | Base64-encoded DOCX |
| `report_summary` | Text | Report summary |
| `report_generated_at` | String | Report timestamp |
| `report_file_path` | Text | Saved report path |
| `created_at` | DateTime | Created timestamp |
| `updated_at` | DateTime | Updated timestamp |

### `vulnerabilities`

| Column | Type | Description |
|--------|------|-------------|
| `id` | String | Composite key `{scan_id}::{analysis_key}` |
| `scan_id` | String | Parent scan ID |
| `name` | String | Finding name |
| `severity` | String | High, Medium, Low, or Info |
| `url` | String | Affected URL |
| `method` | String | HTTP method |
| `parameter` | String | Affected parameter |
| `description` | Text | Finding description |
| `solution` | Text | Remediation guidance |
| `evidence` | Text | Evidence from scanning |
| `attack` | Text | Attack payload details |
| `reference` | Text | Reference links or text |
| `other_info` | Text | Additional metadata |
| `cwe_id` | String | CWE identifier |
| `wasc_id` | String | WASC identifier |
| `plugin_id` | String | Scanner plugin identifier |
| `confidence` | String | Confidence level |
| `owasp_id` | String | OWASP Top 10 mapping |
| `request_header` | Text | Captured request headers |
| `request_body` | Text | Captured request body |
| `response_header` | Text | Captured response headers |
| `response_body` | Text | Captured response body |

## Development

### Project Structure

```text
agentic-ai-scanner/
|-- backend/
|   |-- agents/
|   |   |-- analysis_agent.py
|   |   |-- langchain_orchestrator.py
|   |   |-- langchain_tools.py
|   |   |-- mcp_agent.py
|   |   |-- owasp_mapper.py
|   |   |-- report_agent.py
|   |   `-- scanner_agent.py
|   |-- app/
|   |   |-- routes/
|   |   `-- main.py
|   |-- constants/
|   |-- middleware/
|   |-- utils/
|   |-- config.py
|   |-- db.py
|   |-- main.py
|   `-- requirements.txt
|-- frontend/
|   |-- src/
|   |   |-- api/
|   |   |-- components/
|   |   |-- guards/
|   |   |-- hooks/
|   |   |-- pages/
|   |   |-- routes/
|   |   |-- App.js
|   |   `-- index.js
|   `-- package.json
|-- AGENTIC_AI_WORKFLOW.md
|-- package.json
`-- README.md
```

### Backend Dependencies

| Package | Purpose |
|---------|---------|
| `fastapi` | Web framework |
| `uvicorn` | ASGI server |
| `sqlalchemy` | ORM and async DB access |
| `asyncpg` | PostgreSQL driver |
| `python-dotenv` | Environment loading |
| `requests` | HTTP client |
| `httpx` | Async HTTP client |
| `python-owasp-zap-v2.4` | ZAP API client |
| `python-docx` | DOCX generation |
| `google-genai` | Gemini SDK |
| `groq` | Groq SDK |
| `langchain` | Optional agent framework |
| `langchain-google-genai` | LangChain Gemini integration |
| `langchain-core` | LangChain core primitives |

### Frontend Dependencies

- React 18
- React Router
- Material UI
- Create React App toolchain

## License

This project is licensed under the MIT License.

> **Security Notice:** Use this tool only for authorized testing. Always obtain explicit permission before scanning a target.
