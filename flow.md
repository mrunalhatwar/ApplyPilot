## `applypilot init`

**Entry:** `cli.py:69` → `wizard/init.py:run_wizard()`

Runs an interactive 5-step setup wizard that creates `~/.applypilot/`:

| Step | Function | What it does |
|------|----------|--------------|
| 1 | `_setup_resume()` | Copies your `.txt`/`.pdf` resume to `~/.applypilot/resume.txt` |
| 2 | `_setup_profile()` | Prompts for name, email, skills, experience → saves `profile.json` |
| 3 | `_setup_searches()` | Prompts for roles + location → writes `searches.yaml` |
| 4 | `_setup_ai_features()` | Optionally configures `GEMINI_API_KEY`/`OPENAI_API_KEY` → writes `.env` |
| 5 | `_setup_auto_apply()` | Checks for Claude Code CLI; optionally saves `CAPSOLVER_API_KEY` |

At the end, it detects your current tier (1/2/3) and shows what's unlocked.

---

## `applypilot run`

**Entry:** `cli.py:77` → `_bootstrap()` → `pipeline.py:run_pipeline()`

`_bootstrap()` runs first: loads `.env`, creates dirs, inits the SQLite DB.

Then `run_pipeline()` executes 6 stages in order (sequential by default, or concurrent with `--stream`):

```
discover → enrich → score → tailor → cover → pdf
```

| Stage | File called | What it does |
|-------|-------------|--------------|
| **discover** | `discovery/jobspy.py`, `discovery/workday.py`, `discovery/smartextract.py` | Scrapes job boards (LinkedIn/Indeed via JobSpy), 48 Workday portals, and direct career sites → inserts rows into SQLite `jobs` table |
| **enrich** | `enrichment/detail.py` | Fetches full job description for each discovered job (JSON-LD → CSS selectors → AI fallback) |
| **score** | `scoring/scorer.py` | Sends each job + your profile to the LLM → stores `fit_score` (1–10) in DB |
| **tailor** | `scoring/tailor.py` | For jobs with score ≥ `--min-score`, LLM rewrites your resume for that job → saved to `tailored_resumes/` |
| **cover** | `scoring/cover_letter.py` | Generates a cover letter per tailored job → saved to `cover_letters/` |
| **pdf** | `scoring/pdf.py` | Converts `.txt` outputs to PDF |

**Streaming mode** (`--stream`): each stage runs in its own thread, polling the DB for new work produced by upstream stages — acts as a conveyor belt. The dependency order is enforced via `_UPSTREAM` and `_PENDING_SQL` in `pipeline.py`.
