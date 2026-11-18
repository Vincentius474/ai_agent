# Community Voice

Community Voice is a multilingual service-delivery reporting platform that combines a FastAPI AI backend, a Streamlit teaching interface, and a Next.js citizen/admin interface.

Citizens can submit a report as text, a voice note, or a photo. The system:

1. Redacts personal information locally.
2. Translates and triages the report.
3. Routes it to a municipality and department.
4. Checks relevant web context and possible duplicates.
5. Pauses for human review.
6. Drafts a policy-grounded response.
7. Opens a conversation if an approved draft is sent.

Supported report categories are electricity, roads, sanitation, waste, and water. The UI also includes session-based conversations, dashboards, report history, review queues, and human approval controls.

> **Demo scope:** all sample data is fictional. This project is intended for learning and demonstration, not as a production municipal system. AI output must be reviewed by a human.

## Table of contents

- [Architecture](#architecture)
- [Features](#features)
- [Technology stack](#technology-stack)
- [Repository layout](#repository-layout)
- [Prerequisites](#prerequisites)
- [Quick start](#quick-start)
- [Configuration](#configuration)
- [Running the applications](#running-the-applications)
- [Demo accounts](#demo-accounts)
- [Privacy and voice processing](#privacy-and-voice-processing)
- [Workflow](#workflow)
- [API overview](#api-overview)
- [Eval and quality workflows](#eval-and-quality-workflows)
- [Docker](#docker)
- [Resetting local data](#resetting-local-data)
- [Troubleshooting](#troubleshooting)

## Architecture

```text
Next.js frontend  ─────┐
                       ├── HTTP / SSE ──> FastAPI API
Streamlit app     ─────┘                  │
                                          ├── LangGraph workflow
                                          ├── Gemini / Groq / Tavily
                                          ├── Local Ollama privacy filter
                                          ├── SQLite repositories
                                          ├── Chroma RAG indexes
                                          └── MCP directory tools
```

The FastAPI service is the common backend for both interfaces. It owns authentication, report submission, workflow streaming, reviews, conversations, media, and persisted application state.

## Features

### Citizens

- Sign in and submit reports.
- Enter text or upload an optional photo.
- Record or upload an audio note for transcription.
- Review the transcript before submission.
- Track only their own reports and replies.
- Read and continue conversations.
- Mark a report as fixed.

### Administrators

- Review the report queue by urgency, category, status, or age.
- Inspect triage, routing, web context, duplicate, review, and draft details.
- Approve or reject workflow stages.
- Retry stopped workflow steps.
- Approve, edit, or reject drafted replies.
- View all reports and operational statistics.
- Continue conversations and use a suggested AI chat reply.
- Close conversations.

### AI workflow

- Structured report triage.
- Language handling and translation.
- Department and municipality routing.
- Web context lookup with Tavily.
- Semantic duplicate detection.
- Policy search through ChromaDB retrieval.
- Draft response generation.
- Human-review checkpoints and resumable workflow state.

## Technology stack

| Area | Technology |
|---|---|
| Backend | Python 3.12, FastAPI, Pydantic |
| Agent framework | LangChain, LangGraph |
| Cloud AI | Google Gemini, Groq, Tavily |
| Local AI | Ollama |
| Retrieval | ChromaDB |
| Persistence | SQLite |
| Human UI | Streamlit |
| Web UI | Next.js 16, React 19, TypeScript, Tailwind CSS |
| UI primitives | Base UI, shadcn-style components |
| Package management | uv, npm |
| Evaluation | pytest-compatible eval scripts, LangSmith optional |

## Repository layout

```text
community-voice-app/
├── src/community_voice/
│   ├── agents/        Triage, routing, context, drafter, and chat agents
│   ├── api/           FastAPI routes, authentication dependencies, streaming
│   ├── auth/          Password, JWT, and account seed logic
│   ├── core/          Environment and tracing configuration
│   ├── db/            SQLite repositories
│   ├── graph/         LangGraph workflow, memory, and persistence
│   ├── llm/           Provider/model configuration
│   ├── media/         Audio transcription and photo handling
│   ├── privacy/       Local PII redaction
│   ├── rag/           Chroma policy and report retrieval
│   ├── schemas/       Pydantic domain models
│   └── tools/         Directory, policy, web, and duplicate tools
├── app/               Streamlit application
├── frontend/          Next.js web application
├── data/
│   ├── knowledge_base/ Municipal directory, places, and policy documents
│   ├── sample_reports/ Demo reports, audio, and photos
│   └── evals/         Labelled and privacy evaluation data
├── evals/             Evaluation runners and scoring
├── notebooks/         Course and concept notebooks
├── Dockerfile.backend Backend container definition
├── pyproject.toml     Python dependencies and tool configuration
├── uv.lock            Locked Python dependency versions
└── .env.example       Backend environment template
```

Runtime files such as SQLite databases, uploaded photos, and Chroma indexes are created under `data/` and are intentionally ignored by Git.

## Prerequisites

- Python 3.12
- uv
- Node.js 24 and npm
- Gemini API key
- Groq API key
- Tavily API key
- Ollama for the local privacy filter

Recommended Ollama model:

```bash
ollama pull llama3.2:3b
```

LangSmith is optional.

## Quick start

Run commands from `community-voice-app/`.

### 1. Install Python dependencies

Windows:

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/0.13.7/install.ps1 | iex"
```

macOS:

```bash
curl -LsSf https://astral.sh/uv/0.13.7/install.sh | sh
```

Then install and verify the project environment:

```bash
uv python install 3.12
uv sync --locked
uv run python --version
```

Use `uv sync --locked` to ensure the dependency versions match `uv.lock`.

### 2. Configure the backend

```bash
# macOS/Linux
cp .env.example .env

# Windows PowerShell
Copy-Item .env.example .env
```

Fill in the required API keys described in [Configuration](#configuration).

### 3. Create demo accounts

```bash
uv run python -m community_voice.auth.seed
```

### 4. Configure the frontend

```bash
cd frontend
npm ci
cp .env.example .env.local
cd ..
```

PowerShell users can use `Copy-Item .env.example .env.local` for the second copy command.

### 5. Start the services

Start each service in a separate terminal.

**Terminal 1: FastAPI**

```bash
uv run uvicorn community_voice.api.main:app --host 127.0.0.1 --port 8800
```

API docs: <http://127.0.0.1:8800/docs>

**Terminal 2: Streamlit**

```bash
uv run streamlit run app/streamlit_app.py
```

Streamlit: <http://localhost:8501>

**Terminal 3: Next.js**

```bash
cd frontend
npm run dev
```

Next.js: <http://localhost:3000>

## Configuration

Backend variables are read from `.env`.

| Variable | Required | Purpose |
|---|---:|---|
| `GEMINI_API_KEY` | Yes | Gemini triage, drafting, chat, and transcription |
| `GROQ_API_KEY` | Yes | Routing and context-check models |
| `TAVILY_API_KEY` | Yes | Web context search |
| `JWT_SECRET` | Yes | Signs login tokens |
| `GEMINI_MODEL` | No | Override the Gemini model |
| `GROQ_MODEL` | No | Override the Groq model |
| `OLLAMA_MODEL` | No | Privacy model; default `llama3.2:3b` |
| `OLLAMA_BASE_URL` | No | Default `http://localhost:11434` |
| `PRIVACY_FILTER` | No | Set `false` to disable the Ollama privacy model |
| `LANGSMITH_API_KEY` | No | Enables LangSmith tracing |
| `LANGSMITH_TRACING` | No | Set `true` to trace LangSmith runs |
| `LANGSMITH_PROJECT` | No | LangSmith project name |

Generate a JWT secret:

```bash
openssl rand -hex 32
```

The Next.js frontend uses `frontend/.env.local`:

```dotenv
NEXT_PUBLIC_API_URL=http://localhost:8800
```

## Running the applications

FastAPI must start before submitting reports or using either UI. API startup builds the LangGraph workflow and opens the local persistence/checkpoint stores.

Ollama must be running for full local redaction:

```bash
ollama serve
```

Check it in a browser or terminal:

```text
http://localhost:11434
```

The first AI or embedding operation can be slower because dependencies and indexes may initialize on the first run.

## Demo accounts

These accounts are for local demonstration only.

| Role | Email | Password |
|---|---|---|
| Administrator | `admin@communityvoice.test` | `AdminPass123` |
| Citizen | `citizen1@communityvoice.test` | `Citizen1Pass123` |
| Citizen | `citizen2@communityvoice.test` | `Citizen2Pass123` |

Citizens can submit reports and access only their own report and conversation data. Administrators can use review, dashboard, history, and conversation tools.

## Privacy and voice processing

Text reports and citizen chat messages are processed by a local Ollama privacy filter before cloud AI providers receive them.

The filter targets names, street addresses, phone numbers, identity numbers, and email addresses. Place names in `data/knowledge_base/places.json` can be retained so the routing workflow has useful location context.

Important behavior:

- If Ollama is unavailable and `allow_unfiltered` is accepted, only numbers and emails receive fallback redaction.
- Set `PRIVACY_FILTER=false` to skip the local model. Rule-based number and email redaction still applies.
- Voice notes are transcribed by Gemini before text privacy filtering; a transcript should always be reviewed by the citizen.
- Never send real personal data to cloud providers or tracing services. Use fictional demo data.
- Privacy filtering is assistance, not a guarantee of anonymity or regulatory compliance.

Run privacy evaluations:

```bash
uv run python -m evals.run_privacy_evals
```

## Workflow

```text
Privacy filter
  -> Triage
  -> Municipality/department routing
  -> Web context check
  -> Duplicate check
  -> Human triage review
  -> Draft response
  -> Human draft review
  -> Ready
  -> Conversation
```

### Human review

Workflow checkpoints persist the graph state. An administrator can approve a stage or stop and correct a workflow. If a provider or network call fails, the report may remain as `Stopped`; the stopped step can be retried after restarting or correcting the configuration.

### Conversations

After an approved reply is sent:

- Citizens can read the conversation and send follow-up messages.
- Citizens can mark the report as fixed.
- Administrators receive attention indicators, can read messages, and can generate an editable suggested reply.
- Only an administrator can close a conversation.
- Closed conversations remain readable but cannot accept new messages.

## API overview

FastAPI Swagger documentation is available at <http://127.0.0.1:8800/docs>.

Representative endpoints:

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/health` | API health check |
| `POST` | `/auth/login` | Create a login token |
| `POST` | `/transcriptions` | Transcribe an uploaded voice note |
| `POST` | `/reports` | Submit a report as an SSE workflow stream |
| `GET` | `/reports` | List all reports (admin) |
| `GET` | `/my/reports` | List the current citizen's reports |
| `GET` | `/reviews` | List reports waiting for review |
| `GET` | `/stats` | Report statistics (admin) |
| `GET` | `/reports/{report_id}` | Read report detail |
| `GET` | `/threads` | List all conversations (admin) |
| `GET` | `/my/threads` | List the current citizen's conversations |
| `GET` | `/threads/{report_id}` | Read a conversation |
| `POST` | `/threads/{report_id}/messages` | Send a conversation message |
| `POST` | `/threads/{report_id}/fixed` | Mark a report as fixed (citizen) |
| `POST` | `/threads/{report_id}/suggest` | Generate an admin reply suggestion |
| `POST` | `/threads/{report_id}/close` | Close a conversation (admin) |

Authentication uses JWT bearer tokens. Next.js stores the token in the browser and sends it through the `Authorization` header.

## Eval and quality workflows

Run the offline/unit-style checks:

```bash
uv run pytest
```

No test directory is currently included in this repository snapshot, so pytest will report if no tests are discoverable.

Run live provider tests only when API keys are configured:

```bash
uv run pytest -m live -s
```

Run triage/routing evaluations:

```bash
uv run python -m evals.run_evals --refresh
```

Run privacy evaluations:

```bash
uv run python -m evals.run_privacy_evals
```

Lint and format:

```bash
uv run ruff check
uv run ruff format
```

Frontend lint:

```bash
cd frontend
npm run lint
npm run build
```

Notebooks are stored in `notebooks/`. Select the project `.venv` kernel and clear outputs before committing notebook changes.

## Docker

Build the backend image:

```bash
docker build -f Dockerfile.backend -t community-voice-api .
```

Run it with your environment file:

```bash
docker run --rm -p 8800:8800 --env-file .env -v "${PWD}/data:/app/data" community-voice-api
```

This command is written for PowerShell. The bind mount is optional if you want container-local data, but SQLite and uploaded files will not persist after the container is removed unless `data/` is mounted.

The Dockerfile starts FastAPI on `0.0.0.0:8800`. It does not start Streamlit, Next.js, or Ollama; run those separately or add an orchestration file for a multi-container deployment.

## Resetting local data

Stop all running services before deleting state.

Delete the following to reset reports and workflow state:

- `data/app.db`
- `data/checkpoints.db`
- `data/memory.db`

Then recreate demo accounts:

```bash
uv run python -m community_voice.auth.seed
```

Other generated state may include `data/chroma/` and `data/uploads/`. Use targeted deletion only if a fresh retrieval index or uploaded media reset is also needed.

## Troubleshooting

### Ollama is unavailable

- Confirm `ollama serve` is running.
- Confirm `OLLAMA_BASE_URL` is correct.
- Run `ollama pull llama3.2:3b`.
- Restart FastAPI after changing `.env`.

### API is not reachable

- Confirm FastAPI is running on port 8800.
- Open <http://127.0.0.1:8800/health>.
- Confirm `NEXT_PUBLIC_API_URL=http://localhost:8800` in `frontend/.env.local`.

### Dependency lock error

Restore matching `pyproject.toml` and `uv.lock` files. Install with `uv sync --locked`, and do not update locked dependencies while following a pinned course/demo setup.

### Frontend dependency error

Use:

```bash
cd frontend
npm ci
```

Do not substitute `npm install` for `npm ci` when lockfile reproducibility matters.

### AI provider errors

Free-tier providers can return 503, timeout, quota, or rate-limit errors. Check credentials, wait briefly, and retry. Report workflows can remain stopped and be retried from the admin review interface.
