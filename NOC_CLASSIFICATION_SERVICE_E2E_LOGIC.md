# NOC Binding & Pipeline Migration — Python Developer Guide

> **Audience:** Python developer implementing the NOC binding server and pipeline integration.  
> **PHP repo path:** `QjOnline-main/public_html/`  
> **Last updated:** 2026-04-29

---

## Table of Contents

1. [Pipeline Overview](#1-pipeline-overview)
2. [Status Machines](#2-status-machines)
3. [Database Tables — Full Schemas](#3-database-tables--full-schemas)
4. [Stage-by-Stage Responsibilities](#4-stage-by-stage-responsibilities)
5. [API Triggers — Both Directions](#5-api-triggers--both-directions)
6. [NOC Binding — What Python Must Write](#6-noc-binding--what-python-must-write)
7. [Multi-NOC Architecture](#7-multi-noc-architecture)
8. [PHP NOC Classification Logic (Reference)](#8-php-noc-classification-logic-reference)
9. [Environment Variables](#9-environment-variables)
10. [Python Files to Create](#10-python-files-to-create)
11. [End-to-End Flow Walkthrough](#11-end-to-end-flow-walkthrough)

---

## 1. Pipeline Overview

The scraping-to-publishing pipeline is **status-driven**. Both PHP and Python share the same MySQL database. API calls between them carry **no payload** — they are pure trigger signals. All coordination happens via status columns in the DB.

```
[Python Scraper]
      |  scrapes jobs, writes to t_jobs_scraped_raw (process_status='scraped')
      |  POST /api/internal/trigger/normalize  -->  [PHP]
      v
[PHP: NormalizeScrapedJobsJob]
      |  reads 'normalizing' rows, inserts/updates t_jobs (noc_status='noc_pending')
      |  POST {PYTHON_BASE_URL}/trigger/noc  -->  [Python]
      v
[Python: NOC Binder]
      |  reads noc_status='noc_pending' jobs
      |  flips to 'noc_binding', classifies, writes t_job_noc_codes rows
      |  flips to 'noc_done'
      |  POST /api/internal/trigger/pr-employer  -->  [PHP]
      v
[PHP: PrEmployerIntelJob]
      |  scores PR likelihood, computes employer intelligence
      |  flips noc_status --> 'complete'
      |  flips process_status --> 'processed'
      v
[Done -- job is live]
```

---

## 2. Status Machines

### 2.1 `t_jobs_scraped_raw.process_status`

| Value | Set by | Meaning |
|---|---|---|
| `pending` | Python (initial insert) | Row created, scrape not yet run |
| `scraped` | Python | Raw JSON written, ready for normalization |
| `normalizing` | PHP (atomic, in transaction) | PHP has claimed this row for processing |
| `noc_pending` | PHP | Normalization done, waiting for Python NOC binding |
| `noc_done` | Python (optional) | Python finished NOC binding for this batch |
| `processed` | PHP | All downstream stages complete |
| `error` | PHP | Normalization failed (malformed JSON, no `jobs` key) |

### 2.2 `t_jobs.noc_status`

| Value | Set by | Meaning |
|---|---|---|
| `noc_pending` | PHP (NormalizeScrapedJobsJob) | Job created/updated, waiting for NOC binding |
| `noc_binding` | **Python** | Python has claimed this job for classification |
| `noc_done` | **Python** | NOC codes written to `t_job_noc_codes`, ready for PR scoring |
| `complete` | PHP (PrEmployerIntelJob) | PR likelihood + employer intel computed, job fully processed |
| `noc_error` | **Python** | Classification failed for this job |

---

## 3. Database Tables — Full Schemas

### 3.1 `t_jobs_scraped_raw`

| Column | Type | Notes |
|---|---|---|
| `id` | BIGINT UNSIGNED PK | Auto-increment |
| `t_aggregator_id` | BIGINT UNSIGNED | FK -> `t_aggregators.id` |
| `raw_json` | LONGTEXT | Full scraped payload. Must contain `{"jobs": [...]}` |
| `process_status` | ENUM | See section 2.1 above |
| `is_processed` | TINYINT(1) | 1 when fully done |
| `jobs_processed_count` | INT | How many jobs were inserted/updated |
| `jobs_excluded_count` | INT | How many jobs were filtered out |
| `expired_status` | TINYINT(1) | 1 when a zero-job scrape triggered expiry review |
| `created_at` | TIMESTAMP | |
| `updated_at` | TIMESTAMP | |

**Python writes:** `process_status='scraped'` after a successful scrape run.  
**Python optionally writes:** `process_status='noc_done'` after finishing NOC binding for a batch (see section 4.3).

### 3.2 `t_jobs`

Relevant columns only (full table is much larger):

| Column | Type | Notes |
|---|---|---|
| `id` | BIGINT UNSIGNED PK | |
| `title` | VARCHAR | Job title from scraper |
| `description` | TEXT | Full job description |
| `location` | VARCHAR | |
| `t_aggregator_id` | BIGINT UNSIGNED | Which aggregator this came from |
| `employer_id` | BIGINT UNSIGNED | FK -> `t_employers.id` |
| `noc_code` | VARCHAR(10) | **Primary NOC code** (mirror of `is_primary=1` row in `t_job_noc_codes`) |
| `teer_level` | TINYINT | TEER level of the primary NOC code (0-5) |
| `noc_confidence` | DECIMAL(5,4) | Confidence of primary NOC match |
| `noc_match_layer` | VARCHAR | Which layer produced the primary match |
| `noc_review_status` | ENUM | `auto_assigned`, `flagged_for_review`, `admin_confirmed`, `admin_overridden`, or NULL |
| `noc_status` | ENUM | Pipeline status (see section 2.2) |
| `noc_codes_count` | TINYINT UNSIGNED | Number of rows written to `t_job_noc_codes` (0 = legacy/unbound) |
| `noc_reviewed_at` | TIMESTAMP nullable | When an admin last reviewed |
| `noc_overridden_by` | INT nullable | Admin user ID if overridden |
| `status` | ENUM | `live`, `inactive`, `admin_review`, etc. — do not touch |
| `created_at` | TIMESTAMP | |
| `updated_at` | TIMESTAMP | |

**Python reads:** rows where `noc_status='noc_pending'` to find work.  
**Python writes:**
- `noc_status='noc_binding'` when claiming a job
- `noc_code`, `teer_level`, `noc_confidence`, `noc_match_layer`, `noc_review_status` — primary NOC mirror
- `noc_codes_count` — count of rows inserted into `t_job_noc_codes`
- `noc_status='noc_done'` when all codes are written
- `noc_status='noc_error'` on failure

### 3.3 `t_job_noc_codes` — Python's primary output table

| Column | Type | Notes |
|---|---|---|
| `id` | BIGINT UNSIGNED PK | Auto-increment |
| `t_job_id` | BIGINT UNSIGNED | FK -> `t_jobs.id` ON DELETE CASCADE |
| `noc_code` | VARCHAR(10) | 5-digit NOC 2021 code, e.g. `"21230"` |
| `teer_level` | TINYINT UNSIGNED | Second digit of NOC code (0-5) |
| `confidence` | DECIMAL(5,4) | Match confidence 0.0000-1.0000 |
| `is_primary` | TINYINT(1) DEFAULT 0 | **1 for the highest-confidence NOC only** |
| `match_layer` | VARCHAR(50) nullable | How it was matched (see section 6.4) |
| `review_required` | TINYINT(1) DEFAULT 0 | 1 if confidence is low but above minimum threshold |
| `created_at` | TIMESTAMP | |
| `updated_at` | TIMESTAMP | |

Indexes on: `t_job_id`, `noc_code`, composite `(t_job_id, is_primary)`.

### 3.4 `t_noc_keywords` (PHP-managed, Python may read)

| Column | Type | Notes |
|---|---|---|
| `id` | BIGINT UNSIGNED PK | |
| `noc_code` | VARCHAR(10) | 5-digit NOC code |
| `noc_title` | VARCHAR | Human-readable title for the NOC |
| `teer_level` | TINYINT | |
| `keywords` | TEXT | Comma-separated general keywords |
| `strong_keywords` | TEXT | Comma-separated strong-signal keywords |
| `title_keywords` | TEXT | Comma-separated title-specific keywords |
| `negative_keywords` | TEXT | Comma-separated exclusion keywords |

### 3.5 `t_whitelisted_employers`

| Column | Type | Notes |
|---|---|---|
| `id` | BIGINT UNSIGNED PK | |
| `name` | VARCHAR(255) UNIQUE | Raw company name as entered by admin |
| `is_active` | TINYINT(1) DEFAULT 1 | Only active entries are used for filtering |
| `created_at` | TIMESTAMP | |
| `updated_at` | TIMESTAMP | |

> **Note for Python scraper:** When the whitelist is non-empty, PHP's normalization stage already filters out non-whitelisted companies before jobs reach `t_jobs`. Python does not need to enforce this filter.

---

## 4. Stage-by-Stage Responsibilities

### 4.1 Python — Scraping Stage

1. Run scraper for an aggregator.
2. Insert a row into `t_jobs_scraped_raw`:
   ```sql
   INSERT INTO t_jobs_scraped_raw
     (t_aggregator_id, raw_json, process_status, created_at, updated_at)
   VALUES
     (?, '{"jobs": [...]}', 'scraped', NOW(), NOW())
   ```
   The `raw_json` must be valid JSON with a top-level `jobs` array. Each element must have at minimum: `title`, `company`, `location`, `description`, `link`.
3. Fire trigger to PHP:
   ```
   POST /api/internal/trigger/normalize
   Header: X-Internal-Token: <INTERNAL_API_TOKEN>
   Body: (empty)
   ```

### 4.2 PHP — Normalization Stage

PHP handles this automatically after receiving the trigger. Python has no action here.

PHP will:
- Atomically flip `process_status = 'scraped'` to `'normalizing'`
- Filter jobs by whitelist, exclude/include keywords
- Insert/update rows in `t_jobs` with `noc_status = 'noc_pending'`
- Fire trigger to Python NOC server:
  ```
  POST {PYTHON_BASE_URL}/trigger/noc
  Header: X-Internal-Token: <INTERNAL_API_TOKEN>
  Body: (empty)
  ```
- Set raw rows to `process_status = 'noc_pending'`

### 4.3 Python — NOC Binding Stage

This is Python's primary responsibility. Triggered by PHP or by Python's own scheduler.

```
On trigger received at /trigger/noc:
  1. Query: SELECT id, title, description FROM t_jobs WHERE noc_status = 'noc_pending'
  2. For each job (process in batches of 50-100):
     a. UPDATE t_jobs SET noc_status='noc_binding' WHERE id=? AND noc_status='noc_pending'
        (atomic claim -- skip if 0 rows affected)
     b. Run classification logic -> produce list of NOC codes with confidence scores
     c. INSERT INTO t_job_noc_codes ... (one row per qualifying NOC code)
     d. UPDATE t_jobs SET
           noc_code=<primary_noc>,
           teer_level=<primary_teer>,
           noc_confidence=<primary_conf>,
           noc_match_layer=<layer>,
           noc_review_status=<status>,
           noc_codes_count=<count>,
           noc_status='noc_done'
        WHERE id=?
  3. After all jobs processed, fire trigger to PHP:
     POST {PHP_BASE_URL}/api/internal/trigger/pr-employer
     Header: X-Internal-Token: <INTERNAL_API_TOKEN>
     Body: (empty)
  4. (Optional) Update t_jobs_scraped_raw: SET process_status='noc_done'
     WHERE t_aggregator_id IN (<aggregators whose jobs are all noc_done>)
```

### 4.4 PHP — PR Likelihood + Employer Intel Stage

PHP handles this automatically. Python has no action here.

PHP's `PrEmployerIntelJob` will:
- Score PR likelihood for all `noc_status='noc_done'` jobs
- Recompute employer intelligence for touched employers
- Flip `noc_status='complete'` for each job
- Flip `process_status='processed'` for raw records whose jobs are all complete

---

## 5. API Triggers — Both Directions

### 5.1 PHP Endpoints (Python calls these)

**Base URL:** `https://yourdomain.com` (or `http://localhost` in dev)

| Endpoint | Method | When to call | PHP action |
|---|---|---|---|
| `/api/internal/trigger/normalize` | POST | After writing scraped rows | Dispatches `NormalizeScrapedJobsJob` |
| `/api/internal/trigger/pr-employer` | POST | After NOC binding is done | Dispatches `PrEmployerIntelJob` |

**Required header for both:**
```
X-Internal-Token: <INTERNAL_API_TOKEN>
Content-Type: application/json
```

**Response (both endpoints):**
```
HTTP 202 Accepted
{"message": "normalize job dispatched"}
```

Error: `401 Unauthorized` if token is missing or wrong.

### 5.2 Python Endpoints (PHP calls these)

Python must expose a FastAPI (or similar) server. PHP expects:

| Endpoint | Method | When PHP calls it | Python action |
|---|---|---|---|
| `/trigger/noc` | POST | After normalization batch is complete | Begin NOC binding |

**PHP sends:**
```
POST {PYTHON_BASE_URL}/trigger/noc
Header: X-Internal-Token: <INTERNAL_API_TOKEN>
Body: (empty)
```

PHP treats any response (even a timeout) as non-fatal. Python's own scheduler will also poll for `noc_pending` jobs independently, so if the trigger is missed, the work still happens.

**Python should respond:**
```
HTTP 202 Accepted
{"message": "noc binding started"}
```

---

## 6. NOC Binding — What Python Must Write

### 6.1 Confidence Thresholds

| Threshold | Value | Effect |
|---|---|---|
| `CONFIDENCE_AUTO` | `0.55` | Include NOC code, set `noc_review_status='auto_assigned'` |
| `CONFIDENCE_REVIEW` | `0.30` | Include NOC code but set `review_required=1` |
| Minimum | `0.30` | Below this, do NOT insert into `t_job_noc_codes` |

These thresholds apply per-NOC-code. A single job may have some codes above 0.55 and others between 0.30-0.55.

### 6.2 Rules for Each NOC Code Candidate

```python
qualified_codes = []
for noc_code, confidence, teer_level, match_layer in classified_codes:
    if confidence >= 0.55:
        review_required = 0
        noc_review_status = 'auto_assigned'
    elif confidence >= 0.30:
        review_required = 1
        noc_review_status = 'flagged_for_review'
    else:
        continue  # skip -- below minimum threshold
    qualified_codes.append({
        'noc_code': noc_code,
        'teer_level': teer_level,
        'confidence': confidence,
        'match_layer': match_layer,
        'review_required': review_required,
        'noc_review_status': noc_review_status,
    })

# Sort by confidence desc, take top 5
qualified_codes = sorted(qualified_codes, key=lambda x: -x['confidence'])[:5]
```

### 6.3 Primary NOC Code Selection

- The highest-confidence code gets `is_primary = 1`
- All others get `is_primary = 0`
- Mirror the primary code back to `t_jobs`:

```sql
UPDATE t_jobs SET
  noc_code          = '<primary_noc_code>',
  teer_level        = <primary_teer_level>,
  noc_confidence    = <primary_confidence>,
  noc_match_layer   = '<primary_match_layer>',
  noc_review_status = '<auto_assigned|flagged_for_review>',
  noc_codes_count   = <total_codes_inserted>,
  noc_status        = 'noc_done',
  updated_at        = NOW()
WHERE id = <job_id>
```

### 6.4 Match Layer Values

Use these exact strings in the `match_layer` column — PHP's downstream services read them:

| Layer | String value | Confidence range |
|---|---|---|
| Exact profile title match | `profile_exact` | 0.90-0.95 |
| Fuzzy profile title match | `profile_fuzzy` | 0.60-0.92 |
| DB keyword scoring | `keyword` | 0.18-1.00 |
| TEER-only inference | `teer_infer` | 0.08-0.38 |
| Python ML model | `python_ml` | any |
| Python semantic match | `python_semantic` | any |
| Admin override | `admin` | 1.00 |

You may use your own strings for Python-specific methods. The values above come from the existing PHP implementation — matching them improves compatibility with PHP admin tools.

### 6.5 Error Handling

If classification fails for a job:
```sql
UPDATE t_jobs SET noc_status='noc_error', updated_at=NOW() WHERE id=?
```

PHP will not re-attempt `noc_error` jobs automatically. Handle retries in Python or expose an admin endpoint.

---

## 7. Multi-NOC Architecture

### Why multiple NOC codes per job?

A job listing for "Software Engineer — Machine Learning & DevOps" might legitimately belong to multiple NOC codes (e.g., `21230` Computer systems developers, `21211` Data scientists, `21223` Web designers). Storing all relevant codes improves downstream PR scoring accuracy.

### How PHP uses `t_job_noc_codes`

**PR Likelihood Scoring (`PrLikelihoodScoringService`):**
- Reads ALL rows for a job from `t_job_noc_codes` (not just primary)
- Weights each NOC's provincial demand score by that NOC's confidence
- Formula: `weighted_demand = sum(confidence_i * demand_score_i) / sum(confidence_i)`
- If no `t_job_noc_codes` rows exist (`noc_codes_count=0`), falls back to `t_jobs.noc_code`

**Employer Intelligence (`EmployerIntelligenceService`):**
- Joins `t_job_noc_codes WHERE is_primary=1` to `t_employers` to compute top NOC codes per employer
- Falls back to `t_jobs.noc_code` for legacy jobs with `noc_codes_count=0`

### Maximum codes per job

Up to **5** NOC codes per job. If your classifier produces more, take the top 5 by confidence after filtering by minimum threshold.

---

## 8. PHP NOC Classification Logic (Reference)

The existing PHP classifier (`NocClassificationService`) uses 4 layers. Python should aim to replicate or improve on this logic using its own models.

### Layer A — Profile Exact Match (`profile_exact`)

- Source: `app/Services/noc_profiles.json`
- Each profile has `example_job_titles` and `index_of_titles` arrays
- Title is normalized: lowercase, strip punctuation (keep `'` and `-`), collapse spaces
- Synonym map applied first (e.g. "react developer" -> "software developer")
- Variants generated: strip trailing parentheticals, split on ` - ` / ` | `
- Confidence: `example_job_titles` -> **0.95**, `index_of_titles` -> **0.90**

### Layer B — Profile Fuzzy Match (`profile_fuzzy`)

- Same `noc_profiles.json` source
- Tokenize title, remove stopwords (a, an, the, and, or, of, in, at, to, for, with...), min token length 3
- Bidirectional token similarity (Jaccard-like with per-token Levenshtein)
- Minimum similarity threshold: **0.72**
- Confidence formula:
  - From `example_job_titles`: `min(0.92, 0.68 + sim * 0.24)`
  - From `index_of_titles`: `min(0.85, 0.60 + sim * 0.25)`

### Layer C — DB Keyword Scoring (`keyword`)

- Source: `t_noc_keywords` table
- Scoring weights:

| Match type | Location | Weight |
|---|---|---|
| `strong_keywords` | title | 2.5 (x3.0 if phrase = 7.5) |
| `title_keywords` | title | 1.5 (x3.0 if phrase = 4.5) |
| `keywords` | title | 1.4 (x3.0 if phrase = 4.2) |
| `keywords` | description (first 2000 chars) | 0.4 (x3.0 if phrase = 1.2) |
| `negative_keywords` | anywhere | -3.0 penalty |

- Confidence formula: `min(1.0, 0.18 + score * 0.012 + title_token_overlap * 0.35)`

### Layer D — TEER Inference (`teer_infer`)

- Source: `teer_codes` table
- Matches job against TEER-level keyword descriptors
- Produces a TEER level only (no specific NOC code, `noc_code=NULL`)
- Confidence: `min(0.38, 0.08 + score * 0.06)` — **always below CONFIDENCE_AUTO**
- Used only as absolute last resort when no other layer matched

### Confidence to Review Status Mapping (PHP single-NOC, reference)

| Confidence | `noc_review_status` on `t_jobs` |
|---|---|
| >= 0.55 | `auto_assigned` |
| 0.20 to 0.54 | `flagged_for_review` |
| < 0.20 | NULL (unclassified) |

> The PHP thresholds use 0.20 for single-NOC review. For multi-NOC insertion into `t_job_noc_codes`, the minimum threshold was raised to **0.30** to reduce noise.

---

## 9. Environment Variables

### On Python server

```env
# PHP triggers this URL when normalization is done
PHP_BASE_URL=https://yourdomain.com

# Shared secret for authenticating internal API calls
INTERNAL_API_TOKEN=your-shared-secret-here

# MySQL connection
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=your_db_name
DB_USER=your_db_user
DB_PASSWORD=your_db_password
```

### On PHP server (Laravel `.env`)

```env
# Python triggers this URL when NOC binding is done
PYTHON_BASE_URL=http://127.0.0.1:8765

# Shared secret -- must match Python's INTERNAL_API_TOKEN
INTERNAL_API_TOKEN=your-shared-secret-here
```

---

## 10. Python Files to Create

### `noc_server.py` — FastAPI trigger server

```python
import os
from fastapi import FastAPI, Header, HTTPException, BackgroundTasks
from noc_binder import run_noc_binding

app = FastAPI()
INTERNAL_TOKEN = os.getenv("INTERNAL_API_TOKEN")

@app.post("/trigger/noc", status_code=202)
async def trigger_noc(
    background_tasks: BackgroundTasks,
    x_internal_token: str = Header(None)
):
    if x_internal_token != INTERNAL_TOKEN:
        raise HTTPException(status_code=401, detail="Unauthorized")
    background_tasks.add_task(run_noc_binding)
    return {"message": "noc binding started"}
```

### `noc_binder.py` — Core classification loop

```python
def run_noc_binding():
    jobs = fetch_pending_jobs()          # SELECT WHERE noc_status='noc_pending'
    for job in jobs:
        if not claim_job(job['id']):     # atomic UPDATE WHERE noc_status='noc_pending'
            continue
        try:
            codes = classify(job)        # your ML / semantic / keyword logic
            write_noc_codes(job['id'], codes)
            finalize_job(job['id'], codes)
        except Exception as e:
            mark_error(job['id'])

    if jobs:
        notify_php_pr_employer()


def claim_job(job_id: int) -> bool:
    result = db.execute(
        "UPDATE t_jobs SET noc_status='noc_binding', updated_at=NOW()"
        " WHERE id=? AND noc_status='noc_pending'",
        [job_id]
    )
    return result.rowcount == 1


def write_noc_codes(job_id: int, raw_codes: list[dict]):
    # raw_codes = [{"noc_code": "21230", "teer_level": 2, "confidence": 0.87, "match_layer": "python_ml"}, ...]
    qualified = [c for c in raw_codes if c['confidence'] >= 0.30]
    qualified = sorted(qualified, key=lambda x: -x['confidence'])[:5]

    for i, code in enumerate(qualified):
        review_required = 0 if code['confidence'] >= 0.55 else 1
        db.execute("""
            INSERT INTO t_job_noc_codes
              (t_job_id, noc_code, teer_level, confidence, is_primary,
               match_layer, review_required, created_at, updated_at)
            VALUES (?, ?, ?, ?, ?, ?, ?, NOW(), NOW())
        """, [
            job_id,
            code['noc_code'],
            code['teer_level'],
            round(code['confidence'], 4),
            1 if i == 0 else 0,
            code['match_layer'],
            review_required,
        ])


def finalize_job(job_id: int, raw_codes: list[dict]):
    qualified = [c for c in raw_codes if c['confidence'] >= 0.30]
    if not qualified:
        mark_error(job_id)
        return

    primary = max(qualified, key=lambda x: x['confidence'])
    review_status = 'auto_assigned' if primary['confidence'] >= 0.55 else 'flagged_for_review'
    count = min(len(qualified), 5)

    db.execute("""
        UPDATE t_jobs SET
          noc_code=?, teer_level=?, noc_confidence=?,
          noc_match_layer=?, noc_review_status=?,
          noc_codes_count=?, noc_status='noc_done', updated_at=NOW()
        WHERE id=?
    """, [
        primary['noc_code'], primary['teer_level'], round(primary['confidence'], 4),
        primary['match_layer'], review_status, count, job_id
    ])


def mark_error(job_id: int):
    db.execute(
        "UPDATE t_jobs SET noc_status='noc_error', updated_at=NOW() WHERE id=?",
        [job_id]
    )
```

### `api_client.py` — PHP trigger calls

```python
import os, requests

PHP_BASE_URL = os.getenv("PHP_BASE_URL", "http://localhost")
INTERNAL_TOKEN = os.getenv("INTERNAL_API_TOKEN")
HEADERS = {"X-Internal-Token": INTERNAL_TOKEN}

def notify_php_normalize():
    """Call after writing scraped rows -- PHP will normalize."""
    try:
        requests.post(f"{PHP_BASE_URL}/api/internal/trigger/normalize",
                      headers=HEADERS, timeout=5)
    except Exception:
        pass  # PHP also polls on a schedule as fallback

def notify_php_pr_employer():
    """Call after NOC binding is done -- PHP will score PR + employer intel."""
    try:
        requests.post(f"{PHP_BASE_URL}/api/internal/trigger/pr-employer",
                      headers=HEADERS, timeout=5)
    except Exception:
        pass
```

### `scheduler.py` — Fallback poller

```python
import schedule, time
from noc_binder import run_noc_binding

# Poll every 5 minutes as safety net if trigger was missed
schedule.every(5).minutes.do(run_noc_binding)

while True:
    schedule.run_pending()
    time.sleep(30)
```

---

## 11. End-to-End Flow Walkthrough

```
TIME 0: Python scraper runs for aggregator ID=7
  - Writes t_jobs_scraped_raw row:
      {t_aggregator_id:7, raw_json:'{"jobs":[...]}', process_status:'scraped'}
  - POST https://yourdomain.com/api/internal/trigger/normalize
    Header: X-Internal-Token: abc123
    -> PHP responds 202

TIME 1: PHP NormalizeScrapedJobsJob executes (async, queued)
  - DB transaction: UPDATE t_jobs_scraped_raw SET process_status='normalizing'
      WHERE process_status='scraped'
  - Reads raw rows, filters jobs through whitelist + keyword rules
  - Inserts/updates rows in t_jobs with noc_status='noc_pending'
  - Sets t_jobs_scraped_raw: process_status='noc_pending'
  - POST http://127.0.0.1:8765/trigger/noc
    Header: X-Internal-Token: abc123
    -> Python responds 202

TIME 2: Python NOC Binder executes (triggered or scheduled)
  - SELECT id,title,description FROM t_jobs WHERE noc_status='noc_pending' LIMIT 100
  - For job id=1001 "Machine Learning Engineer at Shopify":
      UPDATE t_jobs SET noc_status='noc_binding'
        WHERE id=1001 AND noc_status='noc_pending'   -- atomic claim
      -> classifies -> [
           {noc_code:'21211', conf:0.91, layer:'python_ml', teer:1},
           {noc_code:'21230', conf:0.74, layer:'python_semantic', teer:2}
         ]
      INSERT t_job_noc_codes (job=1001, noc='21211', conf=0.91, is_primary=1, review_required=0)
      INSERT t_job_noc_codes (job=1001, noc='21230', conf=0.74, is_primary=0, review_required=0)
      UPDATE t_jobs SET
        noc_code='21211', teer_level=1, noc_confidence=0.91,
        noc_match_layer='python_ml', noc_review_status='auto_assigned',
        noc_codes_count=2, noc_status='noc_done'
      WHERE id=1001
  - (repeat for all noc_pending jobs)
  - POST https://yourdomain.com/api/internal/trigger/pr-employer

TIME 3: PHP PrEmployerIntelJob executes
  - SELECT * FROM t_jobs WHERE noc_status='noc_done' (chunks of 100)
  - For job 1001: joins t_job_noc_codes, computes confidence-weighted PR demand score
  - UPDATE t_jobs SET noc_status='complete' WHERE id=1001
  - Recomputes employer intelligence for Shopify (employer_id=X)
  - Checks all jobs for aggregator 7 are 'complete'
  - UPDATE t_jobs_scraped_raw SET process_status='processed'

DONE: Job is visible, PR scores are set, employer intelligence is updated.
```

---

## Quick Reference Card

```
Python WRITES to DB:
  t_jobs_scraped_raw.process_status = 'scraped'       (after scrape)
  t_jobs.noc_status                 = 'noc_binding'   (when claiming)
  t_job_noc_codes                   = INSERT rows      (classification result)
  t_jobs.*                          = UPDATE primary NOC + noc_status='noc_done'
  t_jobs.noc_status                 = 'noc_error'      (on failure)

Python READS from DB:
  t_jobs WHERE noc_status='noc_pending'               (find work)

Python CALLS PHP:
  POST /api/internal/trigger/normalize                 (after scrape)
  POST /api/internal/trigger/pr-employer               (after NOC binding)

PHP CALLS Python:
  POST {PYTHON_BASE_URL}/trigger/noc                   (after normalize)

Auth header (all internal calls):
  X-Internal-Token: <INTERNAL_API_TOKEN>
```
