# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Setup

```bash
pip install -e ".[dev]"
playwright install chromium
```

Also install the separately-packaged discovery dependency (pins numpy exactly, requires `--no-deps` workaround):
```bash
pip install --no-deps python-jobspy && pip install pydantic tls-client requests markdownify regex
```

## Commands

```bash
# Run all tests
pytest tests/ -v

# Run a specific test file
pytest tests/test_scoring.py -v

# Run with coverage
pytest tests/ --cov=src/applypilot --cov-report=term-missing

# Lint
ruff check src/

# Auto-fix linting issues
ruff check src/ --fix

# Format
ruff format src/

# Verify installation / check missing deps
applypilot doctor
```

## Architecture

### User Data vs Package Config

All user data lives in `~/.applypilot/` (overridable via `APPLYPILOT_DIR` env var):
- `applypilot.db` — SQLite database (single source of truth for all job state)
- `profile.json` — contact info, preferences, resume facts
- `searches.yaml` — job search queries and targets
- `.env` — API keys (`GEMINI_API_KEY`, `OPENAI_API_KEY`, `LLM_URL`, `CAPSOLVER_API_KEY`)
- `tailored_resumes/`, `cover_letters/` — per-job AI-generated files

Package-shipped YAML registries live in `src/applypilot/config/`:
- `employers.yaml` — 48 Workday employer portals
- `sites.yaml` — direct career sites, blocked sites, manual ATS domains, base URLs

### Pipeline Stages

Six sequential stages, each independent and resumable. Defined in `pipeline.py`:

1. **discover** → `discovery/jobspy.py` (JobSpy boards), `discovery/workday.py` (Workday portals), `discovery/smartextract.py` (direct sites with CSS selectors + AI fallback)
2. **enrich** → `enrichment/detail.py` — 3-tier cascade: JSON-LD → CSS selectors → AI extraction
3. **score** → `scoring/scorer.py` — LLM rates each job 1-10 against your profile
4. **tailor** → `scoring/tailor.py` — LLM rewrites resume per job; `scoring/validator.py` validates output
5. **cover** → `scoring/cover_letter.py` — LLM generates cover letter per job
6. **pdf** → `scoring/pdf.py` — converts `.txt` outputs to PDF

Stages are wired in `pipeline.py` with `_STAGE_RUNNERS`, `_UPSTREAM` (dependency graph), and `_PENDING_SQL` (DB queries that determine if a stage has remaining work). Streaming mode (`--stream`) runs all stages as concurrent threads, polling the DB as a conveyor belt.

### Database

Single SQLite table `jobs` with columns for every stage. Schema is defined in `database.py:_ALL_COLUMNS`. Migrations are additive only — `ensure_columns()` adds missing columns via `ALTER TABLE` on every startup. Thread-local connections with WAL mode for concurrent workers.

### LLM Client (`llm.py`)

Singleton `LLMClient` auto-detects provider from env vars: `GEMINI_API_KEY` → Gemini, `OPENAI_API_KEY` → OpenAI, `LLM_URL` → local (Ollama/llama.cpp). `LLM_MODEL` overrides the model name. On Gemini 403 errors (model not on OpenAI-compat layer), automatically falls back to the native `generateContent` API. Includes exponential backoff for 429/503 rate limits.

### Tier System

Feature gating in `config.py`:
- **Tier 1**: Discovery only (no LLM key)
- **Tier 2**: + LLM API key → scoring, tailoring, cover letters
- **Tier 3**: + Claude Code CLI + Chrome → auto-apply

`check_tier(n, feature)` is called at the top of LLM-dependent commands and exits with a clear message if requirements aren't met.

### Auto-Apply (`apply/`)

- `launcher.py` — orchestrates parallel Chrome workers, manages the apply queue from the DB
- `chrome.py` — Chrome process management, CDP port allocation, worker isolation
- `prompt.py` — builds the Claude Code prompt injected per job (includes profile, tailored resume, cover letter, application URL)
- `dashboard.py` — real-time Rich TUI dashboard showing worker status

Auto-apply spawns Claude Code as a subprocess (`claude --model haiku -p ...`) with a Playwright MCP server configured per-worker at runtime. No manual MCP setup required.

### Validation (`scoring/validator.py`)

Three modes: `strict` (banned words = errors, LLM judge must pass), `normal` (banned words = warnings, default), `lenient` (skip validation entirely). Relevant for tailor and cover stages to prevent AI-fabricated content.

## Code Style

- Type hints required on all function signatures
- Docstrings required on all public functions and classes (Google style)
- Line length: 120 characters (Ruff config)
- When adding a new DB column, add it to `_ALL_COLUMNS` in `database.py` — that's the only migration needed
