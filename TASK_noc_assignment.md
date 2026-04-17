# TASK: NOC Assignment Engine + Admin Panel
**Feature:** 3.1-B — Industry Classification via NOC/TEER Mapping  
**Branch:** `noc-assignment`  
**Depends on:** Nothing (standalone — Employer Intelligence depends on this)  
**Must complete before:** `TASK_employer_intelligence.md` implementation

---

## Context & Why

Canadian NOC (National Occupational Classification) codes are the foundation of every PR pathway:  
- Express Entry points are awarded based on NOC/TEER level  
- OINP, AINP, MPNP streams are NOC-restricted  
- Without a NOC code on a job, the system cannot compute a PR eligibility score  

The client's exact requirement:
> "Category classification is not reliable. We need an NOC and TEER category assignment engine."  
> "If the system has low confidence on whether it provided the correct NOC code to a job title, the system should flag the admin to manually correct the issue."  
> "One job can only have one NOC code."  
> "Admin must be able to edit the NOC code and TEER level."  
> "The system should have its embedded quality control to ensure information are correct."

---

## TEER Reference (Canadian Government — NOC 2021)

| TEER | Occupation Type | Requirement |
|------|----------------|-------------|
| 0 | Management occupations | Management responsibilities |
| 1 | University degree occupations | Bachelor's, master's, or doctorate |
| 2 | College diploma / 2–5yr apprenticeship / supervisory | 2–3yr college, or 2–5yr apprenticeship, or supervisory/safety role |
| 3 | College diploma <2yr / apprenticeship <2yr / 6+ months OJT | Short college, short apprenticeship, or 6+ months on-the-job training |
| 4 | High school diploma / several weeks OJT | Secondary school or several weeks training |
| 5 | No formal education | Short work demonstration, no formal requirements |

**PR pathway eligibility by TEER (Federal Express Entry):**
- TEER 0, 1, 2, 3 → eligible for Federal Skilled Worker, Canadian Experience Class
- TEER 4, 5 → generally NOT eligible for federal programs (may qualify for provincial nominee programs)

---

## Current Codebase State

### Exact hook point — `jobs:normalize`

File: `app/Console/Commands/Jobs/NormalizedProcessedJobs.php`

The normalization loop processes each scraped job and either **creates** or **updates** a `TJob` row.  
Both paths call `$this->syncJobRulesByCategory($job)` as their final step.

**This is the injection point** — after `syncJobRulesByCategory()` on both create and update paths:

```php
// EXISTING (line ~265 for update, ~287 for create):
$this->syncJobRulesByCategory($job);

// ADD THIS after that call:
app(NocClassificationService::class)->classify($job);
```

### What does NOT exist today
- No `noc_code`, `teer_level`, `pr_score` columns on `t_jobs`
- No `t_noc_keywords` table
- No `NocClassificationService`
- No NOC Codes admin page
- No NOC Review Queue in the admin panel

### Existing admin panel patterns to follow
- Sidebar: Bootstrap 5 accordion in `resources/views/admin/jobs-panel/layouts/app.blade.php`
- Taxonomy CRUD pattern: modal-based add/edit (see `jobs-panel/categories/` for reference)
- Review queue with badge: see "Job Review" in sidebar — badge shows pending count
- Admin controllers live in `app/Http/Controllers/` (not a subdirectory for jobs-panel controllers)

---

## Architecture Decisions

### 1. Two-layer classification
- **Layer 1 (keyword match):** Title → keyword lookup in admin-managed `t_noc_keywords` table. Fast, auditable, admin-tunable.
- **Layer 2 (TEER signal inference):** Fallback when no keyword match. Parse title + description against government-defined TEER signal words. Produces a lower confidence score.

### 2. Confidence-based routing
- High confidence (≥ 0.60) → auto-assign, `noc_review_status = 'auto_assigned'`
- Medium confidence (0.30–0.59) → assign best guess + `noc_review_status = 'flagged_for_review'`
- No match (< 0.30) → `noc_code = null` + `noc_review_status = 'flagged_for_review'`

### 3. Admin corrections are permanent locks
Once an admin sets `noc_review_status = 'admin_confirmed'` or `'admin_overridden'`, the normalization loop skips re-classification for that job forever. Only the PR score is recomputed using the locked NOC.

### 4. One NOC per job — enforced at DB level
`t_jobs.noc_code` is a single `varchar(10)` column. The service always resolves to exactly one NOC. No array, no pivot table.

---

## Database Design

### Alter `t_jobs` — add NOC columns

```
noc_code                varchar(10) null        — e.g. "72320"
teer_level              tinyint unsigned null   — 0–5
noc_confidence          float unsigned null     — 0.00–1.00
noc_match_layer         varchar(20) null        — 'keyword' | 'signal' | 'none'
noc_review_status       enum('auto_assigned','flagged_for_review','admin_confirmed','admin_overridden') null
noc_overridden_by       bigint unsigned FK null → users.id (set null on delete)
noc_reviewed_at         timestamp null
```

**Indexes:**
- `noc_review_status` — for the review queue list query
- `noc_code` — for employer stats aggregation

### New table: `t_noc_keywords`

```
id              bigint unsigned PK auto-increment
noc_code        varchar(10) not null        — e.g. "72320"
noc_title       varchar(255) not null       — "Welders and related machine operators"
teer_level      tinyint unsigned not null   — 0–5
keywords        text not null               — comma-separated: "welder,mig welder,tig welder,stick welder,welding"
is_active       boolean default true        — admin can deactivate without deleting
created_at      timestamp
updated_at      timestamp
```

**Indexes:**
- `noc_code` unique
- `teer_level` — for filtering in admin
- `is_active` — exclude inactive from classification

### Migration files to create
```
2026_04_XX_000001_add_noc_columns_to_t_jobs_table.php
2026_04_XX_000002_create_t_noc_keywords_table.php
```

---

## New Files to Create

```
app/
  Models/
    TNocKeyword.php                                     — Eloquent model for t_noc_keywords

  Services/
    NocClassificationService.php                        — Core: keyword match + TEER signal inference

  Http/Controllers/
    NocKeywordController.php                            — Admin CRUD for t_noc_keywords
    NocReviewController.php                             — Admin review queue for flagged jobs

database/migrations/
  2026_04_XX_000001_add_noc_columns_to_t_jobs_table.php
  2026_04_XX_000002_create_t_noc_keywords_table.php

database/seeders/
  NocKeywordsSeeder.php                                 — Seed ~30 common NOC rows

resources/views/admin/jobs-panel/
  noc-keywords/
    index.blade.php                                     — NOC Codes list + add/edit modals
    modals/
      add.blade.php                                     — Add NOC row modal
      edit.blade.php                                    — Edit NOC row modal
  noc-review/
    index.blade.php                                     — NOC review queue: flagged jobs
```

---

## Files to Modify

| File | What changes |
|------|-------------|
| `app/Console/Commands/Jobs/NormalizedProcessedJobs.php` | Add `NocClassificationService::classify($job)` call after `syncJobRulesByCategory()` in both create and update paths |
| `app/Models/TJob.php` | Add new columns to `$fillable` and `$casts` |
| `resources/views/admin/jobs-panel/layouts/app.blade.php` | Add "NOC Codes" under Taxonomy accordion + "NOC Review" under Jobs Aggregation accordion (with badge) |
| `routes/web.php` | Add NOC Codes CRUD routes + NOC Review routes |

---

## Implementation Plan — Step by Step

---

### Phase 1 — Migrations + Models

**Step 1.1** — Create migration `add_noc_columns_to_t_jobs_table.php`

Adds to `t_jobs`: `noc_code`, `teer_level`, `noc_confidence`, `noc_match_layer`, `noc_review_status`, `noc_overridden_by`, `noc_reviewed_at`.

Index on `noc_review_status` (for review queue queries).  
Foreign key: `noc_overridden_by` → `users.id` ON DELETE SET NULL.

**Step 1.2** — Create migration `create_t_noc_keywords_table.php`

Creates `t_noc_keywords` with all columns above. Unique index on `noc_code`.

**Step 1.3** — Create `app/Models/TNocKeyword.php`

```php
class TNocKeyword extends Model {
    protected $table = 't_noc_keywords';
    protected $fillable = ['noc_code', 'noc_title', 'teer_level', 'keywords', 'is_active'];
    protected $casts = ['is_active' => 'boolean', 'teer_level' => 'integer'];

    public function keywordArray(): array {
        return array_values(array_filter(
            array_map('trim', explode(',', strtolower($this->keywords))),
            fn($k) => $k !== ''
        ));
    }
}
```

**Step 1.4** — Update `TJob` model: add new columns to `$fillable` and `$casts`:

```php
// Add to $fillable:
'noc_code', 'teer_level', 'noc_confidence', 'noc_match_layer',
'noc_review_status', 'noc_overridden_by', 'noc_reviewed_at',

// Add to $casts:
'teer_level'     => 'integer',
'noc_confidence' => 'float',
'noc_reviewed_at'=> 'datetime',
```

**Step 1.5** — Create `database/seeders/NocKeywordsSeeder.php`

Seed rows covering the most common job titles in Canadian immigration postings:

| NOC Code | NOC Title | TEER | Sample Keywords |
|----------|-----------|------|----------------|
| 00015 | Senior managers - trade, broadcasting and other services | 0 | `general manager,managing director,executive director,ceo,chief executive` |
| 10010 | Advertising, marketing and PR managers | 0 | `marketing manager,pr manager,advertising manager,brand manager,communications manager` |
| 20010 | Engineering managers | 0 | `engineering manager,head of engineering,director of engineering,vp engineering` |
| 21231 | Software engineers and designers | 1 | `software engineer,software developer,full stack developer,backend developer,frontend developer,web developer,devops engineer` |
| 21232 | Software developers and programmers | 1 | `programmer,application developer,mobile developer,ios developer,android developer` |
| 21222 | Information systems specialists | 1 | `systems analyst,it analyst,business analyst,data analyst,data scientist,machine learning` |
| 31102 | Family medicine physicians | 1 | `physician,family doctor,general practitioner,medical doctor,gp,md ` |
| 31301 | Registered nurses | 1 | `registered nurse,rn,nurse practitioner,np,advanced practice nurse` |
| 11101 | Financial auditors and accountants | 1 | `accountant,auditor,cpa,chartered accountant,financial analyst,controller` |
| 41301 | Lawyers and Quebec notaries | 1 | `lawyer,attorney,legal counsel,solicitor,barrister,notary` |
| 72320 | Welders and related machine operators | 2 | `welder,welding,mig welder,tig welder,stick welder,flux core,welding operator` |
| 72410 | Electricians (except industrial) | 2 | `electrician,electrical journeyman,journeyperson electrician,residential electrician,commercial electrician` |
| 72422 | Plumbers | 2 | `plumber,plumbing journeyman,journeyperson plumber,pipefitter,steamfitter` |
| 72021 | Contractors and supervisors - general construction | 2 | `construction supervisor,site supervisor,foreman,construction foreman,general contractor` |
| 22221 | Computer network technicians | 2 | `network technician,network administrator,it technician,systems administrator,network engineer,sysadmin` |
| 32101 | Licensed practical nurses | 2 | `lpn,licensed practical nurse,registered practical nurse,rpn` |
| 22310 | Electrical and electronics engineering technologists | 2 | `electrical technologist,electronics technologist,electrical technician,instrumentation technician` |
| 63200 | Cooks | 3 | `cook,line cook,prep cook,sous chef,kitchen cook,short order cook` |
| 63210 | Bakers and pastry chefs | 3 | `baker,pastry chef,pastry cook,bread baker,production baker` |
| 32109 | Dental assistants | 3 | `dental assistant,chairside assistant,dental receptionist and assistant` |
| 33102 | Nurse aides, orderlies and patient service associates | 3 | `care aide,health care aide,personal support worker,psw,patient care aide,nursing aide,orderly` |
| 34101 | Early childhood educators and assistants | 3 | `early childhood educator,ece,daycare worker,childcare educator,preschool teacher` |
| 73201 | Heating, refrigeration and air conditioning mechanics | 3 | `hvac technician,hvac mechanic,refrigeration mechanic,heating technician,air conditioning technician` |
| 64100 | Retail salespersons and visual merchandisers | 4 | `retail sales associate,sales associate,retail associate,store associate,visual merchandiser` |
| 44101 | Home support workers | 4 | `home support worker,home care worker,personal care worker,caregiver,live-in caregiver` |
| 75110 | Construction trades helpers and labourers | 4 | `construction labourer,trades helper,general labourer construction,site labourer` |
| 65201 | Food counter attendants | 4 | `food counter attendant,kitchen helper,fast food worker,cafeteria worker` |
| 85100 | Landscaping and grounds maintenance labourers | 5 | `landscaping labourer,grounds maintenance,lawn care worker,groundskeeper` |
| 85101 | Fruit, vegetable and related labourers | 5 | `farm labourer,agricultural worker,fruit picker,crop worker,greenhouse labourer` |
| 74202 | Delivery service drivers and couriers | 5 | `delivery driver,courier,package delivery,parcel delivery,door to door delivery` |

---

### Phase 2 — NocClassificationService

Create `app/Services/NocClassificationService.php`.

**Properties:**
```php
private const CONFIDENCE_AUTO_THRESHOLD   = 0.60;
private const CONFIDENCE_REVIEW_THRESHOLD = 0.30;
```

**TEER signal word map (hardcoded, based on government definitions):**
```php
private const TEER_SIGNALS = [
    0 => ['general manager','managing director','executive director','director of','vp ',
          'vice president','chief executive','chief financial','ceo','cfo','coo','cto',
          'president and','head of operations'],
    1 => ['university degree','bachelor','master\'s','phd','doctorate','engineer p.eng',
          'registered engineer','physician','lawyer','pharmacist','dentist','architect',
          'chartered professional','cpa','software engineer','data scientist'],
    2 => ['college diploma','apprenticeship','journeyman','journeyperson','red seal',
          'technologist','supervisor','supervisory','police officer','firefighter',
          '2-year diploma','3-year diploma','technical diploma'],
    3 => ['on-the-job training','ojt','trade certificate','cook','baker','dental assistant',
          'care aide','support worker','some secondary','6 months experience',
          'less than 2 year','short course','vocational certificate'],
    4 => ['high school diploma','secondary school','secondary education','no degree required',
          'retail','sales associate','cashier','child care provider','home support',
          'several weeks training','entry level no experience'],
    5 => ['no formal education','no education required','no experience required',
          'short demonstration','grounds maintenance','landscaping labourer',
          'farm labourer','delivery driver','door-to-door','entry level no degree'],
];
```

**`classify(TJob $job): void`**

Full method logic:

```
1. Check lock:
   if noc_review_status IN ['admin_confirmed', 'admin_overridden']
       → call recomputePrScoreOnly($job) and return immediately
       → do NOT re-run classification

2. Load all active TNocKeyword rows (cache in memory per process run)

3. Layer 1 — keyword match:
   titleLower = strtolower($job->title)
   foreach $nocKeywords as $noc:
       foreach $noc->keywordArray() as $keyword:
           if str_contains(titleLower, keyword):
               matchedTokenCount++
               bestNoc = $noc (keep highest matchedTokenCount)

   if bestNoc found:
       titleWordCount = count(explode(' ', titleLower))
       confidence = min(1.0, matchedTokenCount / max(1, titleWordCount))
       confidence = max(confidence, 0.40)   // floor: any keyword hit = at least 0.40
       layer = 'keyword'

4. Layer 2 — TEER signal inference (only if Layer 1 found nothing):
   searchText = strtolower($job->title . ' ' . substr($job->description, 0, 500))
   teerHits = [0=>0, 1=>0, 2=>0, 3=>0, 4=>0, 5=>0]
   foreach TEER_SIGNALS as teer => signals:
       foreach signals as signal:
           if str_contains(searchText, signal): teerHits[teer]++
   
   bestTeer = teer with highest hit count (min 1 hit required)
   if bestTeer found:
       hitCount = teerHits[bestTeer]
       confidence = match(hitCount): 1=>0.25, 2=>0.40, 3=>0.55, default=>0.55
       noc_code = null (signal match gives TEER but no specific NOC code)
       teer_level = bestTeer
       layer = 'signal'
   else:
       confidence = 0.00
       layer = 'none'

5. Determine review status:
   if confidence >= CONFIDENCE_AUTO_THRESHOLD:
       status = 'auto_assigned'
   else:
       status = 'flagged_for_review'

6. Update job columns (only NOC columns — never touch posted_at, status, or other fields):
   $job->update([
       'noc_code'          => $nocCode,
       'teer_level'        => $teerLevel,
       'noc_confidence'    => $confidence,
       'noc_match_layer'   => $layer,
       'noc_review_status' => $status,
   ]);
```

**`confirmNoc(TJob $job, string $nocCode, int $adminId): void`**
- Admin confirms the system's suggestion without change
- Sets: `noc_review_status = 'admin_confirmed'`, `noc_overridden_by = $adminId`, `noc_reviewed_at = now()`
- Removes job from review queue

**`overrideNoc(TJob $job, string $nocCode, int $adminId): void`**
- Admin replaces the system's NOC with a different one
- Looks up `teer_level` from `t_noc_keywords` by the new `$nocCode`
- Sets: `noc_code`, `teer_level`, `noc_review_status = 'admin_overridden'`, `noc_overridden_by`, `noc_reviewed_at`
- Sets `noc_confidence = 1.0`, `noc_match_layer = 'admin'`
- Removes job from review queue

**`reclassifyAll(int $chunkSize = 200): void`**
- Re-runs `classify()` on all jobs where `noc_review_status NOT IN ['admin_confirmed', 'admin_overridden']`
- Used by artisan command `noc:reclassify` when admin adds new keyword rows
- Processes in chunks to avoid memory issues

---

### Phase 3 — Hook into `jobs:normalize`

**File:** `app/Console/Commands/Jobs/NormalizedProcessedJobs.php`

Add at top of `handle()` method:
```php
$nocService = app(\App\Services\NocClassificationService::class);
```

In the **update** path, after `$this->syncJobRulesByCategory($job)`:
```php
$nocService->classify($job);
```

In the **create** path, after `$this->syncJobRulesByCategory($job)`:
```php
$nocService->classify($job);
```

The lock check inside `classify()` means existing admin-confirmed jobs are protected automatically — no conditional needed in the command.

---

### Phase 4 — Artisan Command: `noc:reclassify`

Create `app/Console/Commands/NocReclassifyCommand.php`:

```
Signature:  noc:reclassify {--job= : reclassify a single job by ID}
Purpose:    Re-run classification after admin adds/edits keywords in t_noc_keywords
Output:     "Reclassified N jobs. Flagged M for review. Auto-assigned K."
```

Admin uses this after adding new NOC keyword rows to improve coverage on existing jobs.

---

### Phase 5 — Admin Controllers

**`NocKeywordController.php`** — CRUD for `t_noc_keywords`

```php
index()     → paginated list of all NOC rows, searchable by noc_code / teer_level / keyword
store()     → validate + create new row; return redirect with success flash
update()    → validate + update row; return JSON for modal pattern
destroy()   → soft-delete (set is_active = false) — never hard delete active NOC rows
```

Validation rules for `store()` / `update()`:
```php
'noc_code'   => 'required|string|max:10|regex:/^\d{5}$/'  // 5-digit NOC code
'noc_title'  => 'required|string|max:255'
'teer_level' => 'required|integer|between:0,5'
'keywords'   => 'required|string'                          // comma-separated, free text
'is_active'  => 'boolean'
```

**`NocReviewController.php`** — Review queue for flagged jobs

```php
index()     → paginated list of jobs where noc_review_status = 'flagged_for_review'
             → each row shows: job title, location, employer, system guess (noc_code + confidence),
               match layer, "Confirm" button, "Override" dropdown + button
confirm()   → POST: calls NocClassificationService::confirmNoc()
override()  → POST: calls NocClassificationService::overrideNoc() with admin-selected NOC from dropdown
```

The override dropdown is populated from `t_noc_keywords` active rows, grouped by TEER level.

---

### Phase 6 — Admin UI Views

**`noc-keywords/index.blade.php`** — NOC Codes management

Follows exact same pattern as `categories/index.blade.php`:
- DataTable with columns: NOC Code | TEER | Title | Keywords (truncated) | Active | Actions
- "Add NOC Code" button → opens `modals/add.blade.php`
- Edit icon → opens `modals/edit.blade.php` pre-filled
- Deactivate toggle (no hard delete)
- TEER badge colored by level: 0=dark, 1=primary, 2=success, 3=info, 4=warning, 5=secondary

```
┌──────────┬──────┬──────────────────────────────────┬────────────────────────────┬────────┬─────────┐
│ NOC Code │ TEER │ Title                            │ Keywords                   │ Active │ Actions │
├──────────┼──────┼──────────────────────────────────┼────────────────────────────┼────────┼─────────┤
│ 72320    │  2   │ Welders and related operators    │ welder, mig welder, tig... │  ✅   │ Edit    │
│ 21231    │  1   │ Software engineers and designers │ software engineer, full... │  ✅   │ Edit    │
│ 64100    │  4   │ Retail salespersons              │ retail sales, sales asso.. │  ✅   │ Edit    │
└──────────┴──────┴──────────────────────────────────┴────────────────────────────┴────────┴─────────┘
```

**`modals/add.blade.php` and `modals/edit.blade.php`**

Fields:
- NOC Code (text, 5 digits)
- TEER Level (select 0–5, each option shows the government definition as hint text)
- NOC Title (text)
- Keywords (textarea — comma-separated, with helper: "Enter keywords that appear in job titles, separated by commas")
- Active (checkbox, default checked)

**`noc-review/index.blade.php`** — NOC Review Queue

Follows same pattern as `jobs-panel/review/index.blade.php` (existing Job Review):
- Badge in sidebar shows count of pending
- Table: Job Title | Employer | Location | System Guess | TEER | Confidence | Layer | Actions

```
┌───────────────────────────┬──────────────────┬──────────────┬──────────────┬──────┬────────────┬─────────┬──────────────────────┐
│ Job Title                 │ Employer         │ Location     │ System Guess │ TEER │ Confidence │ Layer   │ Actions              │
├───────────────────────────┼──────────────────┼──────────────┼──────────────┼──────┼────────────┼─────────┼──────────────────────┤
│ Senior MIG Welder         │ Northern Steel   │ North Bay ON │ NOC 72320    │  2   │ 40%        │ keyword │ [Confirm] [Override▼]│
│ Production Lead           │ Maple Foods      │ Moncton NB   │ (no match)   │  —   │ 0%         │ none    │ [Override▼]          │
│ Registered Nurse - ICU    │ Atlantic Health  │ Halifax NS   │ NOC 31301    │  1   │ 75%        │ keyword │ [Confirm] [Override▼]│
└───────────────────────────┴──────────────────┴──────────────┴──────────────┴──────┴────────────┴─────────┴──────────────────────┘
```

"Override" opens an inline dropdown: admin selects correct NOC from a grouped list (grouped by TEER 0→5), then clicks "Apply Override".

---

### Phase 7 — Sidebar + Routes

**Sidebar changes** (`layouts/app.blade.php`):

Under **Jobs Aggregation** accordion — add NOC Review Queue item with badge (same pattern as Job Review):
```php
<li class="list-group-item bg-light border-0">
    <a href="{{ route('admin.jobs-panel.noc-review.index') }}"
       class="text-decoration-none d-flex align-items-center justify-content-between">
        <span><i class="bi bi-patch-question me-2"></i> NOC Review</span>
        @php $nocPendingCount = \App\Models\TJob::where('noc_review_status','flagged_for_review')->count(); @endphp
        <span class="badge bg-warning text-dark {{ $nocPendingCount === 0 ? 'd-none' : '' }}">
            {{ $nocPendingCount }}
        </span>
    </a>
</li>
```

Under **Taxonomy** accordion — add NOC Codes item:
```php
<li class="list-group-item bg-light border-0">
    <a href="{{ route('admin.jobs-panel.noc-keywords.index') }}" class="text-decoration-none">
        <i class="bi bi-bookmark-check me-2"></i> NOC Codes
    </a>
</li>
```

**Routes** (`web.php`) — inside the `admin/jobs-panel` group:

```php
use App\Http\Controllers\NocKeywordController;
use App\Http\Controllers\NocReviewController;

// Taxonomy: NOC Codes CRUD
Route::prefix('noc-keywords')->name('noc-keywords.')->group(function () {
    Route::get('/',          [NocKeywordController::class, 'index'])->name('index');
    Route::post('/',         [NocKeywordController::class, 'store'])->name('store');
    Route::put('/{id}',      [NocKeywordController::class, 'update'])->name('update');
    Route::delete('/{id}',   [NocKeywordController::class, 'destroy'])->name('destroy');
});

// Jobs Aggregation: NOC Review Queue
Route::prefix('noc-review')->name('noc-review.')->group(function () {
    Route::get('/',              [NocReviewController::class, 'index'])->name('index');
    Route::post('/{id}/confirm', [NocReviewController::class, 'confirm'])->name('confirm');
    Route::post('/{id}/override',[NocReviewController::class, 'override'])->name('override');
});
```

---

## Data Flow Summary

```
Scraper → t_jobs_scraped_raw (raw_json)
    ↓
php artisan jobs:normalize (NormalizedProcessedJobs.php)
    ↓ per job (create or update):
    1. keyword filtering (existing)
    2. TJob::create() or $job->update() (existing)
    3. syncJobRulesByCategory($job) (existing)
    4. NocClassificationService::classify($job)   ← NEW
       ├─ locked? (admin_confirmed / admin_overridden) → skip, recompute PR score only
       ├─ Layer 1: title keyword → t_noc_keywords lookup
       │   ├─ confidence ≥ 0.60 → auto_assigned
       │   └─ confidence < 0.60 → flagged_for_review
       ├─ Layer 2 fallback: TEER signal inference from title+description
       │   ├─ hits ≥ 3 → confidence 0.55 → flagged_for_review
       │   └─ no hits → confidence 0.00 → flagged_for_review (noc_code = null)
       └─ writes: noc_code, teer_level, noc_confidence, noc_match_layer, noc_review_status
    ↓
Admin: NOC Review Queue (noc_review_status = 'flagged_for_review')
    → Confirm (system was right) → admin_confirmed, locked
    → Override (pick different NOC) → admin_overridden, locked
    ↓
Downstream: EmployerIntelligenceService reads noc_code + teer_level from t_jobs
```

---

## Confidence Score Explained

| Scenario | Layer | Confidence | Action |
|----------|-------|-----------|--------|
| "Senior MIG Welder Alberta" → "mig welder" matches NOC 72320 | keyword | 0.67 | Auto-assigned |
| "Welder" → "welder" matches, title is one word | keyword | 0.50 (floored to 0.40, capped at 1 word) | Flagged |
| "Production Lead" → no keyword match, "supervisor" signal hit (TEER 2) | signal | 0.25 | Flagged |
| "General Helper" → no keyword, no signal | none | 0.00 | Flagged, noc_code = null |
| Title has "software engineer" (2 words matching exact keyword) in a 5-word title | keyword | 0.40+ | Flagged or auto depending on match ratio |

---

## Verification Checklist

- [ ] Migration adds all 7 NOC columns to `t_jobs` with correct types and indexes
- [ ] Migration creates `t_noc_keywords` with unique index on `noc_code`
- [ ] `NocKeywordsSeeder` inserts all 30 seed rows without error
- [ ] Job with title "MIG Welder" maps to NOC 72320, TEER 2, confidence ≥ 0.60, status `auto_assigned`
- [ ] Job with title "Welder" (single word) maps to NOC 72320, TEER 2, confidence < 0.60, status `flagged_for_review`
- [ ] Job with no keyword match but "supervisor" in title → TEER 2 via signal, confidence ~0.25, status `flagged_for_review`
- [ ] Job with no keyword and no signal → `noc_code = null`, confidence 0.00, status `flagged_for_review`
- [ ] Admin-confirmed job: re-running `jobs:normalize` does NOT overwrite the NOC assignment
- [ ] Admin-overridden job: re-running `jobs:normalize` does NOT overwrite the NOC assignment
- [ ] NOC Codes admin page (Taxonomy) lists all seed rows with correct TEER badges
- [ ] Admin can add a new NOC row via modal form
- [ ] Admin can edit keywords on an existing NOC row
- [ ] Admin deactivating a NOC row stops it from being used in future classifications
- [ ] NOC Review Queue shows all `flagged_for_review` jobs with system guess and confidence
- [ ] Badge counter in sidebar shows correct count of pending NOC reviews
- [ ] Badge disappears when count = 0
- [ ] Admin "Confirm" button on a review queue row → sets `admin_confirmed`, job disappears from queue
- [ ] Admin "Override" with a different NOC → sets `admin_overridden`, correct NOC and TEER stored
- [ ] `noc:reclassify` command re-runs classification on unlocked jobs only
- [ ] `noc:reclassify --job=42` reclassifies only job ID 42
- [ ] After admin adds new keywords and runs `noc:reclassify`, previously unmatched jobs now match
- [ ] `noc_overridden_by` stores the correct logged-in admin user ID
- [ ] Deactivating a NOC row in the admin panel does not affect jobs already classified with that NOC

---

## Do NOT

- Do not reclassify jobs with `noc_review_status IN ['admin_confirmed', 'admin_overridden']` — ever
- Do not assign multiple NOC codes to one job — one job, one NOC, enforced by a single varchar column
- Do not hard-delete rows from `t_noc_keywords` — deactivate with `is_active = false`
- Do not run NOC classification on `store()` via the web admin panel — only during `jobs:normalize`
- Do not call an external API for NOC lookups — classification is fully offline, using `t_noc_keywords` only
- Do not assign a TEER level without a NOC code when Layer 1 matched — always take the NOC from the matched row
- Do not show `noc_confidence` as a raw float to end users — only admins see this in the review queue
- Do not touch `posted_at`, `status`, or any non-NOC fields during classification

---

## File Reference Summary

| File | Status |
|------|--------|
| `database/migrations/2026_04_XX_000001_add_noc_columns_to_t_jobs_table.php` | CREATE |
| `database/migrations/2026_04_XX_000002_create_t_noc_keywords_table.php` | CREATE |
| `database/seeders/NocKeywordsSeeder.php` | CREATE |
| `app/Models/TNocKeyword.php` | CREATE |
| `app/Services/NocClassificationService.php` | CREATE |
| `app/Console/Commands/NocReclassifyCommand.php` | CREATE |
| `app/Http/Controllers/NocKeywordController.php` | CREATE |
| `app/Http/Controllers/NocReviewController.php` | CREATE |
| `resources/views/admin/jobs-panel/noc-keywords/index.blade.php` | CREATE |
| `resources/views/admin/jobs-panel/noc-keywords/modals/add.blade.php` | CREATE |
| `resources/views/admin/jobs-panel/noc-keywords/modals/edit.blade.php` | CREATE |
| `resources/views/admin/jobs-panel/noc-review/index.blade.php` | CREATE |
| `app/Console/Commands/Jobs/NormalizedProcessedJobs.php` | MODIFY (add classify() call) |
| `app/Models/TJob.php` | MODIFY (add new columns to fillable + casts) |
| `resources/views/admin/jobs-panel/layouts/app.blade.php` | MODIFY (add 2 sidebar items) |
| `routes/web.php` | MODIFY (add noc-keywords + noc-review routes) |
