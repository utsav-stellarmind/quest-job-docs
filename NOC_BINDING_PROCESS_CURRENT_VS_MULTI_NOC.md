# NOC Code Binding Process

This document separates:

1. The **currently implemented** NOC binding pipeline in this codebase.
2. The **multiple NOC per single job** behavior and what changes are expected/required.

---

## 1) Current Implemented NOC Binding Process

## 1.1 Trigger entry points

- Python (after scraping) calls: `POST /api/internal/trigger/normalize`
- Laravel controller: `app/Http/Controllers/Internal/TriggerController.php`
- Laravel dispatches:
  - `NormalizeScrapedJobsJob` for scraped -> normalized flow
  - `PrEmployerIntelJob` for noc_done -> complete flow

Both endpoints are asynchronous (`202 Accepted`) and work as trigger-only APIs.

## 1.2 Normalize stage (Laravel)

File: `app/Jobs/NormalizeScrapedJobsJob.php`

Current implemented behavior:

1. Claims all rows in `t_jobs_scraped_raw` where `process_status='scraped'` by setting them to `normalizing`.
2. Parses raw payload and creates/updates `t_jobs`.
3. Sets each normalized job to `noc_status='noc_pending'`.
4. Marks raw row `process_status='noc_pending'`.
5. Calls Python trigger `POST {python_base_url}/trigger/noc` (fire-and-forget).

## 1.3 NOC status model (implemented in DB migrations)

- New table: `t_job_noc_codes`
  - Stores per-job NOC rows (`noc_code`, `teer_level`, `confidence`, `is_primary`, `match_layer`, `review_required`)
- `t_jobs` additions:
  - `noc_status`: `noc_pending | noc_binding | noc_done | complete | noc_error`
  - `noc_codes_count` (default `0`)
- `t_jobs_scraped_raw.process_status` expanded to:
  - `pending | scraped | normalizing | noc_pending | noc_done | processed | error`

## 1.4 PR + Employer stage (Laravel)

File: `app/Jobs/PrEmployerIntelJob.php`

Current implemented behavior:

1. Picks jobs with `noc_status='noc_done'`.
2. Runs `PrLikelihoodScoringService::score($job)`.
3. Marks job `noc_status='complete'`.
4. Recomputes employer stats for touched employers via `EmployerIntelligenceService`.
5. Finalizes raw records from `noc_done` -> `processed` when no linked jobs remain in pending NOC states.

## 1.5 Current practical status

- Laravel side is already **multi-NOC aware** in scoring and employer intelligence.
- Flow assumes Python NOC binder will populate:
  - `t_job_noc_codes`
  - primary NOC mirror fields on `t_jobs` (`noc_code`, `teer_level`, etc.)
  - transition jobs to `noc_done`

If Python binder writes are missing/incomplete, jobs remain in `noc_pending`/`noc_binding` and downstream stage will not complete.

---

## 2) After Multiple NOC Allowed for Single Job

This section defines the expected behavior when multi-NOC is fully active and consistently used.

## 2.1 Functional behavior

For each job:

1. NOC classifier returns ranked candidates.
2. Keep up to top N matches (current plan uses top 5).
3. Insert all accepted matches into `t_job_noc_codes`.
4. Mark highest-confidence row as `is_primary=1`.
5. Mirror primary values into `t_jobs` for backward compatibility.
6. Set `t_jobs.noc_codes_count = number of inserted NOCs`.
7. Move `t_jobs.noc_status` to `noc_done`.

## 2.2 Downstream behavior with multi-NOC

### PR scoring

File: `app/Services/PrLikelihoodScoringService.php`

- Uses `t_job_noc_codes` ordered by confidence.
- Computes provincial demand as confidence-weighted score across bound NOCs.
- Uses primary NOC TEER for wage signal baseline.
- Falls back to legacy single NOC (`t_jobs.noc_code`) when no junction rows exist.

### Employer intelligence

File: `app/Services/EmployerIntelligenceService.php`

- Top NOC codes:
  - Uses `t_job_noc_codes` with `is_primary=1` to avoid per-job double counting.
  - Unions with legacy `t_jobs.noc_code` where `noc_codes_count=0`.
- Role repetition:
  - Detects overlap when jobs share at least one NOC code.
  - Blends NOC-overlap logic with legacy title-repeat fallback.

## 2.3 Data compatibility strategy

Keep legacy fields on `t_jobs`:

- `noc_code`
- `teer_level`
- `noc_confidence`
- `noc_match_layer`

Reason: existing UI/queries/export paths can continue reading one primary NOC while new logic uses full multi-NOC rows.

## 2.4 Operational checklist for full multi-NOC readiness

1. Python `trigger/noc` endpoint must:
   - read `noc_pending` jobs
   - set transient lock state (`noc_binding`)
   - write rows to `t_job_noc_codes`
   - set primary row and mirror primary fields to `t_jobs`
   - transition jobs to `noc_done`
2. Raw status should be moved to `noc_done` once relevant jobs are NOC-bound.
3. Python should call `POST /api/internal/trigger/pr-employer` after successful batch binding.
4. Monitoring should track stuck jobs by `noc_status` age (`noc_pending`/`noc_binding` not advancing).

---

## 3) Status Flow Reference

### Raw table (`t_jobs_scraped_raw.process_status`)

`pending -> scraped -> normalizing -> noc_pending -> noc_done -> processed`

Error path: `-> error`

### Jobs table (`t_jobs.noc_status`)

`noc_pending -> noc_binding -> noc_done -> complete`

Error path: `-> noc_error`

