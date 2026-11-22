# FrontierAtlas

# FrontierAtlas Intelligence Ingestion Demo

An asynchronous, source-traceable ingestion pipeline for AI startups, products, research papers, jobs, and news. The project is designed around the assessment's six phases and the schemas in the supplied brief.

> **Data integrity note:** This repository never manufactures entity records to reach a row target. Every accepted row contains a source URL. This workspace had no outbound DNS/network access during the build, so the included workbook is an empty six-tab template and the 1,000-row quotas have **not** been claimed as completed. Run the pipeline from a network-enabled machine with the required credentials to populate it.

## Table of contents

- [Deliverables](#deliverables)
- [Architecture](#architecture)
- [Quick start](#quick-start)
- [Run the pipeline](#run-the-pipeline)
- [Database configuration](#database-configuration)
- [Collection behavior and source coverage](#collection-behavior-and-source-coverage)
- [Freshness contract](#freshness-contract)
- [LLM extraction and reliability](#llm-extraction-and-reliability)
- [Anti-bot policy](#anti-bot-policy)
- [Outputs](#outputs)
- [Google Sheets publication](#google-sheets-publication)
- [GitHub publication](#github-publication)
- [Testing](#testing)
- [Design sources and API documentation](#design-sources-and-api-documentation)

## Deliverables

- `src/frontieratlas/`: asynchronous collectors, date normalizer, LLM fallback and chunking, entity resolver, storage, and export code.
- `config/sources.yaml`: five AI-news feeds and five job-board endpoints.
- `data/seed_entities.json`: seed list for deterministic organization resolution.
- `architecture.pdf` and `architecture.md`: production design (three-page PDF).
- `output/`: CSV export targets for the six Google Sheets tabs, a freshness review queue, and run manifests. These are created or populated by the first run.
- `FrontierAtlas_Output.xlsx`: formatted six-tab workbook generated for manual Google Sheets import.

## Architecture

```text
Directories / Public APIs / RSS / Job APIs
                 |
         Async source adapters
       (aiohttp + Playwright async)
                 |
       robots / host limits / retry
                 |
     raw HTML + API response + URL
                 |
    date normalization + freshness gate
                 |
     LLM extraction (Gemini -> Groq -> DeepSeek)
       semantic chunks; 413 split; 429 backoff
                 |
  Pydantic validation + entity resolution
                 |
 PostgreSQL (JSONB) + URL uniqueness + audit log
     |                  |                  |
  CSV/XLSX         pgvector embeddings       Neo4j
  Sheets export    semantic retrieval       entity graph
```

## Quick start

Python 3.10 or later is required.

### Windows PowerShell

```powershell
py -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -e .
playwright install chromium
Copy-Item .env.example .env
```

### Linux/macOS

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -e .
playwright install chromium
cp .env.example .env
```

Edit `.env` and add credentials. **A GitHub personal access token is recommended for a 1,000-paper run.** GitHub's unauthenticated REST quota is comparatively small; without a token, the code conservatively performs no more than 40 star lookups in one run, leaving other `github_stars` values null rather than inventing metrics.

```ini
GITHUB_TOKEN=...
GEMINI_API_KEY=...
GROQ_API_KEY=...
DEEPSEEK_API_KEY=...
PRODUCT_HUNT_TOKEN=...
PRODUCT_HUNT_TERMS_CONFIRMED=false
```

The LLM providers are optional. When configured, the extraction order is Gemini Flash, Groq Llama, then DeepSeek. Without LLM keys, deterministic parsers still run and optional enrichment remains null.

### Product Hunt legal gate

The Product Hunt connector is disabled until both `PRODUCT_HUNT_TOKEN` and `PRODUCT_HUNT_TERMS_CONFIRMED=true` are set. Product Hunt's published API terms say the API must not be used for commercial purposes without contacting Product Hunt. Set the flag only after confirming the assessment/demo use is permitted, or after obtaining written permission for commercial use. See the API terms in [Design sources and API documentation](#design-sources-and-api-documentation).

## Run the pipeline

Run the entire pipeline:

```bash
frontieratlas
```

Run a single vertical:

```bash
frontieratlas --only papers
frontieratlas --only startups
frontieratlas --only products
frontieratlas --only jobs
frontieratlas --only news
```

Controls are configurable through `.env`:

- `MAX_PAPERS`
- `MAX_STARTUPS`
- `MAX_PRODUCTS`
- `MAX_CONCURRENCY`
- `PER_HOST_CONCURRENCY`
- `FRESHNESS_HOURS`
- `LLM_MAX_CHARS`

Defaults are a 24-hour freshness window, strict date acceptance, and conservative per-host concurrency.

## Database configuration

The default database is local SQLite at `output/frontieratlas.db`.

For production, set:

```ini
DATABASE_URL=postgresql+asyncpg://user:password@host:5432/frontieratlas
```

The schema uses a unique `(record_type, source_url)` key and atomic conflict handling. `seen_items` stores first-seen and last-seen times; `crawl_state` stores source checkpoints and the last successful run. `entity_mapping_log` stores raw-to-canonical entity decisions.

## Collection behavior and source coverage

| Vertical | Sources / method | Freshness or provenance rule |
| --- | --- | --- |
| Startups | Y Combinator public company directory; asynchronous Playwright directory expansion and profile parsing | Company profile URL is stored as the source. `employeeCount` stays null unless present in the source. |
| Products | Product Hunt public GraphQL API | Token and terms confirmation required. Pricing model and startup relationship remain null unless directly evidenced by the source or LLM extraction. |
| Research | Papers with Code public API, `has_code=true`, `include_resources=true`; GitHub REST API for stars | Paper source URL, publication date, repository URL, and latest `stargazers_count` are captured. |
| News | TechCrunch AI, The Verge AI, Ars Technica AI, MIT Technology Review AI, The Batch | RSS discovery, full-text extraction, feed/JSON-LD/meta dates, and strict 24-hour check. |
| Jobs | Remote OK, Remotive, Arbeitnow, Jobicy, Himalayas | Board-specific adapters, AI keyword filter, and strict publication-date check. |

`Remotive` publicly documents that its feed is delayed by 24 hours. Because the task requires a *last 24 hours* guarantee, the freshness gate may reject its postings as too old; they are not backdated or relabelled as fresh. Source terms also require credit and linking for Remotive and Remote OK. Himalayas terms require use of its original `applicationLink`, source credit, removal of expired listings, and prohibit republishing its feed to other job aggregators. Review each provider's terms before redistribution.

## Freshness contract

1. Parse feed/API timestamps deterministically: ISO-8601, RFC-822, Unix epoch, and relative text (for example, `2 hours ago`).
2. When needed, inspect JSON-LD `datePublished`/`dateCreated`, article publication meta tags, and `<time datetime>`.
3. Accept into `Jobs` and `News` only if the timestamp is inside `[now - 24 hours, now]` (with a five-minute future-clock tolerance).
4. If no defensible publication timestamp exists, use a first-seen-since-last-run heuristic and write the item to `output/freshness_review.json` with `heuristicNewSinceLastRun`. **Do not** put it in the certified-fresh tabs. First-seen is not proof of publication time.

## LLM extraction and reliability

- Extraction is source-grounded: the prompt requires null for unavailable fields and prohibits guessing. Deterministic source URLs, collection timestamps, and publication dates are never accepted from the LLM.
- Content is split on headings and paragraphs before hard splitting. If a provider returns HTTP 413, the chain falls through and then recursively halves the chunk at a semantic boundary.
- HTTP 429 and 5xx errors use bounded exponential backoff with jitter; `Retry-After` is honoured by the common HTTP client where supplied.
- Invalid JSON or provider errors trigger the next provider. Pydantic models validate the canonical output.
- All rows retain a `source.url`; failed or blocked sources are recorded in the run-state table and logs.

## Anti-bot policy

This project does **not** bypass Cloudflare, DataDome, CAPTCHAs, access controls, or paywalls. It prefers documented APIs, RSS, sitemaps, and partner feeds. For JavaScript-rendered sites, it uses Playwright only where automated access is allowed by the site's terms and `robots.txt`. On 403, 429, CAPTCHA, or interstitial responses, it records the block, backs off, and requires a licensed API, publisher agreement, or operator allow-list for high-value sources. It does not use residential proxy rotation, CAPTCHA-solving services, or fingerprint evasion.

## Outputs

After a successful run, `output/` contains:

```text
Startups.csv
Products.csv
Research Papers.csv
Jobs.csv
News.csv
Entity Mapping Log.csv
data_manifest.json
freshness_review.json
run_summary.json
frontieratlas.db  # SQLite default
```

`FrontierAtlas_Output.xlsx` is provided as a six-tab workbook template for import. To populate it with live data, use the CSV exports as the source of truth and regenerate the workbook using the supplied workbook build instructions. The ChatGPT-generated assessment bundle includes the blank template only because this build session could not reach remote sources.

## Google Sheets publication

Google Drive/Sheets publishing is not available through the connected account in this session. After a network-enabled local run, install the optional Sheets dependencies and configure the environment variables:

```bash
python -m pip install '.[sheets]'
```

### Linux/macOS

```bash
export GOOGLE_SERVICE_ACCOUNT_FILE=/path/to/service-account.json
export GOOGLE_SHEET_ID=your_sheet_id
export PUBLISH_PUBLIC_SHEET=true
```

### Windows PowerShell

```powershell
$env:GOOGLE_SERVICE_ACCOUNT_FILE = "C:\path\to\service-account.json"
$env:GOOGLE_SHEET_ID = "your_sheet_id"
$env:PUBLISH_PUBLIC_SHEET = "true"
```

Run the uploader:

```bash
python scripts/publish_google_sheets.py
```

The script writes six tabs and only changes sharing to public viewer when `PUBLISH_PUBLIC_SHEET=true`. Keep the service-account JSON outside the repository.

## GitHub publication

This environment cannot authenticate to or push to your GitHub account. From the project folder, after reviewing secrets and `.gitignore`, run:

```bash
gh auth login
gh repo create frontieratlas-ingestion --public --source . --remote origin --push
```

Do not commit `.env`, database files, provider keys, service-account credentials, or scraped full-text data unless you have the right to redistribute that material.

## Testing

Install the test dependencies and run the test suite:

```bash
pip install -e '.[test]'
pytest -q
```

Tests cover alias resolution, relative and ISO date parsing, the 24-hour boundary, and semantic chunk size constraints. For integration validation, run each collector independently with API credentials and inspect `run_summary.json`, `data_manifest.json`, and `freshness_review.json`.

## Design sources and API documentation

- [Papers with Code public API](https://paperswithcode.co/docs/api)
- [Y Combinator company directory](https://www.ycombinator.com/companies)
- [GitHub REST API rate limits](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api)
- [Product Hunt API v2 and usage limitations](https://api.producthunt.com/v2/docs)
- [Remote OK public feeds](https://remoteok.com/faq)
- [Remotive public API and terms](https://remotive.com/remote-jobs/api)
- [Himalayas jobs API and terms](https://himalayas.app/api)
