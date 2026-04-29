# Job Search Aggregator — Implementation Guide

> **Purpose:** This document describes every PHP file, database table, migration, route, model, controller, view, and JavaScript change made for the **Job Search Criteria / Job Search Aggregator** feature. Another developer or AI agent can use this document to recreate the same changes from scratch in a clean codebase.

---

## Table of Contents

1. [Feature Overview](#1-feature-overview)
2. [Database Schema](#2-database-schema)
   - [job_search_criteria](#21-table-job_search_criteria-legacy)
   - [t_job_search_aggregator](#22-table-t_job_search_aggregator-primary)
   - [t_categories — added columns](#23-table-t_categories-added-columns)
   - [t_aggregators — added column](#24-table-t_aggregators-added-column)
   - [t_jobs_scraped_raw — added columns](#25-table-t_jobs_scraped_raw-added-columns)
3. [Migrations (in order)](#3-migrations-in-order)
4. [Models](#4-models)
5. [Controller](#5-controller)
6. [Routes](#6-routes)
7. [Blade Views](#7-blade-views)
8. [JavaScript](#8-javascript)
9. [Key Business Rules](#9-key-business-rules)
10. [Data Flow](#10-data-flow)

---

## 1. Feature Overview

The feature allows admins to define **Job Search Aggregators** — named configurations that tell the scraping pipeline which job titles to search for, on which platforms, in which location, and filtered to specific job categories. When an aggregator is saved, a linked `t_jobs_scraped_raw` record is automatically created (marked `is_dynamic = true`) to trigger the scraping pipeline.

The UI lives at:

```
/admin/jobs-panel/job-search-aggregator
```

Route name prefix: `admin.jobs-panel.job-search-aggregator.*`

---

## 2. Database Schema

### 2.1 Table: `job_search_criteria` (Legacy)

> Created in the first migration as a simpler predecessor. Kept for backwards compatibility but superseded by `t_job_search_aggregator`.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | BIGINT UNSIGNED | No | Primary key, auto-increment |
| `job_title_keywords` | JSON | Yes | Array of keyword strings (migrated from `job_title` string) |
| `location` | VARCHAR(512) | No | Geographic location |
| `platforms` | JSON | Yes | Array of strings: `['linkedin', 'indeed']` (migrated from `platform` string) |
| `created_at` | TIMESTAMP | No | Auto-managed by Laravel |
| `updated_at` | TIMESTAMP | No | Auto-managed by Laravel |

**MySQL index:**
```sql
ALTER TABLE `job_search_criteria`
  ADD INDEX `job_search_criteria_location_idx` (`location`(191));
```

---

### 2.2 Table: `t_job_search_aggregator` (Primary)

> This is the main table used by the current UI and controller.

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `id` | BIGINT UNSIGNED | No | Primary key, auto-increment |
| `aggregator_name` | VARCHAR(255) | No | Human-readable name for the aggregator |
| `job_title_keywords` | JSON | Yes | Array of keyword strings |
| `location` | VARCHAR(512) | Yes | Geographic location |
| `platforms` | JSON | Yes | Array: `['linkedin', 'indeed']` |
| `exclude_keywords` | LONGTEXT | Yes | Comma-separated keywords to exclude from results |
| `include_keywords` | LONGTEXT | Yes | Comma-separated keywords that must be present |
| `t_category_ids` | JSON | Yes | Array of integer IDs referencing `t_categories.id` |
| `last_processed` | TIMESTAMP | Yes | When the aggregator was last processed |
| `last_imported` | TIMESTAMP | Yes | When jobs were last imported |
| `last_run` | TIMESTAMP | Yes | When the aggregator last ran |
| `status` | VARCHAR(255) | Yes | Current processing status |
| `created_at` | TIMESTAMP | No | Auto-managed by Laravel |
| `updated_at` | TIMESTAMP | No | Auto-managed by Laravel |

> **Note:** An older single-FK column `t_category_id` was added in migration `2026_04_17` and then replaced by the JSON array `t_category_ids` in migration `2026_04_18`. The old column no longer exists in the final schema.

---

### 2.3 Table: `t_categories` — Added Columns

Two columns were added to the existing `t_categories` table:

| Column | Type | Nullable | Inserted After | Notes |
|---|---|---|---|---|
| `include_keywords` | LONGTEXT | Yes | `name` | Comma-separated keywords a job must contain to belong to this category |
| `exclude_keywords` | LONGTEXT | Yes | `include_keywords` | Comma-separated keywords that disqualify a job from this category |

---

### 2.4 Table: `t_aggregators` — Added Column

One column was added to the existing `t_aggregators` table, and the old `t_category_id` FK column was removed:

| Change | Column | Type | Notes |
|---|---|---|---|
| Added | `t_category_ids` | JSON, nullable | Array of category IDs; replaces `t_category_id` |
| Removed | `t_category_id` | BIGINT UNSIGNED | FK was dropped; data migrated: `t_category_id → [t_category_id]` |

---

### 2.5 Table: `t_jobs_scraped_raw` — Added Columns

| Column | Type | Nullable | Notes |
|---|---|---|---|
| `is_dynamic` | BOOLEAN | No, default `false` | `true` when this record was triggered by a `TJobSearchAggregator` |
| `t_job_search_aggregator_id` | BIGINT UNSIGNED | Yes | FK → `t_job_search_aggregator.id`, set when `is_dynamic = true` |

---

## 3. Migrations (in order)

All files are in `database/migrations/`.

| File | What it does |
|---|---|
| `2026_04_13_000000_create_job_search_criteria_table.php` | Creates `job_search_criteria` with `job_title` (string) + `location`. Adds prefix index on MySQL. |
| `2026_04_14_000000_add_platform_to_job_search_criteria_table.php` | Adds `platform` (string) column to `job_search_criteria`. |
| `2026_04_14_000100_convert_platform_to_platforms_in_job_search_criteria_table.php` | Renames `platform` → `platforms` (JSON array), migrates existing data. |
| `2026_04_15_000000_add_job_title_keywords_to_job_search_criteria_table.php` | Adds `job_title_keywords` (JSON), copies data from `job_title`, drops `job_title`. |
| `2026_04_16_000000_add_include_exclude_keywords_to_t_categories_table.php` | Adds `include_keywords` + `exclude_keywords` (LONGTEXT nullable) to `t_categories`. |
| `2026_04_17_000000_create_or_align_t_job_search_aggregator_table.php` | Creates `t_job_search_aggregator` (or adds missing columns if it already exists). Adds single `t_category_id` FK at this point. |
| `2026_04_18_000000_add_job_search_fields_to_t_job_search_aggregator_table.php` | Adds `job_title_keywords`, `location`, `platforms`, `t_category_ids` (JSON) to `t_job_search_aggregator`. Migrates `t_category_id → t_category_ids`, then drops `t_category_id`. |
| `2026_04_19_000000_convert_t_category_id_to_t_category_ids_in_t_aggregators.php` | Same pattern for `t_aggregators`: adds `t_category_ids` JSON, migrates data, drops `t_category_id` FK. |

> **Note:** Migrations `2026_04_20_*` and later (for `t_jobs`, NOC, PR scoring) are **not** part of the job search criteria feature — they are separate features.

---

## 4. Models

### 4.1 `App\Models\JobSearchCriterion`

**File:** `app/Models/JobSearchCriterion.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class JobSearchCriterion extends Model
{
    use HasFactory;

    protected $table = 'job_search_criteria';

    protected $fillable = [
        'job_title_keywords',
        'location',
        'platforms',
    ];

    protected $casts = [
        'job_title_keywords' => 'array',
        'platforms'          => 'array',
    ];
}
```

---

### 4.2 `App\Models\TJobSearchAggregator`

**File:** `app/Models/TJobSearchAggregator.php`

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class TJobSearchAggregator extends Model
{
    use HasFactory;

    protected $table = 't_job_search_aggregator';

    protected $fillable = [
        'aggregator_name',
        'job_title_keywords',
        'location',
        'platforms',
        'exclude_keywords',
        'include_keywords',
        'last_processed',
        'last_imported',
        'status',
        'last_run',
        't_category_ids',
    ];

    protected $casts = [
        'job_title_keywords' => 'array',
        'platforms'          => 'array',
        't_category_ids'     => 'array',
        'last_processed'     => 'datetime',
        'last_imported'      => 'datetime',
        'last_run'           => 'datetime',
    ];
}
```

---

### 4.3 `App\Models\TJobScrapedRaw` — Modified

**File:** `app/Models/TJobScrapedRaw.php`

Add the following to `$fillable`:

```php
'is_dynamic',
't_job_search_aggregator_id',
```

Add to `$casts`:

```php
'is_dynamic' => 'boolean',
```

Add relationship method:

```php
public function jobSearchAggregator()
{
    return $this->belongsTo(TJobSearchAggregator::class, 't_job_search_aggregator_id');
}
```

---

## 5. Controller

**File:** `app/Http/Controllers/JobSearchAggregatorController.php`

```php
<?php

namespace App\Http\Controllers;

use App\Models\TCategory;
use App\Models\TJobScrapedRaw;
use App\Models\TJobSearchAggregator;
use Exception;
use Illuminate\Http\Request;

class JobSearchAggregatorController extends Controller
{
    public function index()
    {
        return view('admin.jobs-panel.job-search-criteria.index');
    }

    public function create()
    {
        $categories = TCategory::query()->orderBy('name')->get(['id', 'name']);

        return view('admin.jobs-panel.job-search-criteria.create', compact('categories'));
    }

    public function edit($id)
    {
        $aggregator = TJobSearchAggregator::findOrFail($id);
        $categories = TCategory::query()->orderBy('name')->get(['id', 'name']);

        return view('admin.jobs-panel.job-search-criteria.edit', compact('aggregator', 'categories'));
    }

    public function data(Request $request)
    {
        $query = TJobSearchAggregator::query();

        $orderableColumns = [
            'id'             => 'id',
            'aggregator_name' => 'aggregator_name',
            'location'       => 'location',
            'created_at'     => 'created_at',
        ];

        return datatables()
            ->eloquent($query)
            ->addColumn('category_names', function (TJobSearchAggregator $row) {
                $ids = collect((array) ($row->t_category_ids ?? []))
                    ->map(fn ($v) => (int) $v)
                    ->filter()
                    ->values()
                    ->all();

                if (empty($ids)) {
                    return [];
                }

                return TCategory::query()
                    ->whereIn('id', $ids)
                    ->orderBy('name')
                    ->pluck('name')
                    ->values()
                    ->all();
            })
            ->filter(function ($q) use ($request) {
                $globalSearch = strtolower(htmlentities($request->search['value'] ?? '', ENT_QUOTES, 'UTF-8'));

                if ($globalSearch === '') {
                    return;
                }

                $q->where(function ($sub) use ($globalSearch) {
                    $sub->whereRaw('CAST(id AS CHAR) LIKE ?', ["%{$globalSearch}%"])
                        ->orWhereRaw('LOWER(aggregator_name) LIKE ?', ["%{$globalSearch}%"])
                        ->orWhereRaw('LOWER(location) LIKE ?', ["%{$globalSearch}%"])
                        ->orWhereRaw('LOWER(CAST(job_title_keywords AS CHAR)) LIKE ?', ["%{$globalSearch}%"])
                        ->orWhereRaw('LOWER(CAST(platforms AS CHAR)) LIKE ?', ["%{$globalSearch}%"])
                        ->orWhereRaw('LOWER(CAST(t_category_ids AS CHAR)) LIKE ?', ["%{$globalSearch}%"]);
                });
            })
            ->order(function ($q) use ($request, $orderableColumns) {
                if (! empty($request->order)) {
                    $colIndex = $request->order[0]['column'];
                    $dir      = $request->order[0]['dir'];
                    $colName  = $request->columns[$colIndex]['data'] ?? null;

                    if ($colName && isset($orderableColumns[$colName])) {
                        $q->orderBy($orderableColumns[$colName], $dir);
                    }
                }
            })
            ->make(true);
    }

    public function store(Request $request)
    {
        $keywords = $this->extractKeywordsFromRequest($request);

        $validated = $request->validate([
            'aggregator_name'    => 'required|string|max:255',
            'location'           => 'required|string|max:512',
            'exclude_keywords'   => 'nullable|string',
            'platforms'          => 'required|array|min:1',
            'platforms.*'        => 'required|in:linkedin,indeed',
            't_category_ids'     => 'required|array|min:1',
            't_category_ids.*'   => 'required|integer|exists:t_categories,id',
        ]);

        try {
            if (count($keywords) === 0) {
                $message = 'Add at least one job title keyword.';
                if ($request->expectsJson() || $request->ajax()) {
                    return response()->json([
                        'message' => $message,
                        'errors'  => ['job_title_keywords' => ['Add at least one keyword.']],
                    ], 422);
                }
                return back()->withInput()->withErrors(['job_title_keywords_csv' => $message]);
            }

            $row = TJobSearchAggregator::create([
                'aggregator_name'  => trim($validated['aggregator_name']),
                'job_title_keywords' => $keywords,
                'location'         => trim($validated['location']),
                'exclude_keywords' => $this->normalizeOptionalText($validated['exclude_keywords'] ?? null),
                'platforms'        => collect($validated['platforms'])
                    ->map(fn ($v) => strtolower(trim($v)))
                    ->unique()->values()->all(),
                't_category_ids'   => collect($validated['t_category_ids'])
                    ->map(fn ($v) => (int) $v)
                    ->filter()->unique()->values()->all(),
            ]);

            // Create a dynamic scrape record linked to this aggregator
            TJobScrapedRaw::create([
                'is_dynamic'                 => true,
                't_job_search_aggregator_id' => $row->id,
                't_aggregator_id'            => null,
                'url'                        => null,
                'process_status'             => 'pending',
                'is_processed'               => false,
            ]);

            if ($request->expectsJson() || $request->ajax()) {
                return response()->json(['success' => 'Job search aggregator saved successfully.', 'data' => $row], 201);
            }

            return redirect()
                ->route('admin.jobs-panel.job-search-aggregator.index')
                ->with('success', 'Job search aggregator saved successfully.');
        } catch (Exception $e) {
            if ($request->expectsJson() || $request->ajax()) {
                return response()->json(['error' => $e->getMessage()], 500);
            }
            return back()->withInput()->with('error', $e->getMessage());
        }
    }

    public function update(Request $request, $id)
    {
        $keywords = $this->extractKeywordsFromRequest($request);

        $validated = $request->validate([
            'aggregator_name'    => 'required|string|max:255',
            'location'           => 'required|string|max:512',
            'exclude_keywords'   => 'nullable|string',
            'platforms'          => 'required|array|min:1',
            'platforms.*'        => 'required|in:linkedin,indeed',
            't_category_ids'     => 'required|array|min:1',
            't_category_ids.*'   => 'required|integer|exists:t_categories,id',
        ]);

        try {
            if (count($keywords) === 0) {
                $message = 'Add at least one job title keyword.';
                if ($request->expectsJson() || $request->ajax()) {
                    return response()->json([
                        'message' => $message,
                        'errors'  => ['job_title_keywords' => ['Add at least one keyword.']],
                    ], 422);
                }
                return back()->withInput()->withErrors(['job_title_keywords_csv' => $message]);
            }

            $row = TJobSearchAggregator::findOrFail($id);
            $row->update([
                'aggregator_name'  => trim($validated['aggregator_name']),
                'job_title_keywords' => $keywords,
                'location'         => trim($validated['location']),
                'exclude_keywords' => $this->normalizeOptionalText($validated['exclude_keywords'] ?? null),
                'platforms'        => collect($validated['platforms'])
                    ->map(fn ($v) => strtolower(trim($v)))
                    ->unique()->values()->all(),
                't_category_ids'   => collect($validated['t_category_ids'])
                    ->map(fn ($v) => (int) $v)
                    ->filter()->unique()->values()->all(),
            ]);

            if ($request->expectsJson() || $request->ajax()) {
                return response()->json(['success' => 'Job search aggregator updated successfully.', 'data' => $row], 200);
            }

            return redirect()
                ->route('admin.jobs-panel.job-search-aggregator.index')
                ->with('success', 'Job search aggregator updated successfully.');
        } catch (Exception $e) {
            if ($request->expectsJson() || $request->ajax()) {
                return response()->json(['error' => $e->getMessage()], 500);
            }
            return back()->withInput()->with('error', $e->getMessage());
        }
    }

    public function destroy($id)
    {
        try {
            TJobSearchAggregator::findOrFail($id)->delete();
            return response()->json(['success' => 'Job search aggregator deleted successfully.'], 200);
        } catch (Exception $e) {
            return response()->json(['error' => $e->getMessage()], 500);
        }
    }

    // --- Private helpers ---

    private function normalizeJobTitleKeywords(array $keywords): array
    {
        return collect($keywords)
            ->map(fn ($k) => trim((string) $k))
            ->filter(fn ($k) => $k !== '')
            ->unique()->values()->all();
    }

    private function normalizeOptionalText(?string $value): ?string
    {
        if ($value === null) return null;
        $trimmed = trim($value);
        return $trimmed === '' ? null : $trimmed;
    }

    /**
     * Supports both JSON-style array payload (API) and CSV textarea input (form).
     * Form field name: job_title_keywords_csv
     * API field name:  job_title_keywords (array)
     */
    private function extractKeywordsFromRequest(Request $request): array
    {
        $arr = $request->input('job_title_keywords');
        if (is_array($arr)) {
            return $this->normalizeJobTitleKeywords($arr);
        }

        $csv = (string) $request->input('job_title_keywords_csv', '');
        if ($csv === '') {
            return [];
        }

        $tokens = preg_split('/[\r\n,]+/', $csv) ?: [];
        return $this->normalizeJobTitleKeywords($tokens);
    }
}
```

---

## 6. Routes

**File:** `routes/web.php`

Add the following inside the `Route::middleware([EnsureUserIsAdmin::class])->group(...)` block, inside the `Route::prefix('admin/jobs-panel')->name('admin.jobs-panel.')->group(...)` block.

Also add the import at the top of `web.php`:
```php
use App\Http\Controllers\JobSearchAggregatorController;
```

Route definitions:

```php
// Job Search Aggregator
Route::get('job-search-aggregator', [JobSearchAggregatorController::class, 'index'])
    ->name('job-search-aggregator.index');
Route::get('job-search-aggregator/create', [JobSearchAggregatorController::class, 'create'])
    ->name('job-search-aggregator.create');
Route::get('job-search-aggregator/{id}/edit', [JobSearchAggregatorController::class, 'edit'])
    ->name('job-search-aggregator.edit');
Route::get('job-search-aggregator/data', [JobSearchAggregatorController::class, 'data'])
    ->name('job-search-aggregator.data');
Route::post('job-search-aggregator', [JobSearchAggregatorController::class, 'store'])
    ->name('job-search-aggregator.store');
Route::put('job-search-aggregator/{id}', [JobSearchAggregatorController::class, 'update'])
    ->name('job-search-aggregator.update');
Route::delete('job-search-aggregator/{id}', [JobSearchAggregatorController::class, 'destroy'])
    ->name('job-search-aggregator.destroy');
```

**Named routes (fully qualified):**

| Name | Method | URL |
|---|---|---|
| `admin.jobs-panel.job-search-aggregator.index` | GET | `/admin/jobs-panel/job-search-aggregator` |
| `admin.jobs-panel.job-search-aggregator.create` | GET | `/admin/jobs-panel/job-search-aggregator/create` |
| `admin.jobs-panel.job-search-aggregator.edit` | GET | `/admin/jobs-panel/job-search-aggregator/{id}/edit` |
| `admin.jobs-panel.job-search-aggregator.data` | GET | `/admin/jobs-panel/job-search-aggregator/data` |
| `admin.jobs-panel.job-search-aggregator.store` | POST | `/admin/jobs-panel/job-search-aggregator` |
| `admin.jobs-panel.job-search-aggregator.update` | PUT | `/admin/jobs-panel/job-search-aggregator/{id}` |
| `admin.jobs-panel.job-search-aggregator.destroy` | DELETE | `/admin/jobs-panel/job-search-aggregator/{id}` |

---

## 7. Blade Views

All views extend `admin.jobs-panel.layouts.app`.

**Directory:** `resources/views/admin/jobs-panel/job-search-criteria/`

### 7.1 `index.blade.php`

- Displays an empty `<table id="jobSearchCriteriaTable">` populated by DataTables via AJAX.
- Columns: ID, Aggregator Name, Job title/keywords, Location, Platforms, Categories, Created, Actions.
- "Add aggregator" button links to `admin.jobs-panel.job-search-aggregator.create`.
- Shows session flash `success` and `error` alerts.
- Injects `window.JOB_SEARCH_AGGREGATOR_URLS` JS object with `data`, `store`, `update`, `delete`, `edit` URL strings.
- Loads `js/jobs-panel/crud/crudEvents.js` then `js/jobs-panel/job-search-criteria/index.js`.

### 7.2 `create.blade.php`

Form fields (all POST to `admin.jobs-panel.job-search-aggregator.store`):

| Field name | HTML element | Required | Notes |
|---|---|---|---|
| `aggregator_name` | `<input type="text">` | Yes | max 255 |
| `job_title_keywords_csv` | `<textarea>` | Yes | Comma-separated; parsed server-side |
| `location` | `<input type="text">` | Yes | max 512 |
| `t_category_ids[]` | `<select multiple>` + Select2 | Yes | Options from `$categories` |
| `exclude_keywords` | `<textarea>` | No | Comma-separated |
| `platforms[]` | Checkboxes | Yes | `linkedin` and/or `indeed` |

Select2 initialized via `$(".select2").select2({ width: "100%", placeholder: "Select one or more categories", allowClear: true })`.

### 7.3 `edit.blade.php`

Same fields as `create.blade.php`, but:
- Form method is `PUT` via `@method('PUT')`.
- Posts to `admin.jobs-panel.job-search-aggregator.update` with `$aggregator->id`.
- Pre-populates values from the `$aggregator` model.
- `job_title_keywords_csv` is pre-filled with `implode(', ', $aggregator->job_title_keywords)`.
- Category `<option>` tags are `@selected` by checking `$selectedCategories->contains((int) $category->id)`.

---

## 8. JavaScript

**File:** `public/js/jobs-panel/job-search-criteria/index.js`

Initializes a DataTable on `#jobSearchCriteriaTable` using the shared `initializeDataTable()` helper and binds delete events using `bindCrudEvents()`.

**Columns:**

| `data` key | Orderable | Notes |
|---|---|---|
| `id` | Yes | — |
| `aggregator_name` | Yes | — |
| `job_title_keywords` | No | Renders each keyword as a `<span class="badge bg-light text-dark border">` |
| `location` | Yes | — |
| `platforms` | No | Maps `linkedin` → "LinkedIn", `indeed` → "Indeed"; joins with `, ` |
| `category_names` | No, not searchable | Renders each category name as a badge; data comes from server-side `addColumn('category_names', ...)` |
| `created_at` | Yes | Formatted with `formatDateOnlyForDisplay(data)` |
| Actions | No | Dropdown with Edit (link to `/{id}/edit`) and Delete button (`.delete-job-search-criterion`) |

**Delete binding:**
```js
bindCrudEvents({
    delete: {
        buttonClass: '.delete-job-search-criterion',
        url: (id) => window.JOB_SEARCH_AGGREGATOR_URLS.delete + id,
        message: 'job search aggregator',
    },
}, table);
```

---

## 9. Key Business Rules

1. **Keywords are required.** At least one job title keyword must be provided. The server validates this separately from Laravel's standard validation (keywords come from a CSV textarea, not a structured array).

2. **Keyword input accepts two formats:**
   - JSON array (`job_title_keywords`) — for API/AJAX use
   - Comma-or-newline-separated text (`job_title_keywords_csv`) — for the HTML form

3. **On `store`, a `t_jobs_scraped_raw` record is auto-created** with `is_dynamic = true` and `t_job_search_aggregator_id = $row->id`. This signals the scraping pipeline to start a dynamic search. `process_status` is set to `'pending'`.

4. **Platforms are normalized** to lowercase before saving. Allowed values: `linkedin`, `indeed`.

5. **Category IDs are stored as a JSON integer array** in `t_category_ids`. The `category_names` column is computed on-the-fly in the DataTables `data()` method by joining with `t_categories`.

6. **`exclude_keywords`** is stored as plain text (comma-separated string). Empty strings are normalized to `null`.

7. **Global search** in DataTables searches across: `id`, `aggregator_name`, `location`, `job_title_keywords` (JSON cast to char), `platforms` (JSON cast to char), `t_category_ids` (JSON cast to char).

---

## 10. Data Flow

```
Admin fills form (create.blade.php)
    │
    ▼
POST /admin/jobs-panel/job-search-aggregator
    │
    ▼
JobSearchAggregatorController::store()
    ├── Validates fields
    ├── Extracts + normalizes keywords from CSV textarea
    ├── Creates TJobSearchAggregator record
    └── Creates TJobScrapedRaw { is_dynamic: true, process_status: 'pending' }
                                        │
                                        ▼
                           Scraping pipeline picks up pending
                           is_dynamic records, reads aggregator config,
                           searches platforms, imports jobs
```

```
Table relationships:

t_job_search_aggregator
  ├── t_category_ids (JSON) → t_categories.id (array FK, no DB-level constraint)
  └── hasMany t_jobs_scraped_raw (via t_job_search_aggregator_id)

t_jobs_scraped_raw
  ├── t_aggregator_id → t_aggregators.id (manual scraping)
  └── t_job_search_aggregator_id → t_job_search_aggregator.id (dynamic scraping)

t_categories
  ├── include_keywords (LONGTEXT)
  └── exclude_keywords (LONGTEXT)

t_aggregators
  └── t_category_ids (JSON) → t_categories.id (replaced single t_category_id FK)
```
