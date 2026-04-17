# TASK: PR Likelihood Scoring
**Feature:** 3.2 — Score Jobs by PR Pathway Likelihood  
**Branch:** `pr-likelihood-scoring`  
**Depends on:** `TASK_noc_assignment.md` (needs `noc_code` + `teer_level` on jobs), `TASK_employer_intelligence.md` (needs `pr_alignment_score` on employers — runs in degraded mode without it)

---

## Client Brief — Exact Words

**Q1 — Which jobs get scored?**
> "Only jobs in certain categories/industries. For example - healthcare is highly in demand in Ontario. So Healthcare jobs in Ontario should receive such PR Likelihood. Whereas in New Brunswick, Educator jobs are in so much demand so those jobs should have the PR Likelihood. So the PR likelihood will depend on which jobs are in demand in each province. These information can be found in each provincial website and federal websites."
>
> Example table client provided:
> | Province | NOC Code | Demand Level |
> |----------|----------|-------------|
> | Ontario | 31301 | High |
> | Ontario | 33102 | High |
> | New Brunswick | 41221 | High |
> | Alberta | 62020 | Medium |
>
> "This information must be hybrid as well — automated and can be edited by admin."

**Q2 — Confirmed scoring signals:**
> Job type, employer, industry (NOC), wage. "That's it."

**Q3 — Weights adjustable?**
> "Must be adjustable overtime as in demand jobs change overtime in every province."

**Q4 — User visibility?**
> "Users must be able to filter by it. But the ability of the user to see this information will be based on the package they pay for."

**Q5 — Score format?**
> Display as: 🟢 High PR Likelihood / 🟡 Medium / 🔴 Low

---

## Context & Why

The PR Likelihood Score is the **user-facing output** of all the intelligence features built so far:

```
NOC Assignment     → identifies what the job is
Employer Intel     → identifies how PR-friendly the employer is
PR Likelihood      → combines everything into one actionable label the user sees
```

This score answers the user's core question: **"Is this job worth applying for if I want to stay in Canada?"**

It is:
- Province-aware (demand varies by province — the same NOC can be High in NB and Medium in AB)
- Package-gated (only paying subscribers see the label and can filter by it)
- Admin-adjustable (demand data and signal weights change as provincial programs update)

---

## Scope

| What | In / Out |
|------|---------|
| Per-job PR likelihood score (0–100) + label (high/medium/low) | ✅ In |
| Provincial NOC demand table (admin-managed) | ✅ In |
| 4-signal weighted scoring formula | ✅ In |
| Admin-configurable signal weights | ✅ In |
| Admin-configurable label thresholds | ✅ In |
| Package-gated visibility (frontend) | ✅ In |
| Filter by PR likelihood (frontend API) | ✅ In |
| Score computed at normalization time | ✅ In |
| Artisan command to recompute all existing jobs | ✅ In |
| User-visible raw score (0–100) | ❌ Only show label, never the number |
| Per-user personalized scoring | ❌ Phase 2 |
| Federal program matching (Express Entry CRS points) | ❌ Phase 2 |

---

## Current Codebase State

### Normalization hook order (after NOC assignment task)
In `app/Console/Commands/Jobs/NormalizedProcessedJobs.php`, after each job is created/updated:
```
1. syncJobRulesByCategory($job)          [existing]
2. NocClassificationService::classify($job)    [NOC task]
3. PrLikelihoodScoringService::score($job)     ← this task hooks here
```

### Package model (`Package` → table `packages1`)
Already has per-feature boolean flags: `teer_noc_access`, `job_criteria_access`, `eligibility_guide_access`, etc.  
**This task adds:** `pr_likelihood_access` boolean to the same table.

### Job types
`t_job_types` rows are free-text strings created by the scraper (e.g., "Full-time", "Part-time", "Contract").  
Scoring normalizes them by keyword match — no migration needed on `t_job_types`.

### What does NOT exist today
- No `pr_likelihood_score`, `pr_likelihood_label`, `province_code` columns on `t_jobs`
- No `t_provincial_noc_demand` table
- No `t_pr_scoring_config` table
- No Provincial Demand admin page
- No Scoring Weights admin page
- No `pr_likelihood_access` on packages

---

## Architecture Decisions

### 1. Province extracted from location string
The job's `location` field is a free text string like `"Toronto, ON"` or `"Moncton, New Brunswick"`.  
A `ProvinceExtractorService` normalises it to a 2–3 char province code (`ON`, `NB`, `BC`, etc.).  
Extracted code is stored on the job as `province_code` so it is not re-parsed on every read.

### 2. All 4 signals always run — provincial demand is the dominant signal
Even if a job's NOC is not in the demand table, the other 3 signals still produce a score.  
A job with no demand data entry gets a TEER-level fallback on the demand signal (TEER 0/1 → 50 pts, TEER 2/3 → 35 pts, TEER 4/5 → 10 pts).  
This means **every job with a NOC gets a score**. Only jobs with `noc_code = null` get `pr_likelihood_score = null`.

### 3. Weights and thresholds stored in `t_pr_scoring_config`
A simple key-value config table with 6 rows (4 weights + 2 label thresholds).  
Admin edits these via a single form. Validation enforces weights sum to exactly 100.

### 4. Score recomputed fresh on each normalize cycle — no stale score problem
Because normalization already runs every 8–10 hours, the score stays current.  
Admin changes to demand data or weights trigger a manual `pr:recompute-likelihood` artisan command.

### 5. Package gating is enforced in the API response, not the DB
The `pr_likelihood_label` and `pr_likelihood_score` are always computed and stored.  
The API checks the user's active package for `pr_likelihood_access` before including them in the response.  
**Why:** Keeps scoring logic clean and separate from access control. Admin can always see scores.

---

## Scoring Formula

### Signal 1 — Provincial Demand (default weight: 40%)

Looks up `(province_code, noc_code)` in `t_provincial_noc_demand`:

| Match Result | Points |
|-------------|--------|
| `demand_level = 'high'` | 100 |
| `demand_level = 'medium'` | 60 |
| `demand_level = 'low'` | 25 |
| Not in demand table — TEER 0 or 1 fallback | 50 |
| Not in demand table — TEER 2 or 3 fallback | 35 |
| Not in demand table — TEER 4 or 5 fallback | 10 |
| No province extracted OR no NOC code | 0 |

### Signal 2 — Job Type (default weight: 25%)

Normalises `t_job_types.title` by keyword:

| Job Type Pattern | Points |
|----------------|--------|
| Title contains "permanent" | 100 |
| Title contains "full" (full-time, not permanent) | 75 |
| Title contains "part" | 40 |
| Title contains "contract" or "temp" or "casual" or "seasonal" | 20 |
| No job type linked / unknown | 50 (neutral) |

### Signal 3 — Employer PR Alignment (default weight: 20%)

Reads `t_employers.pr_alignment_score` (0.0–1.0) × 100:

| Employer State | Points |
|---------------|--------|
| `admin_flag = 'known_pr_supportive'` | max(employer_score × 100, 85) |
| `admin_flag = 'agency_spam'` | 0 |
| `admin_flag = 'low_quality'` | min(employer_score × 100, 20) |
| No flag, score present | employer_score × 100 |
| No employer record OR `stats_computed_at = null` | 50 (neutral) |

### Signal 4 — Wage Level (default weight: 15%)

Compares `t_jobs.salary` (parsed as hourly estimate) against TEER-level baseline rates:

| Condition | Points |
|-----------|--------|
| TEER 0/1: salary present and ≥ $30/hr equivalent | 90 |
| TEER 0/1: salary present and < $30/hr | 30 |
| TEER 2/3: salary present and ≥ $22/hr equivalent | 75 |
| TEER 2/3: salary present and < $22/hr | 25 |
| TEER 4/5: salary present and ≥ $16/hr equivalent | 60 |
| TEER 4/5: salary present and < $16/hr | 15 |
| No salary data | 40 (neutral) |

Salary parsing: strip currency symbols, detect per-year vs per-hour, convert annual ÷ 2080.

### Final Score Calculation

```
pr_likelihood_score =
    (demand_pts  × weight_demand  / 100)
  + (jobtype_pts × weight_job_type / 100)
  + (employer_pts × weight_employer / 100)
  + (wage_pts    × weight_wage    / 100)

Rounded to nearest integer. Range: 0–100.
```

### Label Thresholds (admin-configurable, defaults shown)

| Score | Label | Display |
|-------|-------|---------|
| ≥ 70 | `high` | 🟢 High PR Likelihood |
| ≥ 40 | `medium` | 🟡 Medium PR Likelihood |
| < 40 | `low` | 🔴 Low PR Likelihood |

---

## Province Extraction

`ProvinceExtractorService::extract(string $location): ?string`

Lowercases the location string and scans for province aliases:

| Code | Aliases to match |
|------|----------------|
| `ON` | `ontario`, `, on`, ` on,`, `(on)` |
| `BC` | `british columbia`, `, bc`, ` bc,` |
| `AB` | `alberta`, `, ab`, ` ab,` |
| `MB` | `manitoba`, `, mb` |
| `SK` | `saskatchewan`, `, sk` |
| `QC` | `quebec`, `québec`, `, qc` |
| `NS` | `nova scotia`, `, ns` |
| `NB` | `new brunswick`, `, nb` |
| `NL` | `newfoundland`, `labrador`, `, nl` |
| `PE` | `prince edward island`, `pei`, `, pe` |
| `YT` | `yukon`, `, yt` |
| `NT` | `northwest territories`, `, nt` |
| `NU` | `nunavut`, `, nu` |

Returns `null` if no match (score computed without demand signal).  
Stored as `t_jobs.province_code` — not re-extracted on subsequent normalize cycles unless location changes.

---

## Database Design

### Alter `t_jobs` — add PR likelihood columns

```
province_code           char(3) null            — extracted province: ON, BC, NB, etc.
pr_likelihood_score     tinyint unsigned null   — 0–100
pr_likelihood_label     enum('high','medium','low') null
pr_score_demand_pts     tinyint unsigned null   — stored per-signal for admin debug
pr_score_jobtype_pts    tinyint unsigned null
pr_score_employer_pts   tinyint unsigned null
pr_score_wage_pts       tinyint unsigned null
pr_scored_at            timestamp null
```

**Indexes:**
- `pr_likelihood_label` — for frontend filter queries
- `(province_code, pr_likelihood_label)` — composite for province-filtered searches

### New table: `t_provincial_noc_demand`

```
id              bigint unsigned PK auto-increment
province_code   char(3) not null            — ON, BC, AB, etc.
noc_code        varchar(10) not null        — references t_noc_keywords.noc_code (not strict FK)
demand_level    enum('high','medium','low') not null
source_url      varchar(500) null           — e.g. https://ontario.ca/page/oinp-employer-job-offer
source_date     date null                   — date the source data was valid
notes           text null                   — admin notes e.g. "Updated per OINP Oct 2026 bulletin"
is_active       boolean default true
created_at      timestamp
updated_at      timestamp
created_by      bigint unsigned FK null → users.id (set null on delete)
updated_by      bigint unsigned FK null → users.id (set null on delete)
```

**Indexes:**
- `(province_code, noc_code)` unique — one demand level per province-NOC pair
- `province_code` — for admin list filtering
- `demand_level` — for score engine lookups

### New table: `t_pr_scoring_config`

```
id              bigint unsigned PK auto-increment
config_key      varchar(50) not null unique
config_value    tinyint unsigned not null
description     varchar(255) null
updated_at      timestamp
updated_by      bigint unsigned FK null → users.id
```

**Default rows (seeded):**

| config_key | config_value | description |
|-----------|-------------|-------------|
| `weight_demand` | 40 | Weight % for provincial demand signal |
| `weight_job_type` | 25 | Weight % for job type signal |
| `weight_employer` | 20 | Weight % for employer PR alignment signal |
| `weight_wage` | 15 | Weight % for wage level signal |
| `threshold_high` | 70 | Minimum score for High label |
| `threshold_medium` | 40 | Minimum score for Medium label |

### Alter `packages1` — add access flag

```
pr_likelihood_access    boolean default false
```

### Migration files to create
```
2026_04_XX_000001_add_pr_likelihood_columns_to_t_jobs_table.php
2026_04_XX_000002_create_t_provincial_noc_demand_table.php
2026_04_XX_000003_create_t_pr_scoring_config_table.php
2026_04_XX_000004_add_pr_likelihood_access_to_packages_table.php
```

---

## New Files to Create

```
app/
  Models/
    TProvincialNocDemand.php                        — Eloquent model for t_provincial_noc_demand
    TPrScoringConfig.php                            — Eloquent model for t_pr_scoring_config

  Services/
    ProvinceExtractorService.php                    — Extract province code from location string
    PrLikelihoodScoringService.php                  — Core: 4-signal scoring engine

  Http/Controllers/
    ProvincialDemandController.php                  — Admin CRUD for t_provincial_noc_demand
    PrScoringConfigController.php                   — Admin form for weights + thresholds

  Console/Commands/
    RecomputePrLikelihoodCommand.php                — php artisan pr:recompute-likelihood

database/migrations/
  2026_04_XX_000001_add_pr_likelihood_columns_to_t_jobs_table.php
  2026_04_XX_000002_create_t_provincial_noc_demand_table.php
  2026_04_XX_000003_create_t_pr_scoring_config_table.php
  2026_04_XX_000004_add_pr_likelihood_access_to_packages_table.php

database/seeders/
  ProvincialNocDemandSeeder.php                     — Seed initial demand data
  PrScoringConfigSeeder.php                         — Seed default weights + thresholds

resources/views/admin/jobs-panel/
  provincial-demand/
    index.blade.php                                 — Demand table list + add/edit modals
    modals/
      add.blade.php
      edit.blade.php
  pr-scoring-config/
    index.blade.php                                 — Weights + threshold config form
```

---

## Files to Modify

| File | What changes |
|------|-------------|
| `app/Console/Commands/Jobs/NormalizedProcessedJobs.php` | Call `PrLikelihoodScoringService::score($job)` after NOC classify call |
| `app/Models/TJob.php` | Add new columns to `$fillable` + `$casts` |
| `app/Models/Package.php` | Add `pr_likelihood_access` to `$fillable` |
| `resources/views/admin/jobs-panel/layouts/app.blade.php` | Add new "PR Intelligence" accordion section with 2 menu items |
| `routes/web.php` | Add provincial demand + scoring config routes |

---

## Implementation Plan — Step by Step

---

### Phase 1 — Migrations + Models + Seeds

**Step 1.1** — Four migrations in order:
1. Alter `t_jobs`: add all PR likelihood columns + indexes
2. Create `t_provincial_noc_demand`
3. Create `t_pr_scoring_config`
4. Alter `packages1`: add `pr_likelihood_access`

**Step 1.2** — Create `app/Models/TProvincialNocDemand.php`

```php
class TProvincialNocDemand extends Model {
    protected $table = 't_provincial_noc_demand';
    protected $fillable = [
        'province_code', 'noc_code', 'demand_level', 'source_url',
        'source_date', 'notes', 'is_active', 'created_by', 'updated_by',
    ];
    protected $casts = [
        'is_active'   => 'boolean',
        'source_date' => 'date',
    ];
}
```

**Step 1.3** — Create `app/Models/TPrScoringConfig.php`

```php
class TPrScoringConfig extends Model {
    protected $table = 't_pr_scoring_config';
    public $timestamps = false;
    const UPDATED_AT = 'updated_at';
    protected $fillable = ['config_key', 'config_value', 'description', 'updated_by'];

    // Load all config as key→value array, cached per request
    public static function asArray(): array {
        return static::pluck('config_value', 'config_key')->all();
    }
}
```

**Step 1.4** — Update `TJob` model: add new columns to `$fillable` and `$casts`

```php
// Add to $fillable:
'province_code', 'pr_likelihood_score', 'pr_likelihood_label',
'pr_score_demand_pts', 'pr_score_jobtype_pts', 'pr_score_employer_pts',
'pr_score_wage_pts', 'pr_scored_at',

// Add to $casts:
'pr_likelihood_score'   => 'integer',
'pr_score_demand_pts'   => 'integer',
'pr_score_jobtype_pts'  => 'integer',
'pr_score_employer_pts' => 'integer',
'pr_score_wage_pts'     => 'integer',
'pr_scored_at'          => 'datetime',
```

**Step 1.5** — Update `Package` model: add `pr_likelihood_access` to `$fillable`.

**Step 1.6** — Create `ProvincialNocDemandSeeder.php`

Seed the initial dataset (from client example + well-known provincial demand):

| Province | NOC Code | Demand | Source |
|----------|----------|--------|--------|
| ON | 31301 | high | Registered Nurses — OINP Healthcare stream |
| ON | 33102 | high | PSWs — Ontario Personal Support Workers Initiative |
| ON | 32101 | high | Licensed Practical Nurses — OINP |
| ON | 21231 | high | Software Engineers — OINP Tech Draw |
| ON | 72320 | medium | Welders — OINP Skilled Trades |
| ON | 72410 | medium | Electricians — OINP Skilled Trades |
| NB | 41221 | high | Elementary School Teachers — NBPNP |
| NB | 31301 | high | Registered Nurses — NBPNP Healthcare |
| NB | 33102 | high | PSWs — NBPNP |
| NS | 31301 | high | Registered Nurses — NSNP |
| NS | 33102 | high | PSWs — NSNP |
| NS | 72320 | medium | Welders — NSNP |
| NL | 31301 | high | Registered Nurses — NLPNP |
| NL | 72422 | high | Plumbers — NLPNP Skilled Trades |
| PE | 31301 | high | Registered Nurses — PEI PNP |
| PE | 33102 | high | PSWs — PEI PNP |
| AB | 62020 | medium | Retail Trade Supervisors — AINP (client example) |
| AB | 72320 | high | Welders — AINP Opportunity stream |
| AB | 72410 | high | Electricians — AINP |
| MB | 33102 | high | PSWs — MPNP |
| MB | 31301 | high | Registered Nurses — MPNP |
| BC | 21231 | high | Software Engineers — BC PNP Tech |
| BC | 31301 | high | Registered Nurses — BC PNP |
| SK | 31301 | high | Registered Nurses — SINP |
| SK | 72320 | medium | Welders — SINP |

**Step 1.7** — Create `PrScoringConfigSeeder.php`

Inserts 6 rows with default values as specified in the config table design above.

---

### Phase 2 — ProvinceExtractorService

Create `app/Services/ProvinceExtractorService.php`

**`extract(string $location): ?string`**

```
1. Lowercase the location string
2. Iterate province alias map (ordered — check multi-word names like "british columbia" before "bc")
3. Return first matching province code
4. Return null if no match
```

The alias map is a hardcoded private const array — no DB lookup needed.

---

### Phase 3 — PrLikelihoodScoringService

Create `app/Services/PrLikelihoodScoringService.php`

**`score(TJob $job): void`**

```
1. Skip if noc_code is null (cannot score without NOC)
   → set pr_likelihood_score = null, pr_likelihood_label = null, return

2. Load config (TPrScoringConfig::asArray()) — cached per process run

3. Province extraction:
   - if job->province_code is already set: use it (already extracted)
   - if job->province_code is null: call ProvinceExtractorService::extract($job->location)
   - store province_code on job row

4. Signal 1 — Provincial demand:
   - Query t_provincial_noc_demand WHERE province_code = ? AND noc_code = ? AND is_active = true
   - Map demand_level → points using table above
   - If no row: apply TEER fallback
   - If no province: 0 pts

5. Signal 2 — Job type:
   - Load $job->jobType (eager loaded or lazy — use null-safe)
   - Lowercase the title, match keyword patterns → points

6. Signal 3 — Employer alignment:
   - Load $job->employer (eager loaded)
   - Read employer->pr_alignment_score, employer->admin_flag
   - Apply flag overrides → points

7. Signal 4 — Wage:
   - Parse $job->salary string → numeric hourly rate
   - Compare against TEER-level baseline → points
   - If unparseable or null → 40 pts neutral

8. Compute final score:
   $score = (demand_pts  × $config['weight_demand']  / 100)
          + (jobtype_pts × $config['weight_job_type'] / 100)
          + (employer_pts × $config['weight_employer'] / 100)
          + (wage_pts    × $config['weight_wage']    / 100);
   $score = (int) round($score);

9. Determine label:
   if $score >= $config['threshold_high']  → 'high'
   elseif $score >= $config['threshold_medium'] → 'medium'
   else → 'low'

10. Update job (only PR columns — never touch posted_at, status, noc fields):
    $job->update([
        'province_code'         => $provinceCode,
        'pr_likelihood_score'   => $score,
        'pr_likelihood_label'   => $label,
        'pr_score_demand_pts'   => $demandPts,
        'pr_score_jobtype_pts'  => $jobtypePts,
        'pr_score_employer_pts' => $employerPts,
        'pr_score_wage_pts'     => $wagePts,
        'pr_scored_at'          => now(),
    ]);
```

**`parseSalaryToHourly(?string $salary): ?float`** — private helper  
Handles common formats:
- `"$28/hr"` → 28.0
- `"$55,000/year"` or `"$55,000 annually"` → 55000 / 2080 = 26.4
- `"$25 - $30 per hour"` → average: 27.5
- Returns `null` if unparseable

---

### Phase 4 — Artisan Command

Create `app/Console/Commands/RecomputePrLikelihoodCommand.php`:

```
Signature:  pr:recompute-likelihood {--job= : recompute single job by ID}
            {--province= : recompute all jobs in a province e.g. --province=ON}
Purpose:    Re-run scoring after admin changes demand data or weights
Processing: Chunks through qualifying jobs. Skips jobs with noc_code = null.
Output:     "Scored N jobs. High: X, Medium: Y, Low: Z. Skipped K (no NOC)."
```

Admin runs this manually after:
- Adding/editing rows in `t_provincial_noc_demand`
- Changing weights or thresholds in `t_pr_scoring_config`

---

### Phase 5 — Hook into `jobs:normalize`

**File:** `app/Console/Commands/Jobs/NormalizedProcessedJobs.php`

Add at top of `handle()`:
```php
$prScoringService = app(\App\Services\PrLikelihoodScoringService::class);
```

In **both** the update and create paths, after the NOC classify call:
```php
app(\App\Services\NocClassificationService::class)->classify($job);   // existing (NOC task)
$prScoringService->score($job);                                        // ← this task
```

---

### Phase 6 — Admin Controllers

**`ProvincialDemandController.php`** — CRUD for `t_provincial_noc_demand`

```php
index()     → paginated list, filterable by province_code and demand_level
             → shows NOC title (joined from t_noc_keywords) alongside NOC code
store()     → validate + create; on success: flash + redirect
update()    → validate + update row; return JSON for modal
destroy()   → soft-delete (set is_active = false); never hard delete
```

Validation rules:
```php
'province_code' => 'required|string|size:2|in:ON,BC,AB,MB,SK,QC,NS,NB,NL,PE,YT,NT,NU'
'noc_code'      => 'required|string|max:10'
'demand_level'  => 'required|in:high,medium,low'
'source_url'    => 'nullable|url|max:500'
'source_date'   => 'nullable|date'
'notes'         => 'nullable|string|max:1000'
```

Unique check: `(province_code, noc_code)` — reject if pair already exists.

**`PrScoringConfigController.php`** — Weights + thresholds form

```php
index()     → loads all 6 config rows, renders form
update()    → validates all 6 inputs, saves, returns redirect with success flash

// Validation:
'weight_demand'    => 'required|integer|min:0|max:100'
'weight_job_type'  => 'required|integer|min:0|max:100'
'weight_employer'  => 'required|integer|min:0|max:100'
'weight_wage'      => 'required|integer|min:0|max:100'
// Custom: sum of 4 weights must equal 100
'threshold_high'   => 'required|integer|min:1|max:100'
'threshold_medium' => 'required|integer|min:1|lt:threshold_high'
```

---

### Phase 7 — Admin UI Views

**`provincial-demand/index.blade.php`** — Demand table

Follows existing `categories/index.blade.php` modal pattern:

```
┌──────────┬──────────┬──────────────────────────────────┬──────────┬────────────┬────────┬─────────┐
│ Province │ NOC Code │ NOC Title                        │ Demand   │ Source     │ Active │ Actions │
├──────────┼──────────┼──────────────────────────────────┼──────────┼────────────┼────────┼─────────┤
│ ON       │ 31301    │ Registered nurses                │ 🔴 High  │ OINP site  │  ✅   │ Edit    │
│ NB       │ 41221    │ Elementary and kindergarten...   │ 🔴 High  │ NBPNP site │  ✅   │ Edit    │
│ AB       │ 62020    │ Retail trade supervisors         │ 🟡 Med   │ AINP site  │  ✅   │ Edit    │
└──────────┴──────────┴──────────────────────────────────┴──────────┴────────────┴────────┴─────────┘
```

Province filter dropdown above the table. "Add Entry" button opens `modals/add.blade.php`.

The add modal has:
- Province dropdown (all 13 provinces/territories)
- NOC Code input — autocomplete from `t_noc_keywords` (live search as admin types, shows NOC title)
- Demand level radio: High / Medium / Low
- Source URL input
- Source Date picker
- Notes textarea

**`pr-scoring-config/index.blade.php`** — Weights + threshold form

```
┌─────────────────────────────────────────────────────────────────┐
│  Scoring Signal Weights                                          │
│  (must sum to 100)                                               │
├───────────────────────────┬──────────────────────────────────── ┤
│  Provincial Demand        │  [40] %                              │
│  Job Type                 │  [25] %                              │
│  Employer PR Alignment    │  [20] %                              │
│  Wage Level               │  [15] %                              │
│                           │  Total: [100] ✅                     │
├───────────────────────────┴──────────────────────────────────── ┤
│  Label Thresholds                                                │
├───────────────────────────┬─────────────────────────────────────┤
│  🟢 High: score ≥         │  [70]                               │
│  🟡 Medium: score ≥       │  [40]                               │
│  🔴 Low: score <          │  [40] (auto-derived, read-only)     │
├───────────────────────────┴─────────────────────────────────────┤
│                                              [Save Changes]      │
└─────────────────────────────────────────────────────────────────┘
```

JS validates live: sum of 4 weights shows real-time total, turns red if ≠ 100. Save button disabled until sum = 100.

---

### Phase 8 — Sidebar + Routes

**New sidebar section** in `layouts/app.blade.php` — add a third accordion **"PR Intelligence"** after Taxonomy:

```php
<!-- PR Intelligence -->
<li class="list-group-item p-0 bg-light">
    <div class="accordion accordion-flush" id="accordionPR">
        <div class="accordion-item">
            <h2 class="accordion-header">
                <button class="accordion-button collapsed" data-bs-toggle="collapse"
                    data-bs-target="#collapsePR">
                    <i class="bi bi-graph-up-arrow me-2"></i> PR Intelligence
                </button>
            </h2>
            <div id="collapsePR" class="accordion-collapse collapse show">
                <div class="accordion-body bg-light">
                    <ul class="list-group list-group-flush">
                        <li class="list-group-item bg-light border-0">
                            <a href="{{ route('admin.jobs-panel.provincial-demand.index') }}"
                               class="text-decoration-none">
                                <i class="bi bi-map me-2"></i> Provincial Demand
                            </a>
                        </li>
                        <li class="list-group-item bg-light border-0">
                            <a href="{{ route('admin.jobs-panel.pr-scoring-config.index') }}"
                               class="text-decoration-none">
                                <i class="bi bi-sliders me-2"></i> Scoring Weights
                            </a>
                        </li>
                    </ul>
                </div>
            </div>
        </div>
    </div>
</li>
```

**Routes** (`web.php`) inside `admin/jobs-panel` group:

```php
use App\Http\Controllers\ProvincialDemandController;
use App\Http\Controllers\PrScoringConfigController;

Route::prefix('provincial-demand')->name('provincial-demand.')->group(function () {
    Route::get('/',        [ProvincialDemandController::class, 'index'])->name('index');
    Route::post('/',       [ProvincialDemandController::class, 'store'])->name('store');
    Route::put('/{id}',    [ProvincialDemandController::class, 'update'])->name('update');
    Route::delete('/{id}', [ProvincialDemandController::class, 'destroy'])->name('destroy');
});

Route::prefix('pr-scoring-config')->name('pr-scoring-config.')->group(function () {
    Route::get('/',    [PrScoringConfigController::class, 'index'])->name('index');
    Route::post('/',   [PrScoringConfigController::class, 'update'])->name('update');
});
```

---

## Data Flow Summary

```
jobs:normalize → per job:
    1. syncJobRulesByCategory()                 [existing]
    2. NocClassificationService::classify()     [NOC task — sets noc_code, teer_level]
    3. PrLikelihoodScoringService::score()      [this task]
       │
       ├─ noc_code = null? → skip (pr_likelihood_score = null)
       │
       ├─ ProvinceExtractorService::extract(location) → province_code
       │
       ├─ Signal 1: t_provincial_noc_demand lookup (province + noc_code)
       │   └─ no match? TEER fallback → partial demand score
       │
       ├─ Signal 2: t_job_types.title keyword pattern → job type score
       │
       ├─ Signal 3: t_employers.pr_alignment_score + admin_flag → employer score
       │
       ├─ Signal 4: salary string → parseSalaryToHourly() → wage score
       │
       └─ weighted sum → pr_likelihood_score (0–100) + pr_likelihood_label

Nightly: employer:recompute-intel → updates t_employers.pr_alignment_score
         → next normalize cycle picks up fresh employer scores

Admin changes demand table or weights:
    → run: php artisan pr:recompute-likelihood [--province=ON]
    → all unlocked jobs rescored with new data

API response:
    → check user package pr_likelihood_access
    → if true: include pr_likelihood_label in job response
    → if false: omit entirely (field not visible at all)
```

---

## Verification Checklist

- [ ] Migration adds all PR likelihood columns to `t_jobs` with correct types and indexes
- [ ] Migration creates `t_provincial_noc_demand` with unique index on `(province_code, noc_code)`
- [ ] Migration creates `t_pr_scoring_config` with 6 default rows seeded
- [ ] Migration adds `pr_likelihood_access` to `packages1`
- [ ] `ProvincialNocDemandSeeder` inserts all 25 seed rows without duplicate errors
- [ ] Job in Ontario with NOC 31301 (Registered Nurse) receives `pr_likelihood_label = 'high'`
- [ ] Same job title in a province with no demand data entry still gets a score (TEER fallback, not null)
- [ ] Job with `noc_code = null` gets `pr_likelihood_score = null` (not scored)
- [ ] Job with full-time permanent type scores higher than same job with casual/temp type
- [ ] Employer flagged `agency_spam` drives employer signal to 0 pts
- [ ] Employer flagged `known_pr_supportive` drives employer signal to ≥ 85 pts
- [ ] Salary `"$28/hr"` parses to 28.0 correctly
- [ ] Salary `"$58,000/year"` parses to ~27.9/hr correctly
- [ ] Salary `"$25 - $30 per hour"` averages to 27.5 correctly
- [ ] Null salary produces neutral 40 pts, not an error
- [ ] `province_code` extracted correctly from "Toronto, ON" → `ON`
- [ ] `province_code` extracted from "New Brunswick" → `NB`
- [ ] Location with no province match → `province_code = null`, demand signal = 0 pts
- [ ] Weights in `t_pr_scoring_config` changed to 50/30/10/10 — recomputed scores reflect new weights
- [ ] Threshold changed from 70 to 80 — previously High jobs correctly relabelled to Medium
- [ ] Admin form rejects submission when 4 weights do not sum to 100
- [ ] `pr:recompute-likelihood --province=ON` only updates jobs where `province_code = 'ON'`
- [ ] `pr:recompute-likelihood --job=42` only updates job ID 42
- [ ] Provincial Demand admin page lists all active seed rows with correct province + demand badge
- [ ] Admin can add a new demand row for a province/NOC pair not yet in the table
- [ ] Admin deactivating a demand row: on next normalize or recompute, that NOC falls back to TEER scoring
- [ ] Duplicate `(province_code, noc_code)` rejected at controller validation layer
- [ ] Jobs API: user with `pr_likelihood_access = true` receives `pr_likelihood_label` in response
- [ ] Jobs API: user with `pr_likelihood_access = false` does NOT receive `pr_likelihood_label`
- [ ] Frontend filter by `pr_likelihood_label = 'high'` returns only high-scored jobs
- [ ] `pr_score_demand_pts`, `pr_score_jobtype_pts`, etc. stored correctly for admin debug

---

## Do NOT

- Do not show the raw score (0–100) to end users — only show the label (High / Medium / Low)
- Do not compute PR likelihood without a NOC code — return null silently, do not throw
- Do not hard-delete rows from `t_provincial_noc_demand` — deactivate with `is_active = false`
- Do not allow saving weights that do not sum to 100 — reject at validation
- Do not re-extract `province_code` if it is already set on the job — province doesn't change between scrape cycles
- Do not gate admin visibility of `pr_likelihood_label` — only end-user API responses are gated
- Do not score the `province_code = null` demand signal as "0 and penalise" — it means unknown, so only the other 3 signals contribute (score is still computed, just lower)
- Do not connect to any external Canadian government API — all demand data is admin-entered manually

---

## File Reference Summary

| File | Status |
|------|--------|
| `database/migrations/2026_04_XX_000001_add_pr_likelihood_columns_to_t_jobs_table.php` | CREATE |
| `database/migrations/2026_04_XX_000002_create_t_provincial_noc_demand_table.php` | CREATE |
| `database/migrations/2026_04_XX_000003_create_t_pr_scoring_config_table.php` | CREATE |
| `database/migrations/2026_04_XX_000004_add_pr_likelihood_access_to_packages_table.php` | CREATE |
| `database/seeders/ProvincialNocDemandSeeder.php` | CREATE |
| `database/seeders/PrScoringConfigSeeder.php` | CREATE |
| `app/Models/TProvincialNocDemand.php` | CREATE |
| `app/Models/TPrScoringConfig.php` | CREATE |
| `app/Services/ProvinceExtractorService.php` | CREATE |
| `app/Services/PrLikelihoodScoringService.php` | CREATE |
| `app/Console/Commands/RecomputePrLikelihoodCommand.php` | CREATE |
| `app/Http/Controllers/ProvincialDemandController.php` | CREATE |
| `app/Http/Controllers/PrScoringConfigController.php` | CREATE |
| `resources/views/admin/jobs-panel/provincial-demand/index.blade.php` | CREATE |
| `resources/views/admin/jobs-panel/provincial-demand/modals/add.blade.php` | CREATE |
| `resources/views/admin/jobs-panel/provincial-demand/modals/edit.blade.php` | CREATE |
| `resources/views/admin/jobs-panel/pr-scoring-config/index.blade.php` | CREATE |
| `app/Console/Commands/Jobs/NormalizedProcessedJobs.php` | MODIFY (add score() call) |
| `app/Models/TJob.php` | MODIFY (add new columns to fillable + casts) |
| `app/Models/Package.php` | MODIFY (add pr_likelihood_access to fillable) |
| `resources/views/admin/jobs-panel/layouts/app.blade.php` | MODIFY (add PR Intelligence accordion) |
| `routes/web.php` | MODIFY (add provincial demand + scoring config routes) |
