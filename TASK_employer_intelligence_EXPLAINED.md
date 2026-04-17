# Employer Intelligence Logic — Explained

This document explains exactly how an employer goes from a raw company name  
to a **PR Alignment Score** (0.0–1.0), a **label** (High / Medium / Low),  
and a set of **human-readable signal strings** shown to users.

---

## What Employer Intelligence Measures

It answers one question: **"Does this company actually hire people for roles that lead to PR?"**

Not just "do they post jobs" — but do they post:
- The right kinds of roles (PR-eligible NOC codes)
- Consistently (not random one-offs)
- In the right locations (PR-friendly regions)
- As stable, ongoing employment (not temp gigs)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  Everything we know about an employer comes from their job postings.   │
│  No external data. No self-reported info. Pure behaviour history.      │
│                                                                         │
│  All jobs → their pr_score (0–100) → aggregated per employer           │
│                                    → employer pr_alignment_score (0–1) │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## How the Score Is Built

The employer score has **three components**:

```
┌──────────────────────────┬────────────────────────────────────────────────────┐
│  Component               │  What it measures                                  │
├──────────────────────────┼────────────────────────────────────────────────────┤
│  Base (weight: 50%)      │  Average quality of jobs posted (avg pr_score)     │
│  Frequency (weight: 20%) │  How actively they're hiring right now             │
│  Consistency (weight: 30%)│  What % of their jobs are PR-eligible              │
└──────────────────────────┴────────────────────────────────────────────────────┘
```

---

## Component 1 — Base Score (50%)

**What it is:** Average `pr_score` of all jobs posted in the last 90 days ÷ 100

```
base = avg(t_jobs.pr_score WHERE employer_id = X AND posted_at >= 90 days ago) / 100
```

| Average job pr_score | Base value |
|---------------------|-----------|
| 85 avg (high-quality jobs) | 0.85 |
| 55 avg (mixed bag) | 0.55 |
| 20 avg (low-quality jobs) | 0.20 |

> **Why it's the heaviest component (50%):** The quality of what an employer posts IS the employer's intelligence. An employer who consistently posts high-scoring jobs (full-time, PR-eligible NOC, good location) is demonstrably PR-friendly.

---

## Component 2 — Frequency Factor (20%)

**What it is:** How many jobs the employer posted in the last 30 days, normalised to a 0–1 scale capped at 5 jobs/month.

```
frequency_f = min(1.0, posting_frequency_30d / 5)
```

| Jobs posted in last 30 days | Frequency factor |
|---------------------------|-----------------|
| 0 jobs | 0.00 |
| 1 job | 0.20 |
| 2 jobs | 0.40 |
| 3 jobs | 0.60 |
| 4 jobs | 0.80 |
| 5+ jobs | 1.00 (capped) |

> **Why frequency matters:** An employer who posted one great job 6 months ago and nothing since is less valuable to a job-seeker than one who consistently hires. Frequency signals active ongoing need.

> **Why cap at 5:** High-volume agency spam also posts frequently. Capping frequency prevents volume alone from inflating the score. Quality (base) and consistency (below) still dominate.

---

## Component 3 — Consistency Factor (30%)

**What it is:** The percentage of the employer's jobs that have `pr_score >= 50` (PR-eligible threshold).

```
consistency_f = pr_eligible_jobs_count / total_jobs_count
             = pr_eligible_pct  (stored as 0.00–1.00)
```

| PR-eligible job % | Consistency factor |
|------------------|-------------------|
| 90% of jobs are PR-eligible | 0.90 |
| 60% of jobs are PR-eligible | 0.60 |
| 15% of jobs are PR-eligible | 0.15 |

> **Why consistency is second-heaviest (30%):** An employer who *sometimes* posts PR-eligible jobs between a sea of casual/temporary roles is a mixed signal. Consistency identifies employers where PR eligibility is the norm, not the exception.

---

## The Formula

```
computed = (base × 0.5) + (frequency_f × 0.2) + (consistency_f × 0.3)

final = clamp(computed + (admin_score_override × 0.1),  min: 0.0,  max: 1.0)
```

---

## Admin Score Override

Admin can nudge the computed score in either direction without replacing it:

| Override value | Effect on score | Use case |
|---------------|----------------|----------|
| +2 | +0.20 | "We know this employer is excellent — boost them" |
| +1 | +0.10 | "Slightly better than data shows" |
| 0 | No change | Default |
| -1 | -0.10 | "Slightly worse than data shows" |
| -2 | -0.20 | "This employer is not PR-friendly despite the numbers" |

The final score is always clamped between 0.0 and 1.0.

---


## Score Bands and Labels

| Final score | Label | User-facing display |
|-------------|-------|-------------------|
| ≥ 0.70 | High | "Strong PR pathway alignment" |
| 0.40–0.69 | Medium | "Moderate PR pathway alignment" |
| < 0.40 | Low | "Limited PR pathway signals" |

---

## Admin Flags

Three flags admin can apply to any employer:

| Flag | Effect on Employer Intelligence | Effect on PR Likelihood Score (per job) |
|------|---------------------------------|----------------------------------------|
| `known_pr_supportive` | Shown in user-facing signals | Employer signal: max(score × 100, 85 pts) |
| `low_quality` | Admin note only, no user display | Employer signal: min(score × 100, 20 pts) |
| `agency_spam` | Employer visually marked in admin panel | Employer signal: 0 pts always |

> **Flags override the computed score for job-level PR scoring (Employer Intelligence signal), but the employer's `pr_alignment_score` number itself is still computed normally.** The flag is applied during PR Likelihood Scoring, not stored as the score itself.

---

## Signal Strings — What Users See

After the employer stats are computed, the system generates plain-English strings stored in `signals_json`. These are the sentences shown on job listings and employer profiles.

**How each string is generated:**

| Signal key | Generated from | Example output |
|-----------|---------------|----------------|
| `hiring_pattern` | top_titles[0] + top_locations[0] | "Consistently hires Welders in North Bay, ON" |
| `pr_alignment` | pr_eligible_pct × 100, rounded | "78% of roles are PR-eligible" |
| `frequency` | posting_frequency_30d | "Posted 8 jobs in the past 30 days" |
| `admin_flag` | admin_flag value (only if set) | "Known PR-supportive employer" |

**Displayed on job listing (user sees):**
```
Northern Steel Works
🟢 Strong PR pathway alignment
"Consistently hires Welders in North Bay, ON"
"78% of roles are PR-eligible"
```

**Displayed on employer profile (user sees):**
```
Northern Steel Works
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🟢 High likelihood of ongoing hiring
Roles aligned with PR pathways
This employer has a strong history of hiring for PR-eligible roles

Posted 8 jobs in the past 30 days
Top roles: Welder, Structural Fitter, Welding Supervisor
Top locations: North Bay, ON · Sudbury, ON
```

---

## Worked Examples

---

### Example A — Northern Steel Works (Ideal employer)

**Facts:**
- 34 total jobs, 12 active
- avg pr_score of jobs: 72
- Jobs posted last 30 days: 8
- PR-eligible jobs (pr_score ≥ 50): 29 out of 34 = 85%
- Admin flag: `known_pr_supportive`
- Admin override: 0

```
base          = 72 / 100 = 0.72
frequency_f   = min(1.0, 8 / 5) = 1.00
consistency_f = 29 / 34 = 0.85

computed = (0.72 × 0.5) + (1.00 × 0.2) + (0.85 × 0.3)
         = 0.360 + 0.200 + 0.255
         = 0.815

final = clamp(0.815 + (0 × 0.1)) = 0.815 → 0.82
Label: 🟢 High  (0.82 ≥ 0.70)
```

**Signals generated:**
- "Consistently hires Welders in North Bay, ON"
- "85% of roles are PR-eligible"
- "Posted 8 jobs in the past 30 days"
- "Known PR-supportive employer"

---

### Example B — GTA Temp Agency (High volume, low quality)

**Facts:**
- 89 total jobs
- avg pr_score of jobs: 18 (mostly casual, part-time, TEER 4/5)
- Jobs posted last 30 days: 22
- PR-eligible jobs: 11 out of 89 = 12%
- Admin flag: `agency_spam`
- Admin override: 0

```
base          = 18 / 100 = 0.18
frequency_f   = min(1.0, 22 / 5) = 1.00
consistency_f = 11 / 89 = 0.12

computed = (0.18 × 0.5) + (1.00 × 0.2) + (0.12 × 0.3)
         = 0.090 + 0.200 + 0.036
         = 0.326

final = clamp(0.326 + (0 × 0.1)) = 0.33
Label: 🔴 Low  (0.33 < 0.40)
```

> **Key insight:** Despite posting 22 jobs in 30 days (frequency = 1.0), the employer scores Low. High posting volume with low job quality is the agency spam pattern — the consistency component (0.12) and base component (0.18) correctly pull the score down.

---

### Example C — Small Healthcare Clinic (New, few jobs but high quality)

**Facts:**
- 6 total jobs (all registered nurses and LPNs)
- avg pr_score: 85
- Jobs posted last 30 days: 1
- PR-eligible jobs: 6 out of 6 = 100%
- Admin flag: none
- Admin override: 0

```
base          = 85 / 100 = 0.85
frequency_f   = min(1.0, 1 / 5) = 0.20
consistency_f = 6 / 6 = 1.00

computed = (0.85 × 0.5) + (0.20 × 0.2) + (1.00 × 0.3)
         = 0.425 + 0.040 + 0.300
         = 0.765

final = 0.77
Label: 🟢 High  (0.77 ≥ 0.70)
```

> **Key insight:** Even with only 1 job posted last month (frequency = 0.20), this clinic scores High because 100% of its jobs are PR-eligible (consistency = 1.0) and the average job quality is excellent (base = 0.85). Low frequency doesn't kill a good employer — quality and consistency do the heavy lifting.

---

### Example D — Construction Company, Admin Boosts

**Facts:**
- avg pr_score: 58
- posting_frequency_30d: 3
- PR-eligible %: 70%
- Admin override: **+2** ("We verified they've sponsored multiple workers")

```
base          = 58 / 100 = 0.58
frequency_f   = min(1.0, 3 / 5) = 0.60
consistency_f = 0.70

computed = (0.58 × 0.5) + (0.60 × 0.2) + (0.70 × 0.3)
         = 0.290 + 0.120 + 0.210
         = 0.620

final = clamp(0.620 + (2 × 0.1)) = clamp(0.820) = 0.82
Label: 🟢 High  (was Medium at 0.62, now High at 0.82 after override)
```

> **Key insight:** Without the admin override, this company would be Medium (0.62). The admin's verified knowledge that they actively sponsor workers lifts them to High (+0.20). The computed score stays as a reference — the override is transparent in the admin panel.

---

### Example E — Large Retailer, Admin Reduces

**Facts:**
- avg pr_score: 48 (mostly TEER 4 retail jobs)
- posting_frequency_30d: 10
- PR-eligible %: 48%
- Admin override: **-2** ("Known to only hire temporary seasonal workers despite full-time listings")

```
base          = 48 / 100 = 0.48
frequency_f   = min(1.0, 10 / 5) = 1.00
consistency_f = 0.48

computed = (0.48 × 0.5) + (1.00 × 0.2) + (0.48 × 0.3)
         = 0.240 + 0.200 + 0.144
         = 0.584

final = clamp(0.584 + (-2 × 0.1)) = clamp(0.384) = 0.38
Label: 🔴 Low  (was Medium at 0.58, now Low at 0.38 after override)
```

> **Key insight:** The computed score says Medium — but admin knows from complaints that this retailer's "full-time" listings are actually seasonal. The -2 override corrects this, protecting users from a misleading score.

---

### Example F — Brand New Employer (No stats yet)

**Facts:**
- First job just scraped and normalized
- `t_employer_stats` row doesn't exist yet (nightly batch hasn't run)
- `pr_alignment_score = 0.0` (default)

**In PR Likelihood Scoring (for the job's employer signal):**
```
No stats → use neutral 50 pts for employer signal
```

**After first nightly batch runs:**
```
Only 1 job to aggregate from → limited data
base = job's pr_score / 100
frequency_f = min(1.0, 1/5) = 0.20
consistency_f = 1 or 0 (1 job either is or isn't eligible)

If that job has pr_score = 70:
computed = (0.70 × 0.5) + (0.20 × 0.2) + (1.00 × 0.3)
         = 0.350 + 0.040 + 0.300
         = 0.690

Label: 🟡 Medium (0.69, just below High)
```

> **Key insight:** New employers start with limited data and naturally score Medium/Low until they build a job history. This is intentional — the system should not claim High PR alignment for an unknown employer.

---

## What Moves the Score Most

Because base score has a 50% weight, job quality is the dominant factor:

| Change | Score impact |
|--------|-------------|
| avg pr_score improves from 40 → 80 (better jobs) | +0.20 |
| consistency improves from 50% → 90% PR-eligible | +0.12 |
| frequency improves from 1/month → 5+/month | +0.16 |
| Admin override changes 0 → +2 | +0.20 |

> **For an employer to improve their score, they need to post better jobs** — full-time, permanent, appropriate NOC, in PR-friendly locations. The employer can't game the score directly; it follows from the jobs they post.

---

## When Scores Are Recomputed

| Trigger | What happens |
|---------|-------------|
| Nightly at 02:00 AM | `employer:recompute-intel` runs for all employers |
| Admin clicks "Recompute" on employer detail page | Instant recompute for that one employer |
| Admin changes `admin_score_override` or `admin_flag` | `applyAdminOverride()` immediately recomputes and saves |

Jobs processed during `jobs:normalize` read **whatever score is currently on the employer**.  
If the nightly batch hasn't run yet today, jobs scored today use yesterday's employer score.  
This is acceptable — employer scores are directionally stable day-to-day.

---

## Summary: Score Reference Table

| Employer type | Base | Frequency | Consistency | Computed | Final | Label |
|--------------|------|-----------|------------|---------|-------|-------|
| Ideal (high quality, active, consistent) | 0.85 | 1.00 | 0.90 | 0.845 | 0.85 | 🟢 High |
| Good but small (quality jobs, rare posting) | 0.80 | 0.20 | 1.00 | 0.740 | 0.74 | 🟢 High |
| Mixed bag (some good, some bad) | 0.55 | 0.60 | 0.60 | 0.575 | 0.58 | 🟡 Medium |
| Agency spam (high volume, low quality) | 0.18 | 1.00 | 0.12 | 0.326 | 0.33 | 🔴 Low |
| New employer (1 quality job) | 0.70 | 0.20 | 1.00 | 0.690 | 0.69 | 🟡 Medium |
| New employer (1 bad job) | 0.20 | 0.20 | 0.00 | 0.140 | 0.14 | 🔴 Low |
| Admin-boosted (+2) medium employer | — | — | — | 0.58 | 0.78 | 🟢 High |
| Admin-reduced (-2) medium employer | — | — | — | 0.58 | 0.38 | 🔴 Low |
