## OpsPulse (Kestra) — Wakanda Data Demo

OpsPulse is a **Kestra workflow** that ingests daily metrics from multiple “systems” (CSV files for the demo), computes day-over-day and 7‑day baseline deltas, uses **Kestra’s built-in AI Agent** to summarize what changed into strict JSON, then **automates a decision** (green/yellow/red) and **emails** when attention is needed.

### What’s in this repo

- **`data/`**: demo CSV “systems”
  - `revenue.csv` (Stripe-like)
  - `traffic.csv` (GA-like)
  - `support.csv` (Zendesk-like)
  - `deploys.csv` (GitHub/Vercel-like)
- **`kestra/config.yml`**: local Kestra configuration (H2 + local storage + allowed local file paths)
- **`kestra/flow-opspulse.yaml`**: the OpsPulse flow
- **`docker-compose.yml`**: run Kestra locally

### Prerequisites

- Docker Desktop
- An OpenAI API key (or swap the provider in the flow)
- SMTP credentials (Gmail app password is fine)

### Run Kestra locally

From the repo root:

```bash
docker compose up -d
```

Open the UI at `http://localhost:8090` (see `docker-compose.yml` ports).

### Secrets (Community Edition friendly)

Yes — **Community Edition** supports the KV Store, but **for secrets** the simplest/recommended approach is the built-in `secret()` mechanism via environment variables.

This repo is set up to load an `.env_encoded` file via `docker-compose.yml` and the flow references secrets using `{{ secret('...') }}`.

#### Create `.env` (plain) locally (don’t commit it)

Create a file named `.env` in the repo root with:

- `OPENAI_API_KEY`
- `EMAIL_FROM`, `EMAIL_TO`
- `SMTP_HOST`, `SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD`

#### Generate `.env_encoded` (base64)

In bash/WSL from the repo root:

```bash
while IFS='=' read -r key value; do
  [ -z \"$key\" ] && continue
  echo \"SECRET_${key}=$(printf %s \"$value\" | base64 | tr -d '\\n')\"
done < .env > .env_encoded
```

Then start Kestra:

```bash
docker compose up -d
```

### Import and run the flow

1. In the Kestra UI, import `kestra/flow-opspulse.yaml`.
2. Run the flow manually.
3. (Optional) Set `target_date` to a specific date like `2025-12-13`.

### Expected demo behavior

- The included CSVs contain a final-day anomaly (refund spike + conversion drop + support spike + incidents).
- The workflow should produce a **yellow/red** report and send an email.
- Every run stores the agent output as an execution artifact: `opspulse-report.json`.

### Decision logic (high-level)

- AI agent returns `{"status":"green|yellow|red", ... ,"confidence":0-1}`.
- If the agent returns **green** but `confidence < confidence_threshold`, the flow treats it as **yellow** (to avoid missing uncertain cases).

### Demo script (2–3 minutes)

- Show the 4 CSVs and explain they represent different operational systems.
- Run OpsPulse once and show:
  - agent JSON output
  - stored `opspulse-report.json` artifact
  - email (if yellow/red)
- Edit the last row in one CSV (or swap datasets) and rerun to show the status flips.

### Notes

- The Python script tasks run in Docker (via the Docker socket mount) using `python:3.11-slim`.
- Local CSVs are accessed with `file:///files/...` and the repo `data/` directory is mounted into the Kestra container at `/files`.
