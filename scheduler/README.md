# Scheduler Service

A standalone Python process (not FastAPI) that automates the full job discovery pipeline on a cron schedule using **APScheduler 3.x**.

---

## Architecture

```
APScheduler (in-process, background scheduler)
  │
  ├── Every 8h  : run_scrape_cycle()
  │     POST /api/scrape        ← platform scrapers (Naukri/LI/Internshala)
  │     scrapy crawl remotive   ← via subprocess
  │     scrapy crawl yc_jobs    ← via subprocess
  │     Wait 60s → count delta jobs
  │
  ├── Every 24h (+4h offset): run_discover_cycle()
  │     For each new job → POST /api/discover
  │     30s delay between calls
  │
  └── Every 24h (+8h offset): run_draft_cycle()
        For each new job with contacts → POST /api/generate
        Pre-generates cold_outreach drafts in Postgres

After each cycle → /data/run_summary.json
Gateway reads it via GET /api/summary
```

---

## Project layout

```
scheduler/
├── main.py       # APScheduler setup, signal handlers, manual run
├── tasks.py      # One function per scheduled task
├── logger.py     # Structured JSON logging to stdout
├── requirements.txt
├── Dockerfile
└── README.md
```

---

## Configuration (environment variables)

| Variable | Default | Description |
|---|---|---|
| `GATEWAY_URL` | `http://gateway:8080` | Base URL for all API calls |
| `JOBSEEKER_ROLE` | `Backend Engineer` | Role sent to scraper |
| `JOBSEEKER_STACK` | `Python,FastAPI,...` | Stack filter (comma-separated) |
| `CRAWL_INTERVAL_HOURS` | `8` | How often to run the scrape cycle |
| `DISCOVER_INTERVAL_HOURS` | `24` | How often to run discover + draft cycles |
| `SCRAPER_WAIT_SECS` | `60` | Wait time after triggering scrapers |
| `DISCOVER_DELAY_SECS` | `30` | Delay between each per-job discover call |
| `SCRAPY_PROJECT_DIR` | `/crawler` | CWD for `scrapy crawl` subprocess |
| `SUMMARY_PATH` | `/data/run_summary.json` | Output path for run summary |

---

## Quick start

### Docker (recommended — part of full stack)

```bash
cd /path/to/jobCrawler
docker compose up --build
# Scheduler auto-starts after gateway is healthy
```

### Local dev

```bash
cd scheduler
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

export GATEWAY_URL="http://localhost:8080"
export JOBSEEKER_ROLE="Backend Engineer"
export JOBSEEKER_STACK="Python,Go,FastAPI"

python main.py
```

---

## Triggering a manual run without waiting for the cron

Set the `MANUAL_TASK` environment variable before starting the process.
The scheduler will run that task immediately and exit (no APScheduler loop).

```bash
# Run only the scrape cycle now
MANUAL_TASK=scrape python main.py

# Run only the discover cycle now
MANUAL_TASK=discover python main.py

# Run only the draft cycle now
MANUAL_TASK=draft python main.py

# Run all three cycles in sequence
MANUAL_TASK=all python main.py
```

**Via Docker (without restarting the container):**

```bash
docker compose run --rm \
  -e MANUAL_TASK=scrape \
  -e GATEWAY_URL=http://gateway:8080 \
  scheduler
```

---

## Changing the schedule

Edit `main.py → build_scheduler()`.

The three jobs use `trigger="interval"` with an `hours=` parameter.
To switch to a cron-style trigger (e.g., run at 6 AM every day):

```python
from apscheduler.triggers.cron import CronTrigger

scheduler.add_job(
    func=lambda: _run(tasks.run_scrape_cycle, "scrape"),
    trigger=CronTrigger(hour=6, minute=0, timezone="Asia/Kolkata"),
    id="scrape",
)
```

Alternatively, adjust the `CRAWL_INTERVAL_HOURS` and `DISCOVER_INTERVAL_HOURS`
environment variables without rebuilding the image.

---

## Run summary

The scheduler writes `/data/run_summary.json` after every cycle:

```json
{
  "run_at": "2026-03-19T12:00:00Z",
  "jobs_discovered": 42,
  "contacts_found": 18,
  "emails_drafted": 12,
  "errors": []
}
```

Read it via the gateway:

```bash
curl http://localhost:8080/api/summary
```

The `/data` directory is a shared Docker volume (`scheduler_data`) mounted by
both the **scheduler** (writer) and the **gateway** (reader).

---

## Graceful shutdown

The process handles `SIGTERM` and `SIGINT`:

```
SIGTERM received
  → APScheduler stops accepting new jobs
  → Waits for the currently executing task to complete
  → Exits cleanly
```

Docker sends `SIGTERM` on `docker compose stop` / `docker compose down`,
so no data will be lost mid-cycle.
