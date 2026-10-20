# Correra

A reading curriculum generator. Users submit a topic, a time budget, and a difficulty level, and the app returns a five-book ordered curriculum with per-book reasoning. Check-ins on each book feed back into future recommendations.

## What it does

Three free-text inputs produce a ranked reading list with written justifications. As users mark books as finished, abandoned, or in progress, the system tightens or loosens difficulty, excludes books they have already engaged with, and lets them click through to a related curriculum built around an author they finished.

The design optimizes for completion rate rather than user preference. A one-shot LLM query cannot replicate behavioral completion data, persistent check-ins across sessions, or the hard constraint that every pick must exist in a real catalog.

## Architecture

Offline:
- UCSD Goodreads dataset loaded into BigQuery via `load_to_bigquery.sh`.
- Reddit posts from r/books, r/suggestmeabook, and r/literature pulled via public JSON endpoints into BigQuery, then enriched with per-post book mention extraction via Gemini. Stored in `v_reddit_signal`.

Request time:
1. Reflex form captures the three fields and an optional already-read list.
2. `pipeline.generate_for_survey` orchestrates the call.
3. `extract.extract_filters` calls Gemini Flash Lite to turn free text into typed filter parameters.
4. If the user has prior check-ins, `_difficulty_bias_for` nudges the difficulty and page range based on past feedback.
5. `candidates.fetch_candidates` runs a parameterized BigQuery query against 2.3 million books, filtered by language, page range, keyword overlap with shelf metadata, and the user's exclusion list. Reddit signal joins in as a tiebreaker.
6. `curriculum.generate_curriculum` calls Gemini Flash with the top 15 candidates and returns a five-pick ordered curriculum.
7. The result persists to SQLite as a Curriculum row with five Progress rows.
8. A per-user cache keyed on normalized survey text short-circuits repeat requests when no past engagement exists.

User data lives in a local SQLite database managed by SQLModel and Alembic. Books live in BigQuery. The two never mix.

## Stack

| Layer | Tool |
|---|---|
| Frontend and backend | Reflex |
| LLM | Vertex AI Gemini (2.5 Flash for curation, 2.5 Flash Lite for extraction) |
| Books catalog | BigQuery (`molbydickproj.goodreads.books`) |
| Reddit signal | BigQuery (`reddit_posts`, `reddit_book_mentions`, `v_reddit_signal`) |
| User state | SQLite via SQLModel plus Alembic migrations |
| Experiment tracking | MLflow on Cloud Run (separate service for the TF-IDF recommender) |
| CI | GitHub Actions, nightly cron |

## Setup

Requires Python 3.11+, Node 20+, and the Google Cloud SDK.

```
git clone https://github.com/SaiMulagan/mobydick.git
cd mobydick
pip install -r requirements.txt
gcloud auth application-default login
gcloud config set project molbydickproj
reflex db migrate
```

The gcloud application default credentials need `roles/aiplatform.user` on the project.

## Running

Local Reflex app:

```
reflex run
```

Opens at `http://localhost:3000` with the backend on port 8000.

Command line interface for backend-only testing:

```
python main.py --genre "Russian literature" --time "evening" --difficulty "challenging"
```

Reddit ingestion refresh (optional, the demo project already has data loaded):

```
python reddit_fetch.py
python reddit_extract.py
./load_to_bigquery.sh
```

## Project layout

```
config.py                 Project IDs, model names, BigQuery table references
schema.py                 Pydantic models for survey filters and curricula
extract.py                Gemini call 1: text to structured filters
candidates.py             Parameterized BigQuery candidate query
curriculum.py             Gemini call 2: candidates to five-book curriculum
pipeline.py               Orchestration, caching, difficulty bias, persistence
main.py                   argparse CLI
db.py                     SQLModel definitions for User, Curriculum, Progress, ReadBook
reddit_fetch.py           Reddit post ingestion
reddit_extract.py         Per-post book mention extraction
load_to_bigquery.sh       Loads Goodreads and Reddit data into BigQuery
app.py                    FastAPI inference server for the TF-IDF BookRecommender
train.py                  TF-IDF recommender training, registers with MLflow
Dockerfile                Container image for the inference service
mlflow-server/            MLflow tracking server image for Cloud Run
test_nightly.py           29 pytest cases, mocks google.cloud and mlflow
.github/workflows/        Nightly CI
alembic/                  Database migrations
moby/                     Reflex app package
  state.py                State fields and event handlers
  moby.py                 UI components, pages, theme
rxconfig.py               Reflex configuration
context/                  Architecture, product, and decision documents
```

## Features

The user interface has three pages.

**Generate** (`/`) renders the survey form. A pseudo-progress ticker runs under the spinner while the curriculum generates.

**Dashboard** (`/dashboard`) renders one active curriculum in a Kindle-style layout. The featured pick spans the full width; the remaining four fill a 2x2 grid. A segmented progress bar at the top uses solid wine for finished picks, diagonal stripes over cream for in-progress picks, dusty rose for abandoned, and muted cream for not started. Status, rating, difficulty, and comment controls live on every card.

**Your curricula** (`/curricula`) is a collapsible accordion list of every curriculum the user has generated, split into In Progress and Finished tabs. Each row carries a trash icon for deletion. Expanding a row shows the full curriculum and a button to promote it to the Dashboard tab.

Personalization is cookie-based. First visit creates a UUID. Books the user has engaged with never resurface in a new curriculum. Difficulty bias from past check-ins deterministically shifts the extracted filters before the second Gemini call.

## Testing

```
pytest test_nightly.py
```

29 test cases across 9 classes cover schema validation, pipeline error paths, FastAPI endpoint contracts, and the prompt-injection guard on the extraction step. Tests replace `google.cloud` and `mlflow` with MagicMock before any project module imports them, so no real API calls fire. The nightly GitHub Actions workflow runs the same suite at 06:00 UTC and surfaces failures as annotations on the run.

## Deployment

The MLflow tracking server and the TF-IDF BookRecommender service run on Cloud Run. The MLflow tracking server uses Cloud SQL Postgres for the metadata store and GCS for artifacts. The BookRecommender image bakes a self-contained model copy at build time so inference does not depend on the tracking server at request time.

The Reflex application currently runs locally. Cloud Run deployment is the next step.

## License

Educational project. The UCSD Goodreads dataset carries its own license and should not be redistributed from this repository. The `.gitignore` excludes the raw data files.

## Team

Aatish, Brandt, Sai. Advanced MLOps at USF, Spring 2026.

Demo:

https://github.com/user-attachments/assets/150933ef-470a-4b67-8fbc-04ac31453d5d
