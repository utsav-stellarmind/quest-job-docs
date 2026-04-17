# NOC Assignment Logic — Explained

This document explains exactly how a job title goes from a raw scraped string  
to an assigned **NOC code**, **TEER level**, and **confidence score** —  
and how that determines whether the assignment is automatic or flagged for admin review.

---

## Why NOC Matters

Every downstream feature depends on NOC:

```
Job title: "Registered Nurse - ICU"
                │
                ▼
         NOC 31301 / TEER 1
                │
         ┌──────┴──────────────────────────┐
         │                                 │
  PR Likelihood Score              Employer Intelligence
  (is this NOC in demand           (what % of employer's
   in this province?)               jobs are PR-eligible?)
```

Without a NOC, neither feature can function. **One job = one NOC = one TEER.** Always.

---

## The Two-Layer Engine

```
Job title arrives
       │
       ▼
┌──────────────────────────────────────────────────────┐
│  LAYER 1: Keyword Match                              │
│  Check title against admin-managed t_noc_keywords   │
│                                                      │
│  Match found?                                        │
│   YES → compute confidence from match ratio          │
│   NO  → fall through to Layer 2                      │
└──────────────────────────────────────────────────────┘
       │ (only if no keyword match)
       ▼
┌──────────────────────────────────────────────────────┐
│  LAYER 2: TEER Signal Inference                      │
│  Scan title + description for government-defined     │
│  TEER indicator words                                │
│                                                      │
│  Hits found?                                         │
│   YES → assign TEER level (no specific NOC code)     │
│   NO  → noc_code = null, confidence = 0.00           │
└──────────────────────────────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────────────────────┐
│  ROUTING DECISION                                    │
│  confidence ≥ 0.60 → auto_assigned (no review)       │
│  confidence 0.30–0.59 → flagged_for_review           │
│  confidence < 0.30 → flagged_for_review              │
└──────────────────────────────────────────────────────┘
```

---

## Layer 1 — Keyword Match in Detail

The admin maintains a table (`t_noc_keywords`) where each row is one NOC:

```
NOC Code | TEER | NOC Title                              | Keywords
---------|------|----------------------------------------|--------------------------------------------
72320    |  2   | Welders and related machine operators  | welder, mig welder, tig welder, welding, stick welder
21231    |  1   | Software engineers and designers       | software engineer, full stack developer, backend developer, devops engineer
31301    |  1   | Registered nurses                      | registered nurse, rn, nurse practitioner, np
64100    |  4   | Retail salespersons                    | retail sales associate, sales associate, retail associate
```

**How the match works:**

```
titleLower = strtolower(job title)

For each NOC row (active only):
    For each keyword in that row's keyword list:
        if keyword appears anywhere in titleLower:
            matchedTokenCount++
            record this NOC as the best candidate

Keep the NOC row with the highest matchedTokenCount.
```

**Confidence formula:**

```
titleWordCount = number of words in the job title
confidence     = matchedTokenCount / titleWordCount

Floors and caps:
  - Any match → confidence floor of 0.40 (minimum credit for any match)
  - Maximum confidence = 1.00
```

**Routing:**

```
confidence ≥ 0.60  →  auto_assigned   (system is confident enough, no human needed)
confidence < 0.60  →  flagged_for_review (system has a best guess, admin confirms)
```

---

## Layer 1 — Confidence Walkthrough

### "Welder" (1 word title)
```
Keywords matched in NOC 72320: "welder" → 1 match
titleWordCount = 1
confidence = 1 / 1 = 1.00  (floor 0.40, cap 1.00 → 1.00)

→ NOC 72320, TEER 2, confidence 1.00
→ noc_review_status = auto_assigned ✅
```

### "MIG Welder" (2 words)
```
Keywords matched in NOC 72320: "welder" → 1, "mig welder" → 2
titleWordCount = 2
confidence = 2 / 2 = 1.00

→ NOC 72320, TEER 2, confidence 1.00
→ noc_review_status = auto_assigned ✅
```

### "Senior MIG Welder Alberta" (4 words)
```
Keywords matched in NOC 72320: "welder" → 1, "mig welder" → 2
titleWordCount = 4
confidence = 2 / 4 = 0.50  (above floor of 0.40)

→ NOC 72320, TEER 2, confidence 0.50
→ noc_review_status = flagged_for_review ⚠️
  (system's best guess is 72320, admin confirms)
```

### "Senior Welding Supervisor" (3 words)
```
Keywords matched in NOC 72320: "welding" → 1, "welder" → no ("welding" ≠ "welder" as substring)
Wait — "welding" IS in the keyword list → 1 match
titleWordCount = 3
confidence = 1 / 3 = 0.33  (above floor of 0.40 → floored to 0.40)

→ NOC 72320, TEER 2, confidence 0.40
→ noc_review_status = flagged_for_review ⚠️
```

> **Admin action:** Admin sees this in the NOC Review Queue. They look at the title, agree NOC 72320 is correct, click **Confirm** → `admin_confirmed`. Or if it's actually a supervisory role under NOC 72021, they select that and click **Override** → `admin_overridden`.

### "Software Engineer II" (3 words)
```
Keywords matched in NOC 21231: "software engineer" → 1
titleWordCount = 3
confidence = 1 / 3 = 0.33  (floored to 0.40)

→ NOC 21231, TEER 1, confidence 0.40
→ noc_review_status = flagged_for_review ⚠️
```

> **Admin tip:** If admin adds the keyword `"software engineer ii"` to NOC 21231, the match count becomes 2. Confidence = 2/3 = 0.67 → auto_assigned. **Adding specific title variants to keywords reduces the review queue.**

### "Registered Nurse - ICU" (3 meaningful words, hyphen ignored)
```
Keywords matched in NOC 31301: "registered nurse" → 1, "rn" → no
titleWordCount = 4 (including hyphen-separated)
confidence = 1 / 4 = 0.25  (floored to 0.40)

→ NOC 31301, TEER 1, confidence 0.40
→ noc_review_status = flagged_for_review ⚠️
```

> **Admin tip:** Adding `"nurse"` as a standalone keyword for NOC 31301 would match this title too, giving 2 matches → 2/4 = 0.50 → still flagged but confidence higher. Adding `"registered nurse - icu"` gives 3 matches → 3/4 = 0.75 → auto_assigned.

---

## Layer 2 — TEER Signal Inference

Runs **only when Layer 1 finds zero keyword matches**.

The engine checks the title + first 500 characters of the job description against signal words defined directly from the Canadian government TEER definitions:

```
TEER 0 signals: "general manager", "managing director", "vp ", "chief executive",
                "director of", "ceo", "cfo", "coo", "president and"

TEER 1 signals: "degree required", "bachelor", "master's", "phd", "software engineer",
                "financial advisor", "physician", "lawyer", "university degree"

TEER 2 signals: "college diploma", "apprenticeship", "journeyman", "journeyperson",
                "technologist", "supervisor", "red seal", "2-year diploma"

TEER 3 signals: "on-the-job training", "trade certificate", "dental assistant",
                "care aide", "6 months experience", "less than 2 year"

TEER 4 signals: "high school diploma", "no degree required", "retail", "cashier",
                "several weeks training", "child care provider"

TEER 5 signals: "no formal education", "grounds maintenance", "farm labourer",
                "delivery driver", "no experience required"
```

**Signal hit counting:**

```
searchText = lowercase(title) + " " + lowercase(description[0..500])

For each TEER level 0–5:
    Count how many signal phrases appear in searchText

Best TEER = level with highest hit count (minimum 1 hit required)
```

**Confidence from Layer 2:**

| Hit count for winning TEER | Confidence |
|---------------------------|-----------|
| 1 hit | 0.25 |
| 2 hits | 0.40 |
| 3 or more hits | 0.55 |

**Important:** Layer 2 gives you a TEER level, NOT a specific NOC code.  
`noc_code` remains `null`. Only `teer_level` is set.  
These jobs always land in the review queue — admin selects the specific NOC.

---

## Layer 2 — Signal Walkthrough

### "Production Lead, Metal Fabrication" (no keyword match)
```
Layer 1: no keywords match → fall to Layer 2

searchText: "production lead metal fabrication"
TEER 0: 0 hits
TEER 1: 0 hits
TEER 2: "supervisor" not present... but does description say something?
  Assume description says "will supervise a team of 4 operators"
  → "supervise" → 1 hit for TEER 2
TEER 3: 0 hits
TEER 4: 0 hits
TEER 5: 0 hits

Best TEER = 2, hits = 1 → confidence = 0.25
→ noc_code = null, teer_level = 2, confidence 0.25
→ noc_review_status = flagged_for_review ⚠️
```

> **In the review queue:** Admin sees title "Production Lead" with TEER 2 guess. They search the dropdown and select NOC 72021 (Contractors and supervisors — general construction trades) → **Override** → locked.

### "Senior Account Executive" (no keyword match)
```
Layer 1: no keyword match → Layer 2

searchText: "senior account executive ... bachelor's degree required ... 
             3+ years experience in B2B sales..."

TEER 1 hits: "bachelor's" → 1 hit
TEER 0 hits: 0
TEER 2 hits: 0

Best TEER = 1, hits = 1 → confidence = 0.25
→ noc_code = null, teer_level = 1, confidence 0.25
→ flagged_for_review ⚠️
```

### "General Helper" (no keyword match, generic description)
```
Layer 1: no match
Layer 2: no signal hits in title or short description
→ noc_code = null, teer_level = null, confidence = 0.00
→ flagged_for_review ⚠️ (no guess at all — admin must select from scratch)
```

---

## Routing Decision Summary

```
┌──────────────────────────────┬─────────────────────┬───────────────────────────────────┐
│ Confidence                   │ Status              │ What happens                      │
├──────────────────────────────┼─────────────────────┼───────────────────────────────────┤
│ ≥ 0.60 (Layer 1 match)       │ auto_assigned       │ Stored. Used downstream. Done.    │
│ 0.30–0.59 (Layer 1 partial)  │ flagged_for_review  │ Best guess stored. Admin reviews.  │
│ < 0.30 (Layer 2 or no match) │ flagged_for_review  │ Partial/no guess. Admin assigns.  │
│ 0.00 (no match at all)       │ flagged_for_review  │ noc_code = null. Admin picks.     │
└──────────────────────────────┴─────────────────────┴───────────────────────────────────┘
```

In all flagged cases, `pr_likelihood_score` is computed using whatever partial data exists (TEER fallback if no NOC).  
Once admin confirms/overrides, the next normalize cycle recomputes the score with the corrected NOC.

---

## The Admin Lock — Why It Matters

Every time `jobs:normalize` runs (every 8–10 hours), the classification service is called again on every job. Without a lock, an admin correction would be overwritten on the next scrape cycle.

**The lock check — first thing the service does:**

```
if noc_review_status IN ['admin_confirmed', 'admin_overridden']:
    skip classification entirely
    only recompute pr_score using the locked NOC
    return
```

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  Admin confirms "NOC 72320 is correct" for "Senior MIG Welder AB"  │
│               noc_review_status = 'admin_confirmed'                 │
│                                                                     │
│  Next day: scraper re-scrapes the same job                          │
│  jobs:normalize runs                                                 │
│  Service checks: admin_confirmed? YES → SKIP classification         │
│                                                                     │
│  NOC 72320 stays. Forever. Unless admin explicitly changes it again.│
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

**Admin override flow:**

```
Review queue row: "Senior MIG Welder AB | Guess: NOC 72320 (confidence 0.50)"

Option A — Confirm:
  Admin agrees → clicks [Confirm]
  → noc_review_status = 'admin_confirmed'
  → noc_confidence = unchanged (0.50)
  → job disappears from queue
  → locked

Option B — Override:
  Admin disagrees → selects NOC 72021 from dropdown → clicks [Apply Override]
  → noc_code = '72021', teer_level = 2
  → noc_review_status = 'admin_overridden'
  → noc_confidence = 1.00 (admin override = 100% certain)
  → noc_match_layer = 'admin'
  → job disappears from queue
  → locked
```

---

## What Stored Per Job

After classification runs, these columns are written on `t_jobs`:

| Column | Example value | Meaning |
|--------|--------------|---------|
| `noc_code` | `72320` | The assigned NOC |
| `teer_level` | `2` | 0–5 TEER level |
| `noc_confidence` | `0.50` | System's certainty (0.00–1.00) |
| `noc_match_layer` | `keyword` | `keyword`, `signal`, `none`, or `admin` |
| `noc_review_status` | `flagged_for_review` | Current state of this assignment |
| `noc_overridden_by` | `null` or user ID | Admin who last touched this |
| `noc_reviewed_at` | `null` or timestamp | When admin last acted |

---

## How Admin Reduces the Review Queue Over Time

The review queue is a quality signal, not a failure state. It shrinks as admins add better keywords.

```
Day 1:  300 jobs flagged (new system, limited keywords)
        Admin reviews, confirms/overrides all 300

Day 2:  80 new jobs scraped
        60 auto-assigned (new keywords help)
        20 flagged

Day 7:  Admin adds "registered nurse - icu", "icu nurse" keywords to NOC 31301
        Runs: php artisan noc:reclassify
        15 previously-flagged ICU Nurse jobs → now auto_assigned

Steady state: most common title variants are in keywords
              queue only fills with unusual/new titles
```

**Most effective keywords to add:** the exact title strings that appear most often in the review queue.

---

## Full Example — End to End

**Job scraped:** `"ICU Nurse, Full-time, Vancouver BC, $45/hr"`

```
Step 1: jobs:normalize runs

Step 2: NocClassificationService::classify($job)

   noc_review_status check → null (new job) → proceed

   Layer 1 keyword match:
     titleLower = "icu nurse"
     NOC 31301 keywords: "registered nurse", "rn", "nurse practitioner", "np"
     → "nurse" not in keyword list... but wait: is "nurse" a keyword?
       If admin has added "nurse" as keyword → 1 match / 2 words = 0.50 → flagged
       If admin has NOT added "nurse" → 0 matches → Layer 2

   Layer 2 fallback (if no match):
     searchText = "icu nurse full-time vancouver bc ..."
     TEER 1 signals: none hit
     TEER 2 signals: none hit
     No hits → confidence 0.00

   Result without "nurse" keyword:
     noc_code = null, teer_level = null, confidence 0.00
     noc_review_status = 'flagged_for_review'

Step 3: Job appears in NOC Review Queue
   Admin sees: "ICU Nurse | No guess | confidence 0%"
   Admin selects NOC 31301 from dropdown
   Admin clicks [Apply Override]
   → noc_code = '31301', teer_level = 1, admin_overridden, locked

Step 4: Admin also adds "nurse", "icu nurse", "icu rn" to NOC 31301 keyword list

Step 5: Admin runs: php artisan noc:reclassify
   → All similar future jobs auto-assigned to NOC 31301

Step 6: Next scrape cycle — "ICU Nurse" jobs skip review queue entirely
```

---

## Summary Table

| Scenario | Layer | NOC Assigned | TEER | Confidence | Status |
|----------|-------|-------------|------|-----------|--------|
| "Welder" | 1 — exact keyword | 72320 | 2 | 1.00 | auto_assigned |
| "MIG Welder" | 1 — multi keyword | 72320 | 2 | 1.00 | auto_assigned |
| "Senior MIG Welder AB" | 1 — partial | 72320 | 2 | 0.50 | flagged |
| "Registered Nurse - ICU" (with "nurse" keyword) | 1 — partial | 31301 | 1 | 0.40 | flagged |
| "Production Lead" + supervisory description | 2 — signal | null | 2 | 0.25 | flagged |
| "Senior Account Executive" + "bachelor's" in desc | 2 — signal | null | 1 | 0.25 | flagged |
| "General Helper" — no match, no signals | none | null | null | 0.00 | flagged |
| Admin-confirmed "Senior MIG Welder AB" | admin lock | 72320 | 2 | 0.50 | admin_confirmed |
| Admin-overridden "ICU Nurse" → 31301 | admin override | 31301 | 1 | 1.00 | admin_overridden |
