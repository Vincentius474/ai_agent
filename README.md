# Community Voice / AI Agent Project

This repository contains **Community Voice**, a citizen service-delivery reporting system with AI triage, routing, privacy filtering, human review, and conversational follow-up.

## Project location

The complete application is in [`community-voice-app/`](./community-voice-app).

See the detailed setup and usage guide in the [Community Voice README](./community-voice-app/README.md).

## Quick start

```bash
cd community-voice-app
uv sync --locked
uv run python -m community_voice.auth.seed

# Terminal 1
uv run uvicorn community_voice.api.main:app --host 127.0.0.1 --port 8800

# Terminal 2
uv run streamlit run app/streamlit_app.py

# Terminal 3
cd frontend
npm ci
npm run dev
```

Before starting, copy and configure `.env.example`, and configure `frontend/.env.local`. Run Ollama locally for the privacy filter.
