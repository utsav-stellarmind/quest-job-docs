# Implementation Guide — Commits 7fb81fa & ad22bde

> **Scope:** Changes introduced in commits `7fb81fa` (Apr 13 2026) and `ad22bde` (Apr 14 2026),
> **excluding** the Admin Monitoring Dashboard feature and the Job Search Aggregator / Job Search Criteria
> feature (covered separately in `job-search-criteria-implementation.md`).
>
> The remaining work falls into four areas:
> 1. **Categories — include/exclude keywords** (the primary feature in these commits)
> 2. **`uiHelpers.js` — `formatDateOnlyForDisplay()` helper**
> 3. **Python parser — local database config**
> 4. **Admin sidebar — navigation link updates**

---

## Table of Contents

1. [Categories — Include / Exclude Keywords](#1-categories--include--exclude-keywords)
   - [Database schema change](#11-database-schema-change)
   - [Migration](#12-migration)
   - [Model — TCategory](#13-model--tcategory)
   - [Controller — CategoryController](#14-controller--categorycontroller)
   - [Blade views](#15-blade-views)
   - [JavaScript — categories/index.js](#16-javascript--categoriesindexjs)
2. [uiHelpers.js — formatDateOnlyForDisplay()](#2-uihelpersjs--formatdateonlyfordisplay)
3. [Python Parser — Local DB Config](#3-python-parser--local-db-config)
4. [Admin Sidebar — Navigation Changes](#4-admin-sidebar--navigation-changes)

---

## 1. Categories — Include / Exclude Keywords

### 1.1 Database schema change

**Table:** `t_categories`

Two new nullable `LONGTEXT` columns were added to the existing table:

| Column | Type | Nullable | Inserted after | Notes |
|---|---|---|---|---|
| `include_keywords` | LONGTEXT | Yes | `name` | Comma-separated keywords a job must contain to belong to this category |
| `exclude_keywords` | LONGTEXT | Yes | `include_keywords` | Comma-separated keywords that disqualify a job from this category |

Both columns store plain text (comma-separated lists). Empty strings are normalized to `NULL` by the controller before saving.

---

### 1.2 Migration

**File:** `database/migrations/2026_04_16_000000_add_include_exclude_keywords_to_t_categories_table.php`

```php
<?php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        if (! Schema::hasTable('t_categories')) {
            return;
        }

        Schema::table('t_categories', function (Blueprint $table) {
            if (! Schema::hasColumn('t_categories', 'include_keywords')) {
                $table->longText('include_keywords')->nullable()->after('name');
            }
            if (! Schema::hasColumn('t_categories', 'exclude_keywords')) {
                $table->longText('exclude_keywords')->nullable()->after('include_keywords');
            }
        });
    }

    public function down(): void
    {
        if (! Schema::hasTable('t_categories')) {
            return;
        }

        Schema::table('t_categories', function (Blueprint $table) {
            if (Schema::hasColumn('t_categories', 'exclude_keywords')) {
                $table->dropColumn('exclude_keywords');
            }
            if (Schema::hasColumn('t_categories', 'include_keywords')) {
                $table->dropColumn('include_keywords');
            }
        });
    }
};
```

---

### 1.3 Model — TCategory

**File:** `app/Models/TCategory.php`

Add `include_keywords` and `exclude_keywords` to the `$fillable` array:

```php
protected $fillable = [
    'name',
    'slug',
    'icon_image_url',
    'include_keywords',   // ← added
    'exclude_keywords',   // ← added
];
```

No cast is needed — these are stored as plain `LONGTEXT` strings (comma-separated).

---

### 1.4 Controller — CategoryController

**File:** `app/Http/Controllers/CategoryController.php`

Three changes were made:

#### a) Global search — add include/exclude to search filter

In the `data()` method's `->filter()` callback, extend the `WHERE` clause to also search the new columns:

```php
// Before (only searched id and name):
$sub->whereRaw("CAST(id AS CHAR) LIKE ?", ["%{$globalSearch}%"])
    ->orWhereRaw("LOWER(name) LIKE ?", ["%{$globalSearch}%"]);

// After (also searches keyword columns):
$sub->whereRaw("CAST(id AS CHAR) LIKE ?", ["%{$globalSearch}%"])
    ->orWhereRaw("LOWER(name) LIKE ?", ["%{$globalSearch}%"])
    ->orWhereRaw('LOWER(COALESCE(include_keywords, "")) LIKE ?', ["%{$globalSearch}%"])
    ->orWhereRaw('LOWER(COALESCE(exclude_keywords, "")) LIKE ?', ["%{$globalSearch}%"]);
```

> `COALESCE` is used so `NULL` columns do not cause the `LIKE` to fail.

#### b) `store()` — accept and normalize new fields

Add validation rules and normalize before creating:

```php
// Validation (add to existing rules array):
'include_keywords' => 'nullable|string',
'exclude_keywords' => 'nullable|string',

// After validation, normalize before create:
$validatedData['include_keywords'] = $this->normalizeKeywordField($request->input('include_keywords'));
$validatedData['exclude_keywords'] = $this->normalizeKeywordField($request->input('exclude_keywords'));

$tCategory = TCategory::create($validatedData);
```

#### c) `update()` — accept and normalize new fields

Same as `store()`:

```php
'include_keywords' => 'nullable|string',
'exclude_keywords' => 'nullable|string',

$validatedData['include_keywords'] = $this->normalizeKeywordField($request->input('include_keywords'));
$validatedData['exclude_keywords'] = $this->normalizeKeywordField($request->input('exclude_keywords'));
```

#### d) New private helper `normalizeKeywordField()`

Add this method to the controller class:

```php
/**
 * Empty string → null; otherwise trimmed.
 */
private function normalizeKeywordField(?string $value): ?string
{
    if ($value === null) {
        return null;
    }
    $trimmed = trim($value);

    return $trimmed === '' ? null : $trimmed;
}
```

---

### 1.5 Blade views

#### `resources/views/admin/jobs-panel/categories/index.blade.php`

Add two `<th>` headers between "Name" and "Actions":

```html
<th>Include keywords</th>
<th>Exclude keywords</th>
```

Result table header row becomes:

```html
<tr>
    <th>ID</th>
    <th>Name</th>
    <th>Include keywords</th>   {{-- added --}}
    <th>Exclude keywords</th>   {{-- added --}}
    <th>Actions</th>
</tr>
```

---

#### `resources/views/admin/jobs-panel/categories/modals/add.blade.php`

Add two `<textarea>` fields inside `#addCategoryForm`, after the existing name field:

```html
<div class="mb-3">
    <label class="form-label" for="categoryIncludeKeywords">Include keywords</label>
    <textarea class="form-control" id="categoryIncludeKeywords" name="include_keywords" rows="3"
        placeholder="Keywords to include (comma-separated)"></textarea>
    <small class="form-text text-muted">Same style as aggregator keyword lists.</small>
</div>

<div class="mb-3">
    <label class="form-label" for="categoryExcludeKeywords">Exclude keywords</label>
    <textarea class="form-control" id="categoryExcludeKeywords" name="exclude_keywords" rows="3"
        placeholder="Keywords to exclude (comma-separated)"></textarea>
</div>
```

---

#### `resources/views/admin/jobs-panel/categories/modals/edit.blade.php`

Add the same two textareas inside `#editCategoryForm`, after the existing name field. Use element IDs prefixed with `edit`:

```html
<div class="mb-3">
    <label class="form-label" for="editCategoryIncludeKeywords">Include keywords</label>
    <textarea class="form-control" id="editCategoryIncludeKeywords" name="include_keywords" rows="3"
        placeholder="Keywords to include (comma-separated)"></textarea>
</div>

<div class="mb-3">
    <label class="form-label" for="editCategoryExcludeKeywords">Exclude keywords</label>
    <textarea class="form-control" id="editCategoryExcludeKeywords" name="exclude_keywords" rows="3"
        placeholder="Keywords to exclude (comma-separated)"></textarea>
</div>
```

> Note: These textareas have no `value` attribute — they are populated by JavaScript in the `fillForm` callback below.

---

### 1.6 JavaScript — categories/index.js

**File:** `public/js/jobs-panel/categories/index.js`

Three additions were made:

#### a) Add helper functions at the top of the `$(document).ready()` block

```js
function escapeHtml(text) {
    if (text == null || text === "") return "";
    const div = document.createElement("div");
    div.textContent = String(text);
    return div.innerHTML;
}

function escapeAttr(text) {
    return String(text)
        .replace(/&/g, "&amp;")
        .replace(/"/g, "&quot;")
        .replace(/</g, "&lt;")
        .replace(/>/g, "&gt;");
}

function renderKeywordsCell(data, type) {
    if (type === "sort" || type === "filter" || type === "type") {
        return data ? String(data) : "";
    }
    if (!data || String(data).trim() === "") {
        return '<span class="text-muted">—</span>';
    }
    const full = String(data);
    const max = 120;
    const truncated = full.length > max ? full.slice(0, max) + "…" : full;
    const titleAttr = full.length > max ? ` title="${escapeAttr(full)}"` : "";
    const escTrunc = escapeHtml(truncated);
    return `<span class="small"${titleAttr}>${escTrunc}</span>`;
}
```

#### b) Add two new DataTable columns inside `initializeDataTable({ columns: [...] })`

Insert after the `name` column definition and before the `actions` column:

```js
{
    data: "include_keywords",
    name: "include_keywords",
    orderable: false,
    render: renderKeywordsCell,
},
{
    data: "exclude_keywords",
    name: "exclude_keywords",
    orderable: false,
    render: renderKeywordsCell,
},
```

#### c) Populate new fields in the edit `fillForm` callback

Inside `bindCrudEvents({ edit: { fillForm: (id, row) => { ... } } })`, add the two new lines:

```js
fillForm: (id, row) => {
    $("#editCategoryId").val(id);
    $("#editCategoryName").val(row.name);
    $("#editCategoryIncludeKeywords").val(row.include_keywords ?? "");  // ← added
    $("#editCategoryExcludeKeywords").val(row.exclude_keywords ?? "");  // ← added
},
```

---

## 2. uiHelpers.js — formatDateOnlyForDisplay()

**File:** `public/js/jobs-panel/uiHelpers.js`
**Commit:** `7fb81fa`

A new shared utility function was prepended to the top of the file. It formats any ISO datetime string into a locale-aware date-only string (no time, no timezone offset) for display in admin tables.

```js
/**
 * Format an ISO / datetime string for admin tables: local calendar date only (no time or timezone).
 * @param {string} value
 * @returns {string}
 */
function formatDateOnlyForDisplay(value) {
    if (!value) {
        return "";
    }
    const d = new Date(value);
    if (Number.isNaN(d.getTime())) {
        return value;
    }
    return new Intl.DateTimeFormat(undefined, {
        year: "numeric",
        month: "short",
        day: "numeric",
    }).format(d);
}
```

**Behaviour:**
- Returns `""` for null/empty input.
- Returns the raw value unchanged if the string is not a valid date.
- Uses `Intl.DateTimeFormat` with the browser's default locale (e.g. `Apr 13, 2026` in `en-US`).
- This function is used by DataTable `created_at` column renderers across the jobs-panel JS files.

---

## 3. Python Parser — Local DB Config

**File:** `Jobs-Panel-Parser-main/src/config.py`
**Commit:** `7fb81fa`

The `local` environment block in the `DBS` dictionary was updated to point to the local development database:

```python
# Before:
"local": {
    "DB_HOST": "",
    "DB_PORT": 3306,
    "DB_NAME": "",
    "DB_USER": "root",
    "DB_PASS": "",
},

# After:
"local": {
    "DB_HOST": "localhost",
    "DB_PORT": 3306,
    "DB_NAME": "u428142103_test_parsing",
    "DB_USER": "root",
    "DB_PASS": "",
},
```

> This is a local development config — do not commit real credentials and do not use this config block in production deployments.

---

## 4. Admin Sidebar — Navigation Changes

**File:** `resources/views/admin/jobs-panel/layouts/app.blade.php`

### Commit 7fb81fa — added "Job Search Criteria" nav link

A new `<li>` was added to the sidebar accordion list:

```html
<li class="list-group-item bg-light border-0">
    <a href="{{ route('admin.jobs-panel.job-search-criteria.index') }}"
        class="text-decoration-none">
        <i class="bi bi-search me-2"></i> Job Search Criteria
    </a>
</li>
```

### Commit ad22bde — renamed nav link to "Job Search Aggregator"

The route name and label were updated to reflect the rename from criteria to aggregator:

```html
{{-- Before: --}}
<a href="{{ route('admin.jobs-panel.job-search-criteria.index') }}" class="text-decoration-none">
    <i class="bi bi-search me-2"></i> Job Search Criteria
</a>

{{-- After: --}}
<a href="{{ route('admin.jobs-panel.job-search-aggregator.index') }}" class="text-decoration-none">
    <i class="bi bi-search me-2"></i> Job Search Aggregator
</a>
```

> The route name change reflects the migration of the feature from the simple `JobSearchCriterionController` (modal-based CRUD on `job_search_criteria` table) to the full `JobSearchAggregatorController` (page-based CRUD on `t_job_search_aggregator` table).

---

## Summary of all files changed (excluding monitoring dashboard & job-search-criteria)

| File | Commit | Type of change |
|---|---|---|
| `database/migrations/2026_04_16_000000_add_include_exclude_keywords_to_t_categories_table.php` | ad22bde | New migration |
| `app/Models/TCategory.php` | ad22bde | Added fields to `$fillable` |
| `app/Http/Controllers/CategoryController.php` | ad22bde | Search + store + update + helper |
| `resources/views/admin/jobs-panel/categories/index.blade.php` | ad22bde | Added 2 `<th>` headers |
| `resources/views/admin/jobs-panel/categories/modals/add.blade.php` | ad22bde | Added 2 textarea fields |
| `resources/views/admin/jobs-panel/categories/modals/edit.blade.php` | ad22bde | Added 2 textarea fields |
| `public/js/jobs-panel/categories/index.js` | ad22bde | 3 additions: helpers, 2 columns, fillForm |
| `public/js/jobs-panel/uiHelpers.js` | 7fb81fa | Added `formatDateOnlyForDisplay()` |
| `Jobs-Panel-Parser-main/src/config.py` | 7fb81fa | Local DB host/name populated |
| `resources/views/admin/jobs-panel/layouts/app.blade.php` | 7fb81fa + ad22bde | Nav link added then renamed |
