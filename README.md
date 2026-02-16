# JobPostTracker

JobPostTracker is a reliability-first job intelligence pipeline that monitors company career pages, detects newly added roles, extracts structured job data (including minimum years of experience), and queues relevant roles for curated email digests.

---

## Joint Project

This is a collaborative project built by:

- Archit Shukla — https://github.com/YOUR_USERNAME  
- Varunnagarnahalli Raghavendra — https://github.com/HIS_USERNAME  

Both contributors were involved in the system design, scraping pipeline, extraction logic, and infrastructure decisions.

---

## Overview

The system continuously tracks company career pages and processes new job postings through a structured extraction pipeline.

### Core Components

### Batch Runner
- Fetches company career pages
- Extracts links using Node + Puppeteer
- Diffs against previous snapshots
- Enqueues newly discovered job URLs

### Inference Worker
- Consumes queued job tasks
- Uses Puppeteer to render dynamic job pages
- Passes structured content to a Python extraction pipeline
- Extracts:
  - Job title
  - Minimum years of experience (LLM-based extraction)

### Storage Layer
A local SQLite database (`state/snapshots.sqlite3`) stores:
- Page snapshots
- Diff queue
- Job tasks
- Extracted job details
- Email state tracking

---

## Architecture

Node.js handles dynamic rendering and scraping.  
Python handles structured inference and LLM-based extraction.  
SQLite manages state and diff tracking.

### Pipeline Flow

Company Pages  
→ Node Link Extractor (Puppeteer)  
→ Snapshot Diff Engine  
→ Queue New Jobs  
→ Puppeteer Job Scraper  
→ Python Extraction (LLM)  
→ SQLite Storage  
→ Email Digest  

---

## Project Structure

```
node_link_extractor/
    getAllLinks.js

job-alert/
    puppeteer_scraper/
        puppeteer_scrapper.js
    extract_experience.py

tracker/
    batch_runner.py
    inference_worker.py
    seed_current_snapshot_from_csv_threaded.py
    database layer + utilities

scripts/
    cron helpers + utilities

state/
    snapshots.sqlite3
    logs

inputs/
    companies.csv
```

---

## Stability & Safety Improvements

To prevent runaway Chromium processes and disk accumulation:

- Reuse in-process Chromium instances where possible
- Managed temporary `userDataDir` cleanup on exit
- Enforced `page.close()` in finally blocks
- Signal and exception handlers for graceful shutdown
- Per-process page semaphore (`PPT_MAX_PAGES`)
- Per-scrape watchdog timeout (`PPT_SCRAPE_WATCHDOG_MS`)
- Optional global Chromium slot cap (`PPT_GLOBAL_CHROME_SLOTS`)

These changes reduce:
- Orphaned Chrome helper processes
- Disk bloat from browser profiles
- Concurrency-related instability

---

## Environment Variables

| Variable | Default | Purpose |
|-----------|----------|----------|
| PPT_MAX_PAGES | 4 | Max pages per Node process |
| PPT_SCRAPE_WATCHDOG_MS | 120000 | Per-scrape timeout |
| PPT_MAX_CHROME_HELPERS | 15 | macOS helper guard |
| PPT_GLOBAL_CHROME_SLOTS | 0 | Cross-process Chromium cap |
| PPT_GLOBAL_SLOT_TIMEOUT_MS | 30000 | Slot wait timeout |
| PPT_GLOBAL_CHROME_SLOTS_DIR | temp dir | Slot state storage |

For experience extraction:

```
export OPENAI_API_KEY="your_key_here"
```

---

## Running Locally

### Install Dependencies

Node (v24+ recommended):

```
cd node_link_extractor
npm install
```

```
cd job-alert/puppeteer_scraper
npm install
```

Python:

```
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

---

### Seed Snapshots

```
export PPT_GLOBAL_CHROME_SLOTS=4
export PPT_MAX_PAGES=2

python -m tracker.seed_current_snapshot_from_csv_threaded \
  --csv ./inputs/companies.csv \
  --db ./state/snapshots.sqlite3 \
  --node-workdir ./node_link_extractor \
  --node-bin $(which node) \
  --node-timeout-seconds 180 \
  --max-workers 8 \
  --clear-current-snapshot-first \
  -v
```

---

### Run Batch

```
python -m tracker.batch_runner \
  --csv inputs/companies.csv \
  --node-workdir ./node_link_extractor \
  --max-workers 1
```

---

### Start Inference Worker

```
python -m tracker.inference_worker \
  --puppeteer-script ./job-alert/puppeteer_scraper/puppeteer_scrapper.js \
  --extract-experience-py ./job-alert/extract_experience.py \
  --verbose
```

---

## Database Utilities

Backup:

```
cp state/snapshots.sqlite3 state/snapshots.sqlite3.bak.$(date +%Y%m%dT%H%M%S)
```

Export:

```
sqlite3 -header -csv state/snapshots.sqlite3 \
"SELECT * FROM job_details;" > job_details.csv
```

Clear job details:

```
sqlite3 state/snapshots.sqlite3 <<'SQL'
BEGIN;
DELETE FROM job_details;
COMMIT;
VACUUM;
SQL
```

---

## Deployment Notes

Recommended for Puppeteer-heavy workloads:

- EC2 / ECS for simplicity and stability
- Moderate throughput: m6i.xlarge
- Budget testing: t3.large or m5.large

Lambda is possible but requires container images, EFS persistence, and higher memory allocation.

---

## Future Improvements

- Persistent Node scraping server mode (reduce process spawn overhead)
- Integration test verifying userDataDir cleanup
- Monitoring for Chromium process growth
- Secret detection hooks (detect-secrets, git-secrets)

---

## Security

- API keys stored via environment variables
- If a secret is committed:
  - Revoke immediately
  - Clean history using git-filter-repo
  - Force push cleaned mirror
