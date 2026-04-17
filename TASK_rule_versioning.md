# TASK: Rule Versioning
**Feature:** 3.4 — Track and Rollback Rule Changes  
**Branch:** `rule-versioning`  
**Depends on:** No dependencies (standalone feature)

---

## Context & Why

Canadian provinces and IRCC change immigration/job rules frequently.  
Admins need to:
- Edit Tag Rules, Category keywords, Aggregator filters in the admin panel
- If a new rule breaks scoring/filtering, **roll back to the last working version in one click**
- See the full history of every change with a timestamp and change note

**Scope of versioning:**
| Rule Type | What Gets Versioned |
|-----------|-------------------|
| Tag Rules (`t_rules`) | name, t_category_id, all tag attachments (tag1/2/3), roadmap attachments |
| Category Keywords (`t_categories`) | include_keywords, exclude_keywords |
| Aggregator Keywords (`t_aggregators`) | include_keywords, exclude_keywords |

**Not in scope (Phase 1):** PR scoring, employer scoring (no models yet — add when those are built)

---

## Current Codebase State

### Models that need versioning
| Model | Table | Key fields |
|-------|-------|-----------|
| `TRule` | `t_rules` | `name`, `t_category_id` + pivot: `t_rule_t_tag`, `t_rule_t_tag_2`, `t_rule_t_tag_3`, `t_rule_roadmap` |
| `TCategory` | `t_categories` | `include_keywords`, `exclude_keywords` |
| `TAggregator` | `t_aggregators` | `include_keywords`, `exclude_keywords` |

### Controllers that make changes
- `TagRuleController` — `store()`, `update()`, `destroy()`
- `CategoryController` — `store()`, `update()`, `destroy()`
- `AggregatorController` — any update methods

### No versioning exists today
No `*_versions` tables, no audit log, no history UI.

---

## Architecture Decision: JSON Snapshot

Each version stores a **complete JSON snapshot** of the rule at the moment of change.

**Why JSON snapshot over column-level audit:**
- Rules are complex objects (a TRule has name + category + 3 tag sets + roadmaps)
- Rollback = just re-apply the snapshot (no complex diff logic)
- A single row holds everything needed to restore
- Easy to display as a diff in the UI

**Snapshot format (TRule example):**
```json
{
  "name": "Skilled Worker - Manitoba",
  "t_category_id": 5,
  "tag_ids":      [1, 4, 7],
  "tag2_ids":     [2, 9],
  "tag3_ids":     [3],
  "roadmap_ids":  [1]
}
```

**Snapshot format (TCategory example):**
```json
{
  "name": "Manitoba Skilled Worker",
  "include_keywords": "welder,electrician,plumber",
  "exclude_keywords": "senior,director"
}
```

---

## Database Design

### New table: `t_rule_versions`

```
id                  bigint unsigned PK auto-increment
versionable_type    varchar(100)        — e.g. "App\Models\TRule"
versionable_id      bigint unsigned     — FK to the versioned record
version_number      smallint unsigned   — 1, 2, 3 ... per record
snapshot            json                — full state at this version
change_note         varchar(255) null   — admin-provided reason e.g. "IRCC update Oct 2026"
changed_by          bigint unsigned FK null → users.id (set null on delete)
created_at          timestamp
```

**Indexes:**
- `(versionable_type, versionable_id, version_number)` — unique composite
- `(versionable_type, versionable_id)` — for history lookup
- `(changed_by)`

No `updated_at` — versions are immutable once written.

### Migration file to create
`database/migrations/2026_04_XX_000001_create_t_rule_versions_table.php`

---

## New Files to Create

```
app/
  Models/
    RuleVersion.php                        — Eloquent model for t_rule_versions
  Services/
    RuleVersioningService.php              — Core snapshot/restore logic
  Http/Controllers/
    RuleVersionController.php              — History list + rollback endpoints

resources/views/admin/jobs-panel/
  rule-versions/
    history.blade.php                      — Version history modal/page
    partials/
      diff.blade.php                       — Side-by-side diff of two snapshots

database/migrations/
  2026_04_XX_000001_create_t_rule_versions_table.php
```

---

## Files to Modify

| File | What changes |
|------|-------------|
| `app/Http/Controllers/TagRuleController.php` | Call `RuleVersioningService::snapshot()` before `update()` and `destroy()` |
| `app/Http/Controllers/CategoryController.php` | Same — snapshot before `update()` |
| `app/Http/Controllers/AggregatorController.php` | Same — snapshot before `update()` |
| `routes/web.php` | Add `rule-versions` routes |
| `resources/views/admin/jobs-panel/tag-rules/edit.blade.php` | Add "Version History" button + change note input |
| `resources/views/admin/jobs-panel/layouts/app.blade.php` | No change needed |

---

## Implementation Plan — Step by Step

---

### Phase 1 — Migration + Model

**Step 1.1** — Create migration `2026_04_XX_000001_create_t_rule_versions_table.php`
- Columns as described above
- Composite unique index on `(versionable_type, versionable_id, version_number)`

**Step 1.2** — Create `app/Models/RuleVersion.php`
```php
class RuleVersion extends Model {
    public $timestamps = false;       // only created_at
    const UPDATED_AT = null;

    protected $table = 't_rule_versions';
    protected $fillable = [
        'versionable_type', 'versionable_id',
        'version_number', 'snapshot', 'change_note', 'changed_by', 'created_at',
    ];
    protected $casts = [
        'snapshot'    => 'array',
        'created_at'  => 'datetime',
    ];

    public function versionable() {
        return $this->morphTo();   // polymorphic — works for TRule, TCategory, TAggregator
    }
    public function changedBy() {
        return $this->belongsTo(User::class, 'changed_by');
    }
}
```

**Step 1.3** — Add `HasVersions` trait to `TRule`, `TCategory`, `TAggregator`
```php
// Added to each model:
public function versions() {
    return $this->morphMany(RuleVersion::class, 'versionable')->orderByDesc('version_number');
}
public function latestVersion(): ?RuleVersion {
    return $this->versions()->first();
}
```

Run migration: `php artisan migrate --path=database/migrations/2026_04_XX_000001_create_t_rule_versions_table.php`

---

### Phase 2 — RuleVersioningService

Create `app/Services/RuleVersioningService.php` with two public methods:

**`snapshot(Model $model, string $changeNote = null): RuleVersion`**
- Builds the JSON snapshot depending on model type:
  - `TRule` → `['name', 't_category_id', 'tag_ids', 'tag2_ids', 'tag3_ids', 'roadmap_ids']`
  - `TCategory` → `['name', 'include_keywords', 'exclude_keywords']`
  - `TAggregator` → `['aggregator_name', 'include_keywords', 'exclude_keywords']`
- Gets next version number: `max(version_number) + 1` for this record (starts at 1)
- Inserts into `t_rule_versions`
- Returns the new `RuleVersion` record

**`rollback(RuleVersion $version): void`**
- Loads `$version->versionable` (the original model)
- Re-applies the snapshot fields to the model
- For `TRule`: restores `name`, `t_category_id`, then calls `sync()` on each pivot:
  - `$rule->tags()->sync($snapshot['tag_ids'])`
  - `$rule->tags2()->sync($snapshot['tag2_ids'])`
  - `$rule->tags3()->sync($snapshot['tag3_ids'])`
  - `$rule->roadmaps()->sync($snapshot['roadmap_ids'])`
- Wraps everything in `DB::transaction()`
- Does NOT create a new version entry on rollback — rollback IS the new version (calls `snapshot()` after restore so the rollback itself is tracked)

**`diffSnapshots(array $snapshotA, array $snapshotB): array`**
- Returns array of `['field' => string, 'old' => mixed, 'new' => mixed]`
- Used by the history view to show what changed between versions

---

### Phase 3 — Hook into Existing Controllers

**TagRuleController changes:**

`update(Request $request, $id)`:
```php
// BEFORE saving:
$changeNote = $request->input('change_note');
app(RuleVersioningService::class)->snapshot($rule, $changeNote);

// then proceed with the existing update logic...
```

`destroy($id)`:
```php
// Snapshot before delete so the last state is recoverable:
app(RuleVersioningService::class)->snapshot($rule, 'Rule deleted');
// then delete...
```

Same pattern applied to `CategoryController::update()` and `AggregatorController::update()`.

**Add `change_note` input to edit forms:**
- A small optional text input below the save button
- Placeholder: `"Reason for change (e.g. IRCC update April 2026)"`
- Not required — if blank, stored as `null`

---

### Phase 4 — RuleVersionController

Create `app/Http/Controllers/RuleVersionController.php`:

**`history(string $type, int $id)`**
- `$type` maps to model class: `rule` → `TRule`, `category` → `TCategory`, `aggregator` → `TAggregator`
- Loads all versions for this record ordered by `version_number DESC`
- Each version row shows: version number, change note, changed by, created at, "View Diff" link, "Rollback" button
- Returns JSON for DataTable or a partial view (modal)

**`diff(int $versionA, int $versionB)`**
- Loads both `RuleVersion` records
- Calls `RuleVersioningService::diffSnapshots()` 
- Returns JSON array of field changes
- Used by the frontend to render a side-by-side diff

**`rollback(int $versionId)`**
- POST endpoint
- Loads the `RuleVersion`
- Calls `RuleVersioningService::rollback($version)`
- Returns `{ success: true, message: "Rolled back to version N" }`

---

### Phase 5 — Routes

Add inside the `admin/jobs-panel` route group in `web.php`:

```php
use App\Http\Controllers\RuleVersionController;

Route::prefix('rule-versions')->name('rule-versions.')->group(function () {
    Route::get('/history/{type}/{id}',  [RuleVersionController::class, 'history'])->name('history');
    Route::get('/diff/{vA}/{vB}',       [RuleVersionController::class, 'diff'])->name('diff');
    Route::post('/rollback/{versionId}',[RuleVersionController::class, 'rollback'])->name('rollback');
});
```

---

### Phase 6 — Admin UI

**History modal** (shown when clicking "Version History" on the edit page):
```
┌─────────────────────────────────────────────────────┐
│  Version History — "Skilled Worker - Manitoba"       │
├──────┬──────────────────────┬──────────┬────────────┤
│  Ver │ Change Note          │ By       │ Date       │
├──────┼──────────────────────┼──────────┼────────────┤
│  v3  │ IRCC update Apr 2026 │ admin    │ 2026-04-10 │ ← current
│  v2  │ Added welder tag     │ admin    │ 2026-03-01 │ [Diff] [Rollback]
│  v1  │ Initial creation     │ admin    │ 2026-01-15 │ [Diff] [Rollback]
└──────┴──────────────────────┴──────────┴────────────┘
```

**Diff view** (shown when clicking "Diff"):
```
┌───────────────┬──────────────────┬──────────────────┐
│ Field         │ Version 1 (old)  │ Version 2 (new)  │
├───────────────┼──────────────────┼──────────────────┤
│ name          │ SK Skilled Worker│ SK Skilled Worker│ (unchanged)
│ tag_ids       │ [1, 4]           │ [1, 4, 7]        │ (added 7)
│ excl_keywords │ senior,director  │ senior           │ (removed director)
└───────────────┴──────────────────┴──────────────────┘
```

**Rollback confirmation modal:**
> "Roll back 'Skilled Worker - Manitoba' to Version 2?  
> This was last saved on 2026-03-01 by admin.  
> A new version (v4) will be created to record this rollback."

---

## Verification Checklist

- [ ] Migration creates `t_rule_versions` with correct columns and indexes
- [ ] Editing a Tag Rule and saving creates a new row in `t_rule_versions` with correct snapshot
- [ ] `change_note` is saved when provided, null when blank
- [ ] Editing without tags (bare rule) snapshots correctly
- [ ] Version history page shows all versions in descending order
- [ ] Diff view correctly identifies added/removed tag IDs and changed keyword strings
- [ ] Rollback restores: rule name, category, all 3 tag pivot sets, roadmap pivot
- [ ] Rollback itself creates a new version entry (so it is auditable)
- [ ] Deleting a rule first snapshots it (last state preserved)
- [ ] Category keyword changes are versioned independently
- [ ] Aggregator keyword changes are versioned independently
- [ ] `changed_by` stores the logged-in user ID
- [ ] UI "Rollback" button shows confirmation modal before calling the API

---

## Do NOT

- Do not delete old version rows — versions are immutable and permanent
- Do not snapshot on `store()` (create) — only on `update()` and `destroy()`
- Do not auto-apply a rollback without a confirmation modal
- Do not version TTag, TTag2, TTag3, or Roadmap records individually — they are reference data, not rules
- Do not change the Python scraper for this feature

---

## File Reference Summary

| File | Status |
|------|--------|
| `database/migrations/2026_04_XX_000001_create_t_rule_versions_table.php` | CREATE |
| `app/Models/RuleVersion.php` | CREATE |
| `app/Services/RuleVersioningService.php` | CREATE |
| `app/Http/Controllers/RuleVersionController.php` | CREATE |
| `resources/views/admin/jobs-panel/rule-versions/history.blade.php` | CREATE |
| `resources/views/admin/jobs-panel/rule-versions/partials/diff.blade.php` | CREATE |
| `app/Http/Controllers/TagRuleController.php` | MODIFY (add snapshot call) |
| `app/Http/Controllers/CategoryController.php` | MODIFY (add snapshot call) |
| `app/Http/Controllers/AggregatorController.php` | MODIFY (add snapshot call) |
| `resources/views/admin/jobs-panel/tag-rules/edit.blade.php` | MODIFY (add change_note + history button) |
| `routes/web.php` | MODIFY (add rule-versions routes) |
