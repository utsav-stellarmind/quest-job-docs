# Job Pipeline Migration Plan
## Status-Driven, API-Triggered Architecture with Multi-NOC Support

---

## 1. Overview

### Current Architecture
```
Python Scrape
    → t_jobs_scraped_raw (status=scraped)
    → PHP jobs:normalize (polls DB)
        → t_jobs
        → NOC classify (single NOC per job)
        → PR likelihood (single NOC)
        → Employer intelligence (single NOC)
```

### Target Architecture
```
Python Scrape
    → t_jobs_scraped_raw (status=scraped)
    → [trigger] → PHP normalize
        → t_jobs (noc_status=noc_pending)
    → [trigger] → Python NOC binding (multiple NOC per job)
        → t_job_noc_codes (1..N rows per job)
        → t_jobs (noc_status=noc_done)
    → [trigger] → PHP PR likelihood (multi-NOC aware)
               → PHP Employer intelligence (multi-NOC aware)
        → t_jobs (noc_status=complete), t_jobs_scraped_raw (status=processed)
```

**Key principle:** API calls are pure trigger signals — no payload, no IDs. Both sides have DB access and use status fields to find their own work queue.

---

## 2. Database Schema Changes

### 2.1 New Table: `t_job_noc_codes`

Stores all NOC codes bound to a job (replaces single `noc_code` on `t_jobs` for multi-NOC).

```sql
CREATE TABLE t_job_noc_codes (
    id                  BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    t_job_id            BIGINT UNSIGNED NOT NULL,
    noc_code            VARCHAR(10) NOT NULL,
    teer_level          TINYINT UNSIGNED NOT NULL,
    confidence          DECIMAL(5,4) NOT NULL,
    is_primary          TINYINT(1) DEFAULT 0,   -- highest confidence match
    match_layer         VARCHAR(50) NULL,        -- e.g. title_match, desc_match
    review_required     TINYINT(1) DEFAULT 0,    -- confidence between 0.30–0.54
    created_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at          TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,

    INDEX idx_job_id         (t_job_id),
    INDEX idx_noc_code       (noc_code),
    INDEX idx_primary        (t_job_id, is_primary),
    FOREIGN KEY (t_job_id) REFERENCES t_jobs(id) ON DELETE CASCADE
);
```

### 2.2 New Column on `t_jobs`: `noc_status`

Tracks where the job is in the NOC + downstream pipeline.

```sql
ALTER TABLE t_jobs
    ADD COLUMN noc_status ENUM(
        'noc_pending',   -- normalized, waiting for Python NOC binding
        'noc_binding',   -- Python currently processing (lock guard)
        'noc_done',      -- Python NOC done, waiting for PHP PR + employer
        'complete',      -- PR likelihood + employer intel done
        'noc_error'      -- NOC binding failed
    ) NOT NULL DEFAULT 'noc_pending' AFTER noc_review_status,

    ADD COLUMN noc_codes_count TINYINT UNSIGNED DEFAULT 0 AFTER noc_status;
    -- how many NOC codes were bound (0 = unclassified)
```

### 2.3 `t_jobs_scraped_raw.process_status` — New Values

Add two intermediate states to existing ENUM:

```
pending → scraped → normalizing → noc_pending → noc_done → processed
                                                           ↘ error
```

```sql
ALTER TABLE t_jobs_scraped_raw
    MODIFY COLUMN process_status ENUM(
        'pending',
        'scraped',
        'normalizing',   -- PHP is currently processing this raw record
        'noc_pending',   -- normalized into t_jobs, awaiting NOC
        'noc_done',      -- NOC done, awaiting PR + employer
        'processed',     -- fully complete
        'error'
    ) NOT NULL DEFAULT 'pending';
```

### 2.4 `t_jobs` — Backward Compatibility

`t_jobs.noc_code`, `t_jobs.teer_level`, `t_jobs.noc_confidence`, `t_jobs.noc_match_layer` — **kept as-is**.
They will store the **primary NOC** (highest confidence from `t_job_noc_codes`) so existing queries, UI, and exports continue to work without modification.

---

## 3. API Trigger Endpoints

All endpoints accept `POST` with **no request body**. They return `202 Accepted` immediately and process asynchronously. Authentication via shared `X-Internal-Token` header (value from `.env`).

### 3.1 PHP Internal Routes (Laravel)

| Route | Triggered By | Action |
|---|---|---|
| `POST /api/internal/trigger/normalize` | Python (after scrape done) | PHP normalizes all `process_status='scraped'` rows into `t_jobs` |
| `POST /api/internal/trigger/pr-employer` | Python (after NOC binding done) | PHP runs PR likelihood + employer intel for all `noc_status='noc_done'` jobs |

**Middleware:** `InternalTokenMiddleware` — validates `X-Internal-Token` header against `config('app.internal_api_token')`.

### 3.2 Python Trigger Server (FastAPI)

| Route | Triggered By | Action |
|---|---|---|
| `POST /trigger/noc` | PHP (after normalize done) | Python reads all `noc_status='noc_pending'` jobs, runs multi-NOC binding |

Runs as a lightweight FastAPI server alongside the existing orchestrator process (separate thread or process, same deployment).

---

## 4. Stage-by-Stage Implementation

---

### Stage 1 — Python: Fire Normalize Trigger After Scrape

**File:** `Jobs-Panel-Parser-main/src/orchestrator.py`  
**File:** `Jobs-Panel-Parser-main/src/api_client.py` (new)

After `mark_job_done()` succeeds, Python fires the normalize trigger:

```
mark_job_done(raw_id, raw_json, status='scraped')
    → api_client.trigger_normalize()
        → POST {PHP_BASE_URL}/api/internal/trigger/normalize
        → headers: { X-Internal-Token: {token} }
        → fire-and-forget (log error on failure, do not block scraping)
```

**Retry logic:** If HTTP call fails, log the error. A fallback cron on PHP side (`jobs:normalize` command) can still poll for missed `scraped` records as a safety net — this is not removed, just demoted to fallback.

---

### Stage 2 — PHP: Normalize Endpoint

**File:** `QjOnline-main/app/Http/Controllers/Internal/TriggerController.php` (new)  
**File:** `QjOnline-main/app/Jobs/NormalizeScrapedJobsJob.php` (new Laravel queued job)

Flow:
```
POST /api/internal/trigger/normalize
    → dispatch(NormalizeScrapedJobsJob)    ← returns 202 immediately
    → NormalizeScrapedJobsJob::handle()
        → SELECT * FROM t_jobs_scraped_raw WHERE process_status='scraped' FOR UPDATE
        → flip status to 'normalizing'     ← lock guard against double-process
        → run existing normalization logic (keyword filter, dedup, insert t_jobs)
        → set t_jobs.noc_status = 'noc_pending'
        → set t_jobs_scraped_raw.process_status = 'noc_pending'
        → Guzzle: POST {PYTHON_BASE_URL}/trigger/noc  (fire-and-forget)
```

**Extracted from:** existing `NormalizedProcessedJobs.php` normalize logic — NOC classify and PR scoring calls are **removed** from here and moved downstream.

---

### Stage 3 — Python: NOC Binding Trigger Server + Multi-NOC Logic

**File:** `Jobs-Panel-Parser-main/src/noc_server.py` (new FastAPI server)  
**File:** `Jobs-Panel-Parser-main/src/noc_binder.py` (new, extracted NOC logic)

#### 3.1 FastAPI Server

```
POST /trigger/noc
    → run noc_binder.run_binding() in background thread
    → return 202 immediately
```

#### 3.2 NOC Binding Logic (`noc_binder.py`)

```
run_binding():
    → SELECT t_jobs WHERE noc_status='noc_pending' FOR UPDATE
    → flip noc_status = 'noc_binding'   ← lock guard
    → for each job:
        → fetch title + description from t_jobs
        → run NOC classifier → returns ranked list of (noc_code, confidence, teer, layer)
        → apply thresholds:
            confidence >= 0.55  → include, review_required=0
            confidence >= 0.30  → include, review_required=1
            confidence <  0.30  → exclude
        → take top 5 matches (cap)
        → INSERT INTO t_job_noc_codes (one row per match)
        → mark is_primary=1 on highest confidence row
        → UPDATE t_jobs SET
            noc_code       = primary.noc_code,
            teer_level     = primary.teer_level,
            noc_confidence = primary.confidence,
            noc_match_layer = primary.match_layer,
            noc_codes_count = count_of_bound_nocs,
            noc_status     = 'noc_done'
    → after all jobs processed:
        → UPDATE t_jobs_scraped_raw: process_status = 'noc_done'  (for jobs in this batch)
        → api_client.trigger_pr_employer()
            → POST {PHP_BASE_URL}/api/internal/trigger/pr-employer
```

#### 3.3 Multi-NOC Rule Change Summary

| Rule | Current | New |
|---|---|---|
| Max NOC per job | 1 | Up to 5 |
| Confidence threshold | 0.55 auto, 0.20 review | 0.55 auto, 0.30 review |
| Storage | `t_jobs.noc_code` only | `t_job_noc_codes` table + primary mirrored to `t_jobs` |
| Zero matches | Flag for review | Same — flag for review |

---

### Stage 4 — PHP: PR Likelihood — Multi-NOC Aware

**File:** `QjOnline-main/app/Services/PrLikelihoodScoringService.php`

#### Current Logic (Single NOC)
```
noc_code = t_jobs.noc_code
demand   = t_provincial_noc_demand WHERE noc_code = noc_code
provincial_demand_score = demand.score
```

#### New Logic (Multi-NOC, Confidence-Weighted)

```
noc_codes = t_job_noc_codes WHERE t_job_id = job.id ORDER BY confidence DESC

For each noc_code in noc_codes:
    demand_row = t_provincial_noc_demand WHERE noc_code = noc_code.noc_code
    if demand_row exists:
        weighted_score += demand_row.score × noc_code.confidence
        total_weight   += noc_code.confidence

provincial_demand_score = weighted_score / total_weight
    (falls back to 0 if no NOC codes have demand data)
```

**Why weighted average:** A job with NOC A (conf=0.80, demand=high) and NOC B (conf=0.40, demand=low) should score closer to A. Weighting by confidence achieves this naturally.

**Fallback:** If `t_job_noc_codes` is empty (zero rows for this job), fall back to `t_jobs.noc_code` as before — backward compatible for any jobs that went through the old pipeline.

#### TEER Level for Wage Baseline

Currently uses `t_jobs.teer_level` (single NOC). With multi-NOC:
- Use TEER of the **primary NOC** (`is_primary=1`) for wage baseline lookup.
- Primary NOC = highest confidence = best classification = most appropriate TEER.

---

### Stage 5 — PHP: Employer Intelligence — Multi-NOC Aware

**File:** `QjOnline-main/app/Services/EmployerIntelligenceService.php`

#### Current Logic (Single NOC)
```
top_noc_codes    = GROUP_BY noc_code on employer's jobs
pr_eligible_pct  = jobs WHERE pr_likelihood_label IN ('high','medium') / total
```

#### New Logic (Multi-NOC Aware)

**5.1 Top NOC Codes Aggregation**

Instead of grouping by `t_jobs.noc_code`, join `t_job_noc_codes`:

```sql
SELECT jn.noc_code, COUNT(*) as frequency
FROM t_jobs j
JOIN t_job_noc_codes jn ON jn.t_job_id = j.id
WHERE j.employer_id = :employer_id
  AND j.posted_at >= NOW() - INTERVAL 90 DAY
  AND jn.is_primary = 1          -- use primary NOC per job to avoid double-counting
GROUP BY jn.noc_code
ORDER BY frequency DESC
LIMIT 5
```

Using `is_primary=1` avoids inflating counts. Secondary NOC codes can optionally be surfaced separately as "related occupations".

**5.2 PR Alignment Score**

Current formula:
```
pr_alignment = 0.5 × (avg_pr_score / 100)
             + 0.2 × freq_30d_normalized
             + 0.3 × pr_eligible_pct
```

No change to formula. `avg_pr_score` now reflects multi-NOC-weighted PR scores from Stage 4, so the employer score naturally improves without formula changes.

**5.3 Role Repetition Score**

Currently: counts jobs with same `noc_code`.

New: count jobs where **any** `t_job_noc_codes` entry overlaps (job shares at least one NOC code with other jobs from same employer). This surfaces employers who repeatedly hire for overlapping roles even if the primary NOC differs slightly.

```sql
SELECT COUNT(DISTINCT j2.id)
FROM t_jobs j1
JOIN t_job_noc_codes n1 ON n1.t_job_id = j1.id
JOIN t_job_noc_codes n2 ON n2.noc_code = n1.noc_code AND n2.t_job_id != j1.id
JOIN t_jobs j2 ON j2.id = n2.t_job_id
WHERE j1.employer_id = :employer_id
  AND j2.employer_id = :employer_id
  AND j1.posted_at >= NOW() - INTERVAL 90 DAY
```

---

### Stage 6 — PHP: Finalize

**File:** `QjOnline-main/app/Jobs/PrEmployerIntelJob.php` (new Laravel queued job)

```
POST /api/internal/trigger/pr-employer
    → dispatch(PrEmployerIntelJob)   ← returns 202 immediately
    → PrEmployerIntelJob::handle()
        → SELECT t_jobs WHERE noc_status='noc_done'
        → for each job:
            → PrLikelihoodScoringService::score(job)    ← multi-NOC aware
            → EmployerIntelligenceService::computeForJob(job)  ← multi-NOC aware
            → UPDATE t_jobs SET noc_status='complete'
        → UPDATE t_jobs_scraped_raw SET process_status='processed'
           WHERE related jobs are all noc_status='complete'
```

---

## 5. Full Status Flow (Combined View)

```
t_jobs_scraped_raw.process_status    t_jobs.noc_status     Action
─────────────────────────────────    ─────────────────     ──────────────────────────────
pending                              (no t_jobs row yet)   Waiting to be scraped
scraped                              (no t_jobs row yet)   Python fires normalize trigger
normalizing                          (being created)       PHP normalize in progress
noc_pending                          noc_pending           PHP fires NOC trigger to Python
noc_pending                          noc_binding           Python NOC binding in progress
noc_done                             noc_done              Python fires pr-employer trigger
processed                            complete              Done — all stages complete
error                                noc_error             Failed — manual review needed
```

---

## 6. Fallback Safety Net

The existing `jobs:normalize` artisan command is **not removed**. It is demoted to a scheduled fallback:
- Runs every 15 minutes via cron
- Picks up any `process_status='scraped'` records that were missed (e.g., if Python trigger HTTP call failed)
- Same for `noc_status='noc_pending'` — a Python cron task re-checks periodically

This means the trigger API is an **optimization** (reduces latency), not a single point of failure.

---

## 7. New Files Summary

### Python (`Jobs-Panel-Parser-main/`)

| File | Purpose |
|---|---|
| `src/api_client.py` | Fires HTTP triggers to PHP (`trigger_normalize`, `trigger_pr_employer`) |
| `src/noc_server.py` | FastAPI server exposing `POST /trigger/noc` |
| `src/noc_binder.py` | Multi-NOC binding logic, writes to `t_job_noc_codes` |

### PHP (`QjOnline-main/`)

| File | Purpose |
|---|---|
| `app/Http/Controllers/Internal/TriggerController.php` | Handles all 3 trigger endpoints |
| `app/Http/Middleware/InternalTokenMiddleware.php` | Validates `X-Internal-Token` header |
| `app/Jobs/NormalizeScrapedJobsJob.php` | Queued job: normalize scraped → t_jobs |
| `app/Jobs/PrEmployerIntelJob.php` | Queued job: PR likelihood + employer intel |
| `database/migrations/*_create_t_job_noc_codes_table.php` | New junction table |
| `database/migrations/*_add_noc_status_to_t_jobs_table.php` | New status column |
| `database/migrations/*_update_process_status_enum.php` | Add new enum values |

### Modified Files

| File | Change |
|---|---|
| `src/orchestrator.py` | Call `api_client.trigger_normalize()` after `mark_job_done()` |
| `app/Console/Commands/Jobs/NormalizedProcessedJobs.php` | Remove NOC + PR calls; demote to fallback poller |
| `app/Services/NocClassificationService.php` | Logic moves to Python; PHP version kept for reclassify command only |
| `app/Services/PrLikelihoodScoringService.php` | Multi-NOC confidence-weighted provincial demand |
| `app/Services/EmployerIntelligenceService.php` | Multi-NOC top NOC codes + role repetition query |

---

## 8. Open Questions Before Implementation

1. **Python NOC server deployment:** Same process as orchestrator (threaded) or separate systemd service?
2. **Queue driver for PHP jobs:** Redis (recommended) or database queue? Affects `NormalizeScrapedJobsJob` and `PrEmployerIntelJob` dispatch.
3. **Batch size cap:** Should PHP normalize all pending `scraped` records in one job dispatch, or chunk them (e.g., 100 at a time)?
4. **`t_provincial_noc_demand` coverage:** If a secondary NOC code has no demand row, it contributes 0 weight — confirm this is acceptable vs. skipping it entirely from the weighted average.
5. **Retroactive multi-NOC:** Does the new NOC binding run on existing `t_jobs` records (full reclassify), or only new jobs going forward?
6. **`noc:reclassify` command:** Update this command to also write to `t_job_noc_codes` and clear old rows before reinserting.
