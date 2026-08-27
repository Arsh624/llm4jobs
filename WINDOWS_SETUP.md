# Running llm4jobs on Windows (from scratch)

This is a self-contained, Dockerized job-alert tracker. Everything runs inside
containers, so the only things you install on the machine are Docker + WSL2.

## What it does
Every 15 min it scrapes ~40 company career pages, finds **new** postings, uses
`gpt-4o-mini` to read each one's title + required years, and emails you a digest
of roles needing **≤ 2 years** experience that are **not** senior/manager/principal.
State lives in a local SQLite DB (`state/snapshots.sqlite3`) and is append-only.

---

## One-time setup on a new Windows machine

### 1. Install Docker Desktop + WSL2
Open **PowerShell as Administrator** and run:
```powershell
wsl --install
winget install --id Docker.DockerDesktop --source winget --accept-package-agreements --accept-source-agreements
```
**Reboot.** Then launch **Docker Desktop** once and wait until it says *Engine running*.

### 2. Get the code
```powershell
git clone https://github.com/Arsh624/llm4jobs.git
cd llm4jobs
```

### 3. Create your .env (holds the 2 secrets)
Copy the template and fill in the two `REPLACE_ME` values:
```powershell
Copy-Item .env.example .env
notepad .env
```
- `OPENAI_API_KEY` — from https://platform.openai.com/api-keys
- `SMTP_PASS` — a Gmail **App Password** (https://myaccount.google.com/apppasswords, needs 2FA)

The `.env` file is git-ignored — it stays on your machine and is never committed.

### 4. Build + start (first build takes a few minutes)
```bash
docker compose up -d --build
```
That's it. Three services start and keep running: `batch`, `inference`, `email`.

---

## Keep it running (so it doesn't stop)
- Containers use `restart: unless-stopped` — they auto-restart on crash and when
  Docker starts.
- In **Docker Desktop → Settings → General**, enable **"Start Docker Desktop when
  you log in"** so it survives reboots.
- The tool only runs while this PC is on and Docker Desktop is running.

## Handy commands
```bash
docker compose logs -f          # watch all logs live
docker compose logs -f batch    # just the scraper
docker compose ps               # container status
docker compose down             # stop everything
docker compose up -d            # start again (no rebuild)
docker compose up -d --build    # rebuild after code/CSV changes
```

## Companies
Edit `inputs/companies.csv` (`company,url`). It's mounted live — a `batch`
restart picks up changes with no rebuild:
```bash
docker compose restart batch
```

## Cost
gpt-4o-mini. First run processes the whole current backlog (~$2-4 once), then
~$10-40/month depending on how noisy the pages are. Lower cost by raising
`BATCH_INTERVAL_SECONDS` in `.env` (e.g. 3600 = hourly).
