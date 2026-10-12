# Job Search Assistant

A Python-based pipeline that turns LinkedIn job-alert emails in Gmail into a clean, deduplicated, searchable SQLite database of job postings.

> **Status:** Phase 1 (Database Creation) is complete. Later phases are planned and listed in the [Roadmap](#roadmap).

---

## Table of Contents

- [Overview](#overview)
- [How It Works](#how-it-works)
- [Features](#features)
- [Tech Stack](#tech-stack)
- [Repository Structure](#repository-structure)
- [Getting Started](#getting-started)
- [Usage](#usage)
- [Database Schema](#database-schema)
- [Configuration](#configuration)
- [Known Limitations](#known-limitations)
- [Roadmap](#roadmap)
- [Security & Privacy](#security--privacy)

---

## Overview

Job-alert emails pile up quickly and are hard to review, compare, or search. Job Search Assistant connects to your Gmail inbox (read-only), finds LinkedIn job-alert emails, extracts each posting, standardizes the data, and stores it in a local SQLite database. A CSV export is also generated for easy human review.

## How It Works

```
Gmail  →  Python  →  Job Parser  →  SQLite  →  CSV
```

1. **Authenticate** with the Gmail API using OAuth 2.0 (read-only scope).
2. **Search** the inbox using a configurable Gmail query (default: `newer_than:7d (LinkedIn)`).
3. **Retrieve and decode** each matching message, including nested MIME parts and Base64URL-encoded bodies.
4. **Parse** the HTML with BeautifulSoup to extract the title, company, location, and job link.
5. **Normalize** each URL to a clean `linkedin.com/comm/jobs/view/<id>/` form and extract the numeric LinkedIn job ID.
6. **Deduplicate** records by job ID.
7. **Insert** new jobs into SQLite, skipping any that already exist.
8. **Export** the database to a CSV file for review.

## Features

- Read-only Gmail access via OAuth 2.0, with token caching and automatic refresh
- Robust recursive extraction of plain-text and HTML email bodies
- Standardized job record format with a consistent schema
- Duplicate protection at two levels: in-memory by LinkedIn job ID, and in the database via a `UNIQUE` constraint on `url`
- Repeatable, idempotent ingestion: safe to re-run at any time
- Built-in validation checks (duplicate IDs, duplicate URLs, missing required fields)
- CSV export for quick review in Excel or Google Sheets

## Tech Stack

| Area | Tools |
| --- | --- |
| Language | Python 3.13 |
| Email access | Gmail API (`google-api-python-client`, `google-auth-httplib2`, `google-auth-oauthlib`) |
| HTML parsing | BeautifulSoup 4 |
| Data handling | pandas |
| Storage | SQLite (`sqlite3`) |
| Environment | Jupyter Notebook |

## Repository Structure

```
job-search-assistant/
├── Job_Assistant_Phase_1.ipynb   # Phase 1 pipeline, step by step
├── .gitignore                    # Excludes credentials, tokens, and local data
└── README.md
```

The following files are created locally when you run the notebook and are intentionally excluded from version control:

```
credentials.json        # Your Google OAuth client secrets
token.json              # Generated after first authorization
data/jobs.db            # SQLite database
data/jobs_review.csv    # CSV export for review
```

## Getting Started

### Prerequisites

- Python 3.9 or newer (developed on 3.13)
- Jupyter Notebook or JupyterLab
- A Google account that receives LinkedIn job-alert emails
- A Google Cloud project with the Gmail API enabled

### 1. Clone the repository

```bash
git clone https://github.com/patelrushik15/job-search-assistant.git
cd job-search-assistant
```

### 2. Install dependencies

```bash
pip install google-api-python-client google-auth-httplib2 google-auth-oauthlib beautifulsoup4 pandas jupyter
```

### 3. Set up Gmail API access

1. Open the [Google Cloud Console](https://console.cloud.google.com/) and create (or select) a project.
2. Enable the **Gmail API**.
3. Configure the OAuth consent screen and add your account as a test user.
4. Create an **OAuth client ID** of type **Desktop app**.
5. Download the client secrets file, rename it to `credentials.json`, and place it in the project root.

The first run opens a browser window to authorize access. A `token.json` file is then saved so you will not need to re-authorize each time.

## Usage

1. Launch Jupyter and open `Job_Assistant_Phase_1.ipynb`:

```bash
   jupyter notebook
```

2. Run the cells in order. The notebook is organized into numbered parts:

   | Part | Purpose |
   | --- | --- |
   | 1 | Import libraries |
   | 2 | Project configuration (paths, scopes, Gmail query) |
   | 3 | Gmail API authentication |
   | 4 | Search Gmail for LinkedIn alerts |
   | 5 | Retrieve a Gmail message |
   | 6 | Extract email body (text and HTML) |
   | 7 | LinkedIn job parsing (record format, URL normalization, job ID extraction, HTML parser) |
   | 8 | Gmail message processing |
   | 9 | Deduplication |
   | 10 | Database management (create table, insert, query) |
   | 11 | Export to CSV |
   | 12 | Repeatable ingestion function: `ingest_linkedin_jobs()` |
   | 13 | Final validation |
   | Appendix A | `job_id` column migration |

3. For routine use, run the setup cells once per session and then call:

```python
   ingest_linkedin_jobs()
```

   This searches Gmail, parses new alerts, and inserts only jobs not already in the database.

## Database Schema

Jobs are stored in the `jobs` table of `data/jobs.db`.

| Column | Type | Notes |
| --- | --- | --- |
| `id` | INTEGER | Primary key, auto-increment |
| `source` | TEXT | Required (currently always `LinkedIn`) |
| `company` | TEXT | Required |
| `title` | TEXT | Required |
| `location` | TEXT | Required |
| `description` | TEXT | Reserved for later phases |
| `url` | TEXT | Required, **unique** |
| `salary` | TEXT | Reserved for later phases |
| `remote_status` | TEXT | Reserved for later phases |
| `date_posted` | TEXT | Reserved for later phases |
| `date_found` | TEXT | Required, UTC ISO 8601 timestamp |
| `job_id` | TEXT | LinkedIn numeric job ID, added by the Appendix A migration |

> **Note:** The `job_id` column was added after the initial table design. If you are starting with a fresh database, run the `add_job_id_column()` migration in Appendix A (or add the column to the `CREATE TABLE` statement) before inserting records. The Appendix A code cells are stored as raw cells and must be converted to code cells to run.

## Configuration

Settings live in **Part 2** of the notebook:

| Setting | Default | Description |
| --- | --- | --- |
| `GMAIL_QUERY` | `newer_than:7d (LinkedIn)` | Gmail search query used to find job alerts |
| `GMAIL_SCOPES` | `gmail.readonly` | Read-only access to your mailbox |
| `DATA_DIR` | `./data` | Folder for the database and CSV export |
| `max_results` | `50` | Maximum emails retrieved per run |

## Known Limitations

- Only LinkedIn job-alert emails are supported.
- Parsing depends on LinkedIn's current email HTML layout and may need updates if it changes.
- Some titles can include trailing badge text from the email (for example "2 company alumni" or salary ranges). Cleaning these is a candidate for a future improvement.
- Fields such as `description`, `salary`, `remote_status`, and `date_posted` are not yet populated.
- The pipeline currently runs from a notebook rather than a standalone script or scheduler.

## Roadmap

- [x] **Phase 1: Database creation.** Gmail → Python → Job Parser → SQLite
- [ ] **Phase 2: Resume and job matching**
- [ ] **Phase 3: Tailored resume generation**
- [ ] **Phase 4: Email report with PDF attachments**
- [ ] **Phase 5: Automation and production deployment**

## Security & Privacy

- Access is **read-only** (`gmail.readonly`); the application never modifies or sends email.
- `credentials.json`, `token.json`, and all generated data are listed in `.gitignore`. **Never commit these files.**
- All data is stored locally on your machine.
