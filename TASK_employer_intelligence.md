# TASK: Employer Intelligence Layer
**Feature:** 3.1 — Track Hiring Patterns and PR Alignment  
**Branch:** `employer-intelligence`  
**Depends on:**
- `TASK_noc_assignment.md` — jobs must have `noc_code` + `teer_level` before employer stats are meaningful
- `TASK_pr_likelihood_scoring.md` — employer stats aggregate `pr_likelihood_score` from jobs; that column must exist

**Build order:** NOC Assignment → PR Likelihood Scoring → **Employer Intelligence**

---

## Ownership Boundary

Three tasks work together on job and employer data. This table shows exactly what each task owns:

| Column / Table | Owned by |
|---------------|----------|
| `t_jobs.noc_code`, `teer_level`, `noc_confidence`, `noc_review_status` | NOC Assignment |
| `t_jobs.pr_likelihood_score`, `pr_likelihood_label`, `province_code` | PR Likelihood Scoring |
| `t_noc_keywords` table | NOC Assignment |
| `t_provincial_noc_demand` table | PR Likelihood Scoring |
| `t_employers.pr_alignment_score`, `admin_flag`, `admin_score_override` | **This task** |
| `t_employer_stats` table | **This task** |

**Employer Intelligence reads** `pr_likelihood_score` from `t_jobs` — it does not write it.  
**Employer Intelligence reads** `noc_code` and `teer_level` from `t_jobs` — it does not write them.

---

## Client Brief — Exact Words

**Q1 — Hiring patterns:**
> "How frequently employer post jobs overtime. Such as role repetition, posting frequency, location. What we want to achieve is: 'This employer consistently hires for X role in Y location'"

**Q2 — PR alignment signals:**
> "It's more probabilistic, not absolute."
>
> **Employer-level signals (what this task computes):**
> - Repeated hiring of PR-eligible roles
> - Hiring in PR-friendly regions (rural, Atlantic)
> - Job structure consistency (not casual/temporary gigs)
>
> *(Job-level signals — NOC/TEER, wage, job type, location — are computed by NOC Assignment and PR Likelihood Scoring and fed into employer stats as aggregated numbers)*

**Q3 — Source:**
> "Hybrid. Semi-automated: employer gets a default PR alignment score based on job history, role types, location. Admin override layer: admins can boost/reduce employer score, flag 'Known PR-supportive employer' or 'Low quality / agency spam'."

**Q4 — User visibility:**
> "On job listing: 'Employer hires frequently for this role', 'Consistent hiring in this location'.  
> On employer profile: 'High likelihood of ongoing hiring', 'Roles aligned with PR pathways', 'This employer has a strong history of hiring for PR-eligible roles'"

---

## Context & Why

QuestJobs' competitive moat is:
> "Anyone can scrape jobs. Almost no one can say: 'We know which employers actually give you a realistic shot at PR.'"

This feature aggregates all the per-job intelligence (NOC codes, PR scores, locations, job types) computed by the upstream tasks into **employer-level signals** — showing users which companies are genuinely worth targeting for permanent residency.

**Scope of this task:**
| What | In/Out |
|------|--------|
| Employer-level stats cache (posting frequency, top roles, top locations) | ✅ In |
| Employer PR alignment score aggregated from job `pr_likelihood_score` values | ✅ In |
| Admin flag + score override UI | ✅ In |
| User-facing signal strings on job listing + employer profile | ✅ In |
| Nightly batch recompute artisan command | ✅ In |
| Public API endpoint for employer intel | ✅ In |
| Job-level PR scoring | ❌ Owned by PR Likelihood Scoring task |
| NOC keyword mapping | ❌ Owned by NOC Assignment task |
| Location-based PR rules | ❌ Owned by PR Likelihood Scoring task |
| ML-based predictions | ❌ Phase 2 |
| Historical employer sponsorship data | ❌ Not publicly available |

---

## Current Codebase State

### Relevant existing models
| Model | Table | Key fields used by this task |
|-------|-------|------------------------------|
| `TEmployer` | `t_employers` | `name`, `logo` |
| `TJob` | `t_jobs` | `employer_id`, `pr_likelihood_score` *(added by PR Likelihood task)*, `noc_code` *(added by NOC task)*, `job_type_id`, `location`, `posted_at` |
| `TJobType` | `t_job_types` | `title` |

### What does NOT exist today
- No `pr_alignment_score`, `admin_flag`, `admin_score_override` on `t_employers`
- No `t_employer_stats` table
- No admin employer management page
- No user-visible employer intelligence signals
- No nightly employer stats computation

### When employer stats can be computed
Employer stats aggregate `pr_likelihood_score` from `t_jobs`. This means the nightly `employer:recompute-intel` command **must run after** `jobs:normalize` has already scored the day's new jobs via the NOC Assignment and PR Likelihood pipelines.

**Nightly sequence:**
```
00:00  jobs:normalize         → scrapes + classifies NOC + scores pr_likelihood_score
02:00  employer:recompute-intel → aggregates job scores into employer stats
```

---

## Employer PR Alignment Score Formula

The employer score has three components, each weighted:

```
base          = avg(t_jobs.pr_likelihood_score) for this employer's jobs in last 90 days / 100
frequency_f   = min(1.0, posting_frequency_30d / 5)      — caps at 5 jobs/month
consistency_f = pr_eligible_pct                           — % of jobs with pr_likelihood_score >= 50

computed      = (base × 0.5) + (frequency_f × 0.2) + (consistency_f × 0.3)
final         = clamp(computed + (admin_score_override × 0.1),  min: 0.0,  max: 1.0)
```

| Component | Weight | What it measures |
|-----------|--------|-----------------|
| Base | 50% | Average quality of PR-eligible jobs this employer posts |
| Consistency | 30% | What fraction of their jobs are PR-eligible (score ≥ 50) |
| Frequency | 20% | How actively they are hiring right now |

**Score bands:**
| Band | Score | User-facing label |
|------|-------|------------------|
| High | ≥ 0.70 | "Strong PR pathway alignment" |
| Medium | 0.40–0.69 | "Moderate PR pathway alignment" |
| Low | < 0.40 | "Limited PR pathway signals" |

**Admin override:**  
`admin_score_override` is an integer −2 to +2. Each point = ±0.10 on the final score, clamped 0.0–1.0.  
Override nudges the algorithm — the underlying computed score is preserved and still shown in admin detail.

---

## Database Design

### 1. Alter `t_employers` — add intel columns

```
pr_alignment_score      float unsigned default 0.0
admin_score_override    tinyint default 0            — range -2 to +2
admin_flag              enum('known_pr_supportive','low_quality','agency_spam') null
admin_note              varchar(255) null
flagged_by              bigint unsigned FK null → users.id (set null on delete)
stats_computed_at       timestamp null
```

### 2. New table: `t_employer_stats`

```
id                       bigint unsigned PK auto-increment
employer_id              bigint unsigned FK → t_employers.id (cascade delete)
total_jobs_count         int unsigned default 0
active_jobs_count        int unsigned default 0
distinct_titles_count    int unsigned default 0
distinct_locations_count int unsigned default 0
pr_eligible_jobs_count   int unsigned default 0    — jobs where pr_likelihood_score >= 50
pr_eligible_pct          float unsigned default 0  — pr_eligible_jobs_count / total_jobs_count
avg_pr_likelihood_score  float unsigned default 0  — avg of t_jobs.pr_likelihood_score
posting_frequency_30d    int unsigned default 0    — jobs posted in last 30 days
posting_frequency_90d    int unsigned default 0
last_posted_at           timestamp null
top_titles               json null                  — ["Welder","Electrician"]
top_locations            json null                  — ["North Bay, ON","Moncton, NB"]
signals_json             json null                  — pre-generated user-facing strings
computed_at              timestamp
```

**Indexes:**
- `employer_id` unique — one stats row per employer
- `avg_pr_likelihood_score` — for admin list sorting

### Migration files to create
```
2026_04_XX_000001_add_intel_columns_to_t_employers_table.php
2026_04_XX_000002_create_t_employer_stats_table.php
```

*(No `t_jobs` migration here — `pr_likelihood_score` is added by PR Likelihood Scoring task. No `t_noc_keywords` here — owned by NOC Assignment task.)*

---

## New Files to Create

```
app/
  Models/
    TEmployerStats.php                              — Eloquent model for t_employer_stats
  Services/
    EmployerIntelligenceService.php                 — Core: stats compute, signal generation, admin override
  Console/Commands/
    RecomputeEmployerIntel.php                      — php artisan employer:recompute-intel
  Http/Controllers/
    Admin/EmployerIntelController.php               — Admin: list, show, flag, recompute
    Api/EmployerIntelApiController.php              — GET /api/employers/{id}/intel

resources/views/admin/jobs-panel/
  employers/
    index.blade.php                                 — Employer list with scores + flags
    show.blade.php                                  — Employer detail + job breakdown
    partials/
      flag-modal.blade.php                          — Flag + override input modal
      signals-card.blade.php                        — Computed signal strings display
```

---

## Files to Modify

| File | What changes |
|------|-------------|
| `app/Models/TEmployer.php` | Add `stats()`, `flaggedBy()` relationships; `prAlignmentLabel()` accessor |
| `app/Models/TJob.php` | No column changes — reads `pr_likelihood_score` which PR Likelihood task already added |
| `resources/views/admin/jobs-panel/layouts/app.blade.php` | Add "Employers" entry under Jobs Aggregation accordion |
| `routes/web.php` | Add admin employer intel routes |
| `routes/api.php` | Add public API intel endpoint |
| `app/Console/Kernel.php` | Register `employer:recompute-intel` daily at 02:00 |

---

## Implementation Plan — Step by Step

---

### Phase 1 — Migrations + Models

**Step 1.1** — Create migration `add_intel_columns_to_t_employers_table.php`

Adds: `pr_alignment_score`, `admin_score_override`, `admin_flag`, `admin_note`, `flagged_by`, `stats_computed_at`.  
Foreign key: `flagged_by → users.id` ON DELETE SET NULL.

**Step 1.2** — Create migration `create_t_employer_stats_table.php`

Creates `t_employer_stats` with all columns above.  
Unique index on `employer_id`. Index on `avg_pr_likelihood_score`.

**Step 1.3** — Create `app/Models/TEmployerStats.php`

```php
class TEmployerStats extends Model {
    const UPDATED_AT = null;
    protected $table = 't_employer_stats';
    protected $fillable = [
        'employer_id', 'total_jobs_count', 'active_jobs_count',
        'distinct_titles_count', 'distinct_locations_count',
        'pr_eligible_jobs_count', 'pr_eligible_pct', 'avg_pr_likelihood_score',
        'posting_frequency_30d', 'posting_frequency_90d',
        'last_posted_at', 'top_titles', 'top_locations', 'signals_json', 'computed_at',
    ];
    protected $casts = [
        'top_titles'    => 'array',
        'top_locations' => 'array',
        'signals_json'  => 'array',
        'last_posted_at'=> 'datetime',
        'computed_at'   => 'datetime',
    ];
    public function employer() {
        return $this->belongsTo(TEmployer::class);
    }
}
```

**Step 1.4** — Update `app/Models/TEmployer.php`

```php
public function stats() {
    return $this->hasOne(TEmployerStats::class, 'employer_id');
}
public function flaggedBy() {
    return $this->belongsTo(User::class, 'flagged_by');
}
public function getPrAlignmentLabelAttribute(): string {
    $score = $this->pr_alignment_score ?? 0;
    if ($score >= 0.70) return 'High';
    if ($score >= 0.40) return 'Medium';
    return 'Low';
}
```

---

### Phase 2 — EmployerIntelligenceService

Create `app/Services/EmployerIntelligenceService.php` with these public methods:

**`computeEmployerStats(TEmployer $employer): void`**

```
1. Query all t_jobs WHERE employer_id = $employer->id

2. From those jobs compute:
   - total_jobs_count
   - active_jobs_count     (status = 'live')
   - distinct_titles_count (count distinct title values)
   - distinct_locations_count
   - pr_eligible_jobs_count  (pr_likelihood_score >= 50)
   - pr_eligible_pct         (pr_eligible / total, as 0.00–1.00)
   - avg_pr_likelihood_score  (avg of pr_likelihood_score, null jobs excluded)
   - posting_frequency_30d   (count posted_at >= now() - 30 days)
   - posting_frequency_90d   (count posted_at >= now() - 90 days)
   - last_posted_at          (max posted_at)
   - top_titles              (top 3 most frequent job titles)
   - top_locations           (top 3 most frequent locations)

3. Compute employer pr_alignment_score:
   base          = avg_pr_likelihood_score / 100
   frequency_f   = min(1.0, posting_frequency_30d / 5)
   consistency_f = pr_eligible_pct
   computed      = (base × 0.5) + (frequency_f × 0.2) + (consistency_f × 0.3)
   final         = clamp(computed + (employer->admin_score_override × 0.1), 0.0, 1.0)

4. Generate signal strings:
   signals_json  = generateSignals($employer, $stats)

5. Upsert t_employer_stats row (updateOrCreate on employer_id)

6. Update t_employers:
   pr_alignment_score = final
   stats_computed_at  = now()

7. Wrap steps 5–6 in DB::transaction()
```

**`generateSignals(TEmployer $employer, TEmployerStats $stats): array`**

Generates plain-English signal strings stored in `signals_json`:

```php
$signals = [];

if ($stats->top_titles && $stats->top_locations) {
    $signals['hiring_pattern'] = "Consistently hires {$stats->top_titles[0]} in {$stats->top_locations[0]}";
}

$pct = round($stats->pr_eligible_pct * 100);
if ($pct >= 50) {
    $signals['pr_alignment'] = "{$pct}% of roles are PR-eligible";
}

if ($stats->posting_frequency_30d > 0) {
    $signals['frequency'] = "Posted {$stats->posting_frequency_30d} jobs in the past 30 days";
}

if ($employer->admin_flag === 'known_pr_supportive') {
    $signals['admin_flag'] = "Known PR-supportive employer";
}

return $signals;
```

**`applyAdminOverride(TEmployer $employer, int $override, ?string $flag, ?string $note, int $adminId): void`**
- Validates `$override` in range −2..+2
- Validates `$flag` against the enum values (or null to clear)
- Updates `t_employers`: `admin_score_override`, `admin_flag`, `admin_note`, `flagged_by`
- Immediately calls `computeEmployerStats($employer)` to regenerate final score

**`batchRecompute(?int $employerId = null): void`**
- If `$employerId` given: recompute that one employer only
- Otherwise: chunk through all employers, call `computeEmployerStats()` on each
- Used by the Artisan command

---

### Phase 3 — Artisan Command

Create `app/Console/Commands/RecomputeEmployerIntel.php`:

```php
// Signature:  employer:recompute-intel {--employer= : specific employer ID}
// Schedule:   daily at 02:00 AM (runs AFTER jobs:normalize which runs at ~00:00)
// Output:     "Processed N employers. High: X, Medium: Y, Low: Z"
```

Register in `app/Console/Kernel.php`:
```php
$schedule->command('employer:recompute-intel')->dailyAt('02:00');
```

---

### Phase 4 — Admin Controller

Create `app/Http/Controllers/Admin/EmployerIntelController.php`:

**`index(Request $request)`**
- Paginated list of all employers joined with `t_employer_stats`
- Sortable by: `pr_alignment_score`, `posting_frequency_30d`, `last_posted_at`
- Filterable by: `admin_flag`, score band (high/medium/low)
- Returns blade view `employers/index`

**`show(int $id)`**
- Loads employer + stats + recent 20 jobs with `pr_likelihood_score`, `noc_code`, `teer_level`
- Shows computed score separately from final score so admin can see effect of override
- Returns blade view `employers/show`

**`updateFlag(Request $request, int $id)`** — POST
- Validates: `override` (integer −2..+2), `flag` (enum or null), `note` (string nullable)
- Calls `EmployerIntelligenceService::applyAdminOverride()`
- Returns redirect back with success flash

**`recompute(int $id)`** — POST
- Calls `EmployerIntelligenceService::computeEmployerStats()`
- Returns JSON `{ success: true, new_score: 0.82, label: "High" }`

---

### Phase 5 — Public API Endpoint

Create `app/Http/Controllers/Api/EmployerIntelApiController.php`:

**`show(int $employerId)`** — GET

Returns:
```json
{
  "employer_id": 42,
  "name": "Northern Steel Works",
  "pr_alignment_score": 0.82,
  "pr_alignment_label": "High",
  "admin_flag": "known_pr_supportive",
  "signals": {
    "hiring_pattern": "Consistently hires Welders in North Bay, ON",
    "pr_alignment": "78% of roles are PR-eligible",
    "frequency": "Posted 8 jobs in the past 30 days",
    "admin_flag": "Known PR-supportive employer"
  },
  "stats": {
    "total_jobs": 34,
    "active_jobs": 12,
    "posting_frequency_30d": 8,
    "last_posted_at": "2026-04-15"
  }
}
```

---

### Phase 6 — Routes

**Admin routes** (inside `admin/jobs-panel` group in `web.php`):
```php
use App\Http\Controllers\Admin\EmployerIntelController;

Route::prefix('employers')->name('employers.')->group(function () {
    Route::get('/',                [EmployerIntelController::class, 'index'])->name('index');
    Route::get('/{id}',            [EmployerIntelController::class, 'show'])->name('show');
    Route::post('/{id}/flag',      [EmployerIntelController::class, 'updateFlag'])->name('flag');
    Route::post('/{id}/recompute', [EmployerIntelController::class, 'recompute'])->name('recompute');
});
```

**API routes** (`routes/api.php`):
```php
use App\Http\Controllers\Api\EmployerIntelApiController;

Route::get('/employers/{employerId}/intel', [EmployerIntelApiController::class, 'show']);
```

**Sidebar** (`layouts/app.blade.php`) — add under Jobs Aggregation accordion:
```php
<li class="list-group-item bg-light border-0">
    <a href="{{ route('admin.jobs-panel.employers.index') }}" class="text-decoration-none">
        <i class="bi bi-building me-2"></i> Employers
    </a>
</li>
```

---

### Phase 7 — Admin UI Views

**`employers/index.blade.php`** — Employer list:
```
┌────────────────────┬──────────┬──────────┬────────┬──────────────────┬──────────┐
│ Employer           │ PR Score │ Band     │ Jobs   │ Posting/30d      │ Flag     │
├────────────────────┼──────────┼──────────┼────────┼──────────────────┼──────────┤
│ Northern Steel     │  0.82    │ High     │  34    │  8               │ ✅ PR    │
│ GTA Temp Agency    │  0.12    │ Low      │  89    │  22              │ 🚫 Spam  │
│ Atlantic Health    │  0.63    │ Medium   │  14    │  3               │  —       │
└────────────────────┴──────────┴──────────┴────────┴──────────────────┴──────────┘
```
- "Flag / Override" button → opens `flag-modal.blade.php`
- "View" → employer detail
- "Recompute" button → triggers POST, refreshes row score via JS

**`employers/show.blade.php`** — Employer detail:
- Score card: `computed_score` (raw formula result) vs `final_score` (after override) — both shown
- Admin override currently applied: −2 / −1 / 0 / +1 / +2
- Signals card: all entries from `signals_json`
- Top roles and top locations (badge chips)
- Recent 20 jobs table: title, location, `noc_code`, `teer_level`, `pr_likelihood_score`, `pr_likelihood_label`

**`partials/flag-modal.blade.php`** — Admin override form:
```
┌──────────────────────────────────────────────────────┐
│ Score Override: [ -2 | -1 |  0  | +1 | +2 ]         │
│ Flag:     [ Known PR-supportive ▼ ]                  │
│ Note:     [ e.g. Verified sponsor, IRCC confirmed ]  │
│                                [Cancel]  [Save]      │
└──────────────────────────────────────────────────────┘
```

---

## Data Flow Summary

```
jobs:normalize (per scrape cycle):
    1. NocClassificationService::classify($job)   [NOC Assignment task]
       → writes noc_code, teer_level to t_jobs
    2. PrLikelihoodScoringService::score($job)    [PR Likelihood task]
       → writes pr_likelihood_score, pr_likelihood_label to t_jobs
       → reads t_employers.pr_alignment_score for Signal 3
         (uses current value — neutral 50 pts if not yet computed)

Nightly at 02:00 AM:
    employer:recompute-intel
    → for each employer:
        reads all t_jobs.pr_likelihood_score for this employer
        computes: avg, pr_eligible_pct, frequency, top_titles, top_locations
        applies formula → pr_alignment_score
        generates signals_json
        writes to t_employer_stats + t_employers.pr_alignment_score

Next scrape cycle (tomorrow):
    PrLikelihoodScoringService reads updated pr_alignment_score
    → employer signal is now real data, not neutral fallback

API:
    GET /api/employers/{id}/intel → signals + score from t_employer_stats
    GET /api/jobs (filtered by pr_likelihood_label) → user sees label if package allows
Admin:
    jobs-panel/employers → list, override, flag, force recompute
```

---

## Dependency Notes

**Circular dependency — resolved by timing:**

| Task | Reads from | Writes to |
|------|-----------|----------|
| NOC Assignment | `t_noc_keywords` | `t_jobs.noc_code`, `teer_level` |
| PR Likelihood Scoring | `t_jobs.noc_code` + `t_employers.pr_alignment_score` | `t_jobs.pr_likelihood_score` |
| **Employer Intelligence** | `t_jobs.pr_likelihood_score` | `t_employers.pr_alignment_score` |

PR Likelihood reads the employer score, and employer stats depend on job scores. This is resolved by:
- First run: employer score = 0.0 (default) → PR Likelihood uses neutral 50 pts for employer signal
- After first nightly batch: real employer scores flow into next day's job scoring
- The system self-corrects within 24 hours of first deployment

---

## Verification Checklist

- [ ] Migration adds all intel columns to `t_employers` with correct FK on `flagged_by`
- [ ] Migration creates `t_employer_stats` with unique index on `employer_id`
- [ ] `TEmployerStats` model casts `top_titles`, `top_locations`, `signals_json` as arrays
- [ ] `TEmployer` model `prAlignmentLabel` accessor returns correct band for 0.80, 0.55, 0.30
- [ ] `computeEmployerStats()` correctly reads `pr_likelihood_score` (not `pr_score`)
- [ ] `avg_pr_likelihood_score` excludes jobs where `pr_likelihood_score IS NULL`
- [ ] `pr_eligible_jobs_count` uses threshold `pr_likelihood_score >= 50`
- [ ] `posting_frequency_30d` counts only jobs with `posted_at >= now() - 30 days`
- [ ] `top_titles` and `top_locations` reflect the 3 most frequent values correctly
- [ ] `signals_json` is populated with non-empty strings after recompute
- [ ] Formula: employer with avg job score 72, freq 8/month, 85% eligible → score ≈ 0.82
- [ ] Formula: employer with avg job score 18, freq 22/month, 12% eligible → score ≈ 0.33
- [ ] Admin override +2 raises final score by exactly 0.20 (capped at 1.0)
- [ ] Admin override −2 lowers final score by exactly 0.20 (floored at 0.0)
- [ ] Admin detail page shows both `computed_score` and `final_score` separately
- [ ] Admin flag `known_pr_supportive` appears in API response `signals.admin_flag`
- [ ] Admin flag `agency_spam` is visible in employer list with correct icon
- [ ] `applyAdminOverride()` immediately recomputes and updates `pr_alignment_score`
- [ ] Employer detail page jobs table shows `pr_likelihood_score` and `pr_likelihood_label`
- [ ] Public API `/api/employers/{id}/intel` returns all signal strings correctly
- [ ] Admin "Recompute" button returns `{ success: true, new_score, label }` JSON
- [ ] `employer:recompute-intel --employer=42` recomputes only employer 42
- [ ] `employer:recompute-intel` (no argument) processes all employers in chunks
- [ ] Command is scheduled at 02:00 AM in `Kernel.php`
- [ ] `flagged_by` stores correct admin user ID when flag is applied

---

## Do NOT

- Do not write `noc_code` or `teer_level` from this service — those are owned by NOC Assignment
- Do not write `pr_likelihood_score` or `pr_likelihood_label` from this service — owned by PR Likelihood Scoring
- Do not include `t_noc_keywords` migrations or models in this task — owned by NOC Assignment
- Do not compute scores on every page load — always read from stored columns
- Do not delete or change any existing `t_employers` columns
- Do not show the raw `pr_alignment_score` float to end users — only show label strings and signal sentences
- Do not run employer stats synchronously during `jobs:normalize` — it is a nightly batch only
- Do not hard-delete employer records — the `admin_flag` enum is the tool for marking spam

---

## File Reference Summary

| File | Status |
|------|--------|
| `database/migrations/2026_04_XX_000001_add_intel_columns_to_t_employers_table.php` | CREATE |
| `database/migrations/2026_04_XX_000002_create_t_employer_stats_table.php` | CREATE |
| `app/Models/TEmployerStats.php` | CREATE |
| `app/Services/EmployerIntelligenceService.php` | CREATE |
| `app/Console/Commands/RecomputeEmployerIntel.php` | CREATE |
| `app/Http/Controllers/Admin/EmployerIntelController.php` | CREATE |
| `app/Http/Controllers/Api/EmployerIntelApiController.php` | CREATE |
| `resources/views/admin/jobs-panel/employers/index.blade.php` | CREATE |
| `resources/views/admin/jobs-panel/employers/show.blade.php` | CREATE |
| `resources/views/admin/jobs-panel/employers/partials/flag-modal.blade.php` | CREATE |
| `resources/views/admin/jobs-panel/employers/partials/signals-card.blade.php` | CREATE |
| `app/Models/TEmployer.php` | MODIFY (add relationships + accessor) |
| `resources/views/admin/jobs-panel/layouts/app.blade.php` | MODIFY (add Employers sidebar entry) |
| `routes/web.php` | MODIFY (add employer admin routes) |
| `routes/api.php` | MODIFY (add public intel endpoint) |
| `app/Console/Kernel.php` | MODIFY (register nightly schedule) |
