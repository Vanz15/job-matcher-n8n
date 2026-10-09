<div align="center">

# job-matcher-n8n

*An LLM-scored job pipeline that filters a resume against live PH job postings*

</div>

An [n8n](https://n8n.io) workflow that scrapes job boards, scores every posting against your resume with an LLM (1–100 fit score), drops anything under threshold, and extracts structured hiring signals for what survives — written straight to Google Sheets.

## How it works

```
Manual Trigger
  └─> Resume Text ............. your resume (summary + full text)
  └─> Fetch Jobs (Apify) ..... JobStreet PH scrape, by search query
  └─> Wait ................... rate-limit backoff before the LLM call
  └─> Trim Job Fields ........ keep title / company / location / url / description + type
       ├─> Get Existing URLs .. read URLs already in the Matches tab
       │    └─> Dedupe Jobs ... drop anything seen before  ← saves tokens
       └─> Aggregate ........... collapse the batch into ONE array
            └─> Score Fit ...... Gemini scores every job in a single call
                 └─> Parse Scores Array
                      └─> Merge ... re-join scores to job data by index
                           └─> Fit Score Gate (≥ 80)
                                ├─ true  ─> Deep Analysis ─> Parse ─> Matches tab
                                └─ false ─> Format Skipped ───────────> Skipped tab
```

Two things make this cheap enough to run on a free tier:

- **One batched LLM call, not one per job.** A run fetches 15 postings and scores all 15 in a single request — 1 call instead of 15, a ~93% reduction.
- **Dedupe before scoring.** Previously-seen URLs are filtered out before any tokens are spent on them.

### The two-tier prompt

The wide pass and the narrow pass ask different questions:

| Pass | Runs on | Prompted with | Returns |
| --- | --- | --- | --- |
| **Score Fit** | every job, batched | `resume_summary` | `fit_score` (1–100) + one-line reasoning |
| **Deep Analysis** | only the ≥ 80 matches | the job description alone | required skills, ATS keywords, seniority, years of experience, nice-to-have, culture signals |

Scoring needs the resume; signal extraction doesn't — it reads the posting on its own terms. Keeping the second pass resume-free also keeps the per-match prompt small.

## Setup

1. **Import** — in n8n, *Workflows → Import from File* and pick `job hunter.json`.
2. **Credentials** — you need three:
   - **Apify** — API token. The scrape uses the `blackfalcondata/jobstreet-scraper` actor, called via a plain HTTP Request node.
   - **Header Auth** — your LLM API key, for the `Score Fit` and `Deep Analysis` nodes.
   - **Google Sheets** — a Service Account, for the two append nodes.
3. **Resume** — open the `Resume Text` node and replace `resume_summary` (the condensed version the scorer sees) and `resume_text` (your full resume, kept in the workflow for your own reference).
4. **Target roles** — edit `searchQuery` and `location` in `Fetch Jobs - Apify`. The Apify actor is called with a single query per run; loop the node if you want several roles at once.
5. **Sheet** — create a Google Sheet with two tabs named `Matches` and `Skipped`, then repoint the three Google Sheets nodes at it.

> [!TIP]
> Exported workflow JSON carries **credential IDs, not secrets** — your keys are safe. But it *does* bake in the original Sheet ID and the resume text. Strip those before sharing publicly.

### What gets written to Sheets

**Matches** (score ≥ 80) — Title, Company, Location, URL, Fit Score, Reasoning, Required Skills, ATS Keywords, Years Experience, Nice to Have, Culture Signals, Seniority.

**Skipped** — Title, Company, URL, Fit Score, Reasoning. Worth keeping: it's the easiest way to tune your threshold and see what you're filtering out.

> [!NOTE]
> The `Matches` tab header has a typo inherited from the export — `Culture SIgnals`. Rename it to `Culture Signals` in your sheet, or the append will fail on an unmatched column.

## Tuning

Most of the behavior lives in three places:

| What | Where |
| --- | --- |
| Score threshold | `Fit Score Gate` — the `80` in the comparison |
| Search targets | `searchQuery` / `location` in `Fetch Jobs - Apify` |
| How many postings | `maxResults` in the Apify request body |

Both LLM calls run at `temperature: 0.3` to keep scoring consistent across runs.

> [!NOTE]
> Scoring output is parsed defensively: a truncated or malformed array falls back to a repair pass (closing the last complete object) instead of failing the batch. The deep-analysis parse is stricter — a bad object there will throw, so it's the one node worth watching if a run errors.

## Stack

n8n (self-hosted, Docker on AWS EC2) · Apify · Gemini API · Google Sheets API

## Why it exists

Built as a learning project to get hands-on with n8n, cloud infrastructure, and LLM integration patterns — in particular working through real free-tier rate limits across Groq, Gemini, OpenRouter, and DeepSeek, and designing the batching and dedupe layers above to route around them.