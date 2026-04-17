# PR Likelihood Score — Calculation Logic Explained

This document explains exactly how a job goes from raw data to a  
🟢 **High** / 🟡 **Medium** / 🔴 **Low** PR Likelihood label.

---

## The Big Picture

The score is built from **4 independent signals**, each worth a certain number of points (0–100).  
The 4 point values are then **blended together using weights** (default: 40 / 25 / 20 / 15).  
The resulting number (0–100) is compared against two thresholds to assign the label.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                                                                 │
│   SIGNAL 1          SIGNAL 2          SIGNAL 3          SIGNAL 4               │
│  Provincial        Job Type          Employer          Wage Level               │
│   Demand           (full-time?)      (PR-friendly?)    (above baseline?)        │
│                                                                                 │
│   0–100 pts        0–100 pts         0–100 pts         0–100 pts               │
│   × 40%            × 25%             × 20%             × 15%                   │
│                                                                                 │
│   └──────────────────────────────────┬──────────────────────────┘              │
│                                      │                                          │
│                               FINAL SCORE (0–100)                              │
│                                      │                                          │
│                  ┌───────────────────┼──────────────────┐                      │
│                  │                   │                  │                       │
│               ≥ 70               40 – 69             < 40                      │
│            🟢 High            🟡 Medium            🔴 Low                      │
│                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Signal 1 — Provincial Demand (weight: 40%)

**Question:** Is this specific NOC code actively in demand in the province where the job is?

Admin maintains a table mapping `(Province, NOC Code) → Demand Level`. This data comes from provincial nominee program (PNP) websites and federal IRCC pages.

| What the lookup finds | Points awarded |
|----------------------|----------------|
| Province + NOC listed as **High demand** | **100 pts** |
| Province + NOC listed as **Medium demand** | **60 pts** |
| Province + NOC listed as **Low demand** | **25 pts** |
| NOC not in the demand table — TEER 0 or 1 job | **50 pts** (educated fallback) |
| NOC not in the demand table — TEER 2 or 3 job | **35 pts** (educated fallback) |
| NOC not in the demand table — TEER 4 or 5 job | **10 pts** (educated fallback) |
| Province could not be extracted from location | **0 pts** |

> **Why this is the heaviest signal (40%):** Provincial demand is the most direct PR pathway signal. A Registered Nurse in Ontario has a dedicated OINP stream. A retail worker in Toronto does not. No other signal carries this specificity.

---

## Signal 2 — Job Type (weight: 25%)

**Question:** Is this a stable, permanent, full-time role — or a temporary gig?

The job type string (from `t_job_types.title`) is matched by keyword:

| Job type title contains | Points awarded |
|------------------------|----------------|
| `"permanent"` | **100 pts** |
| `"full"` (full-time, not permanent) | **75 pts** |
| `"part"` (part-time) | **40 pts** |
| `"contract"`, `"temp"`, `"casual"`, `"seasonal"` | **20 pts** |
| Not set / unknown | **50 pts** (neutral — don't penalise) |

> **Why this matters:** Canadian PR programs (Express Entry, most PNPs) require a genuine job offer, typically full-time and non-seasonal. A casual shift job has almost no PR pathway value even if the NOC is eligible.

---

## Signal 3 — Employer PR Alignment (weight: 20%)

**Question:** Does this employer have a track record of hiring for PR-eligible roles?

Reads from `t_employers.pr_alignment_score` (computed nightly by the Employer Intelligence feature) and the admin flag on the employer.

| Employer state | Points awarded |
|---------------|----------------|
| Admin flagged **"Known PR-supportive"** | max(employer_score × 100, **85 pts**) |
| Admin flagged **"Low quality"** | min(employer_score × 100, **20 pts**) |
| Admin flagged **"Agency spam"** | **0 pts** |
| No flag, employer score present | employer_score × 100 (e.g. 0.65 → 65 pts) |
| No employer stats yet (new employer) | **50 pts** (neutral — benefit of the doubt) |

> **Why this matters:** Two identical job postings for the same NOC can have very different PR outcomes. A company that has hired and retained multiple immigrants for permanent roles is genuinely different from a temp agency posting the same title.

---

## Signal 4 — Wage Level (weight: 15%)

**Question:** Is the salary at a level consistent with a real full-time qualifying role?

The salary string is parsed to an hourly rate, then compared against a TEER-level baseline.  
Baseline rates represent the minimum wage level typically required for federal programs.

| TEER level | Baseline | Salary above baseline | Salary below baseline | No salary data |
|-----------|----------|-----------------------|-----------------------|----------------|
| TEER 0 / 1 | $30/hr | **90 pts** | **30 pts** | **40 pts** |
| TEER 2 / 3 | $22/hr | **75 pts** | **25 pts** | **40 pts** |
| TEER 4 / 5 | $16/hr | **60 pts** | **15 pts** | **40 pts** |

Salary formats handled: `"$28/hr"`, `"$55,000/year"`, `"$25–$30 per hour"` (averaged), `"$28.50 hourly"`.

> **Why this is the lightest signal (15%):** Many legitimate PR-track employers simply don't post salary (especially in healthcare and trades). Missing salary does not mean a bad job — neutral 40 pts avoids penalising good postings.

---

## The Formula

```
score = (demand_pts  × 40  / 100)
      + (jobtype_pts × 25  / 100)
      + (employer_pts × 20 / 100)
      + (wage_pts    × 15  / 100)

Rounded to the nearest whole number.
```

---

## Label Thresholds

| Score range | Label | Meaning |
|-------------|-------|---------|
| 70 – 100 | 🟢 **High PR Likelihood** | Strong signals across most or all dimensions |
| 40 – 69 | 🟡 **Medium PR Likelihood** | Some positive signals — worth considering |
| 0 – 39 | 🔴 **Low PR Likelihood** | Weak signals — unlikely to lead to PR |

Both thresholds (70 and 40) are admin-configurable in the Scoring Weights page.

---

## Worked Examples

---

### Example A — Registered Nurse, Ontario, Full-time Permanent
**Expected label: 🟢 High**

| Signal | Value | Points | Weight | Contribution |
|--------|-------|--------|--------|-------------|
| Provincial demand | ON + 31301 = **High** | 100 | 40% | **40.0** |
| Job type | "Full-time Permanent" | 100 | 25% | **25.0** |
| Employer | score 0.60, no flag | 60 | 20% | **12.0** |
| Wage | $38/hr, TEER 1 baseline $30/hr → above | 90 | 15% | **13.5** |

```
Score = 40.0 + 25.0 + 12.0 + 13.5 = 90.5 → 91
Label: 🟢 High  (91 ≥ 70)
```

---

### Example B — Retail Sales Associate, Toronto, Part-time
**Expected label: 🔴 Low**

| Signal | Value | Points | Weight | Contribution |
|--------|-------|--------|--------|-------------|
| Provincial demand | ON + 64100 = **not in table** → TEER 4 fallback | 10 | 40% | **4.0** |
| Job type | "Part-time" | 40 | 25% | **10.0** |
| Employer | score 0.10, no flag | 10 | 20% | **2.0** |
| Wage | $17/hr, TEER 4 baseline $16/hr → above | 60 | 15% | **9.0** |

```
Score = 4.0 + 10.0 + 2.0 + 9.0 = 25
Label: 🔴 Low  (25 < 40)
```

> **Why:** Retail is not a provincially-designated in-demand occupation, the role is part-time, and the employer has a weak PR track record. Even though the wage is above baseline, it barely moves the needle at 15% weight.

---

### Example C — Welder, Ontario, Full-time, Agency Posting
**Expected label: 🟡 Medium**

| Signal | Value | Points | Weight | Contribution |
|--------|-------|--------|--------|-------------|
| Provincial demand | ON + 72320 = **Medium** | 60 | 40% | **24.0** |
| Job type | "Full-time" | 75 | 25% | **18.75** |
| Employer | admin flag = "Agency spam" | 0 | 20% | **0.0** |
| Wage | $28/hr, TEER 2 baseline $22/hr → above | 75 | 15% | **11.25** |

```
Score = 24.0 + 18.75 + 0.0 + 11.25 = 54
Label: 🟡 Medium  (40 ≤ 54 < 70)
```

> **Key insight:** The employer "agency spam" flag drives the employer signal to 0. Without that flag, the employer signal would have been 50 pts (neutral), contributing +10, pushing the score to 64 — still Medium but closer to High. The admin flag protects users from misleading agency postings.

---

### Example D — Welder, Alberta, Full-time Permanent, Known PR Employer
**Expected label: 🟢 High**

| Signal | Value | Points | Weight | Contribution |
|--------|-------|--------|--------|-------------|
| Provincial demand | AB + 72320 = **High** | 100 | 40% | **40.0** |
| Job type | "Full-time Permanent" | 100 | 25% | **25.0** |
| Employer | flag = "Known PR-supportive", score 0.80 → max(80, 85) | 85 | 20% | **17.0** |
| Wage | $34/hr, TEER 2 baseline $22/hr → above | 75 | 15% | **11.25** |

```
Score = 40.0 + 25.0 + 17.0 + 11.25 = 93.25 → 93
Label: 🟢 High  (93 ≥ 70)
```

---

### Example E — Software Engineer, Ontario, Contract Role, Agency, No Salary
**Expected label: 🟡 Medium**

| Signal | Value | Points | Weight | Contribution |
|--------|-------|--------|--------|-------------|
| Provincial demand | ON + 21231 = **High** | 100 | 40% | **40.0** |
| Job type | "Contract" | 20 | 25% | **5.0** |
| Employer | flag = "Agency spam" | 0 | 20% | **0.0** |
| Wage | No salary data → neutral | 40 | 15% | **6.0** |

```
Score = 40.0 + 5.0 + 0.0 + 6.0 = 51
Label: 🟡 Medium  (40 ≤ 51 < 70)
```

> **Key insight:** Even though Software Engineer is High demand in Ontario, the contract job type (20 pts) and agency spam flag (0 pts) drag the score down. A strong demand signal alone is not enough — the employer and job structure matter. A user sees Medium, not High — this is intentional.

---

### Example F — PSW (Personal Support Worker), New Brunswick, No Salary
**Expected label: 🟢 High**

| Signal | Value | Points | Weight | Contribution |
|--------|-------|--------|--------|-------------|
| Provincial demand | NB + 33102 = **High** | 100 | 40% | **40.0** |
| Job type | "Full-time" | 75 | 25% | **18.75** |
| Employer | No stats yet (new employer) → neutral | 50 | 20% | **10.0** |
| Wage | No salary data → neutral | 40 | 15% | **6.0** |

```
Score = 40.0 + 18.75 + 10.0 + 6.0 = 74.75 → 75
Label: 🟢 High  (75 ≥ 70)
```

> **Key insight:** A High-demand designation in a province is powerful enough (at 40% weight) that even with neutral employer and wage signals, the job reaches High. PSWs are genuinely in demand in NB — the score reflects reality.

---

### Example G — Farm Labourer, Location Unknown
**Expected label: 🔴 Low**

| Signal | Value | Points | Weight | Contribution |
|--------|-------|--------|--------|-------------|
| Provincial demand | Province = null → 0 pts | 0 | 40% | **0.0** |
| Job type | "Seasonal" | 20 | 25% | **5.0** |
| Employer | score 0.15 | 15 | 20% | **3.0** |
| Wage | $15/hr, TEER 5 baseline $16/hr → below | 15 | 15% | **2.25** |

```
Score = 0.0 + 5.0 + 3.0 + 2.25 = 10.25 → 10
Label: 🔴 Low  (10 < 40)
```

---

## Score Sensitivity — What Moves the Needle Most

Because the demand signal is 40% of the total, it has the highest leverage:

| Change | Score impact |
|--------|-------------|
| Province changes from "not in table" (TEER 3 → 35 pts) to "High demand" (100 pts) | +26 points |
| Job type changes from "Part-time" (40 pts) to "Full-time Permanent" (100 pts) | +15 points |
| Employer flag changes from "Agency spam" (0 pts) to neutral (50 pts) | +10 points |
| Salary added above baseline vs no salary (neutral 40 → e.g. 75) | +5.25 points |

> **Implication for admins:** The most effective action to improve scores for a region is to add that province's in-demand NOCs to the Provincial Demand table. That alone can shift a job from Medium to High.

---

## What Happens When Admin Changes the Weights

**Current default:** Demand 40 / Job Type 25 / Employer 20 / Wage 15

**Scenario: Admin shifts to** Demand 50 / Job Type 30 / Employer 10 / Wage 10

Re-scoring Example A (RN Ontario, original score 91):

| Signal | Points | New weight | New contribution |
|--------|--------|-----------|-----------------|
| Provincial demand | 100 | 50% | **50.0** |
| Job type | 100 | 30% | **30.0** |
| Employer | 60 | 10% | **6.0** |
| Wage | 90 | 10% | **9.0** |

```
New score = 50.0 + 30.0 + 6.0 + 9.0 = 95
Label: still 🟢 High
```

Re-scoring Example E (Software Engineer Contract Agency, original score 51):

| Signal | Points | New weight | New contribution |
|--------|--------|-----------|-----------------|
| Provincial demand | 100 | 50% | **50.0** |
| Job type | 20 | 30% | **6.0** |
| Employer | 0 | 10% | **0.0** |
| Wage | 40 | 10% | **4.0** |

```
New score = 50.0 + 6.0 + 0.0 + 4.0 = 60
Label: 🟡 Medium (still, but higher — 60 vs 51)
```

> **Key takeaway:** Increasing demand weight rewards any job in a high-demand province more, even if other signals are weak. If the admin wants the scoring to focus more purely on provincial demand, they increase that weight. If they want employer quality and job permanency to matter more, they shift weight to those signals.

---

## What Score of `null` Means

A job gets `pr_likelihood_score = null` (no label shown at all) when:

- The job's `noc_code` is still `null` — NOC classification has not yet resolved this job's occupation

This happens when:
- The job is brand new and its title matched nothing in `t_noc_keywords`
- The job is in the NOC Review Queue waiting for admin confirmation

Once an admin confirms or overrides the NOC in the review queue, the next normalize cycle (or a manual `pr:recompute-likelihood --job=X`) will compute and store the score.

---

## Summary: Path to 🟢 High

A job needs a score ≥ 70. The most direct path:

| Must have | Points contribution at default weights |
|-----------|--------------------------------------|
| High demand province + NOC | 40 pts (40% of 100) |
| Full-time permanent | 25 pts (25% of 100) |
| Reasonable employer (score 0.5+, no bad flag) | 10+ pts |
| Wage at or above TEER baseline | 9–13 pts |
| **Minimum total** | **84+ pts** → 🟢 High |

A job can still reach High with Medium provincial demand if the other 3 signals are all strong:

| Medium demand (60 pts) | Full-time permanent (100) | Known PR employer (85) | Above-baseline wage (75) |
|------------------------|--------------------------|----------------------|--------------------------|
| 60 × 40% = **24** | 100 × 25% = **25** | 85 × 20% = **17** | 75 × 15% = **11.25** |

```
24 + 25 + 17 + 11.25 = 77.25 → 77 → 🟢 High
```

> A medium-demand NOC with a reputable full-time employer paying a fair wage is a better PR bet than a high-demand NOC posted by a temp agency for a casual role.
