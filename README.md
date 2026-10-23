# LinkedIn Lead Finder (v2.0)

A LinkedIn profile scraper and lead-finding tool.

## Why it was built

Someone asked me if there was a way to find people on LinkedIn who know people in a specific industry. LinkedIn search only lets you filter on preset fields like current company, connection degree and job title, so a lot of relevant people never show up. I thought about how to get around that and built this.

The idea:

1. Collect full profile data (about, experience, education) from LinkedIn.
2. Store it in a structured database with semantic embeddings.
3. Run search and LLM-based analysis over that data to find people that LinkedIn's own filters miss.

## What it does

- **Finds profile links** from LinkedIn people search results, company pages and your own connections list.
- **Scrapes profiles** with Playwright: the intro panel (name, title, location), About, Experience and Education sections.
- **Saves raw HTML first**, then parses it in a separate step. Scraping and parsing are decoupled, so the parser can be fixed or improved without hitting LinkedIn again.
- **Extracts structured data**: name, title, location, positions, institutions and dates. Dates in different LinkedIn formats are normalized.
- **Creates semantic embeddings** (`all-MiniLM-L6-v2`, 384 dimensions) for About, Experience and Education text, so profiles can be searched by meaning instead of exact keywords.
- **Stores everything in Supabase (Postgres)** across normalized tables.
- **Avoids getting blocked**: a burst-style rate limiter that mimics human browsing, a daily request cap, user-agent rotation and reuse of an existing logged-in session.

### Chrome extension

I also built a Chrome extension that captures the LinkedIn profiles you view while browsing normally. Instead of running a bot against LinkedIn, it collects data passively from pages you already opened, which is safer for the account and needs no separate login automation.

## Architecture

```
Profile link discovery ──▶ Profile scraper ──▶ raw_linkedin_data
(search / company /          (Playwright)            │
 connections)                                        ▼
                                     Extractor (parse HTML, normalize dates)
                                                     │
                                                     ▼
                                Embeddings (sentence-transformers)
                                                     │
                                                     ▼
              linkedin_profiles · linkedin_about · linkedin_experiences
                     linkedin_educations · institutions   (Supabase)
```

The code is split into layers behind interfaces, so each part can be swapped out:

| Folder | Role |
|---|---|
| `src/browser/` | Playwright browser wrapper behind a `ScraperInterface` |
| `src/session/` | LinkedIn login and session handling |
| `src/scraper/` | Profile link scraper and people/profile scraper |
| `src/selectors/` | All LinkedIn CSS selectors in one place |
| `src/extractors/` | Turns raw HTML into structured data |
| `src/services/` | Scraping service and data extraction pipeline |
| `src/repository/` | Supabase repository behind a `RepositoryInterface` |
| `src/models/` | Dataclasses for each database table |
| `src/utils/` | Rate limiter, user-agent rotator, date normalizer, vectorization |
| `examples/` | Runnable scripts for each step |

## Tech stack

Python (asyncio) · Playwright · Supabase / Postgres · sentence-transformers · LangChain · fake-useragent

## Getting started

```bash
python -m venv .venv && source .venv/bin/activate
pip install playwright supabase sentence-transformers langchain-community fake-useragent
playwright install chromium

cp .env.example .env   # add LinkedIn and Supabase credentials
```

Set these environment variables:

```
LINKEDIN_USERNAME=...
LINKEDIN_PASSWORD=...
SUPABASE_URL=...
SUPABASE_KEY=...
```

Run the pipeline step by step:

```bash
python examples/extract_profile_links.py        # 1. find profile URLs from a search
python examples/extract_connection_links.py     # or collect from your connections
python examples/dump_profiles_database.py       # 2. scrape profiles, store raw HTML
python examples/example_linkedin_data_extractor.py  # 3. parse, embed and store
```

## Status

Working prototype. Not under active development.

## Disclaimer

Built for personal lead research. Automated scraping may break LinkedIn's Terms of Service. Use it responsibly, at low volume, and only on accounts you own.
