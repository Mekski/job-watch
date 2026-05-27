# Dashboard

Local Flask app for job-watch with two pages: a connection-status dashboard (`/`) and a how-to tutorials reference (`/tutorials`). Read-only — it doesn't change any state, just surfaces what's connected and explains how to operate each piece.

## Why

The bot involves Telegram, Google Sheets, GCP service accounts, GitHub Actions, cron-job.org, and two state files. When something's off, finding the broken piece means clicking through five different services. The **dashboard** answers "is everything wired up?" in one page. The **tutorials** page is the native reference for common tasks (creating a Telegram bot, rotating the SA key, adding a new watched repo, renewing the PAT, etc.) so you don't have to dig through external docs.

## Setup

```sh
cd web
pip install -r requirements.txt
cp .env.example .env
# Edit .env with your real values (same secrets you set on GitHub Actions)
```

For `GOOGLE_SERVICE_ACCOUNT_JSON` you can either:
- Paste the entire JSON content into the variable (as one line, escaping internal newlines), OR
- Set it to a filesystem path like `/Users/you/path/to/sa.json` — the app will read the file

## Run

```sh
python app.py
```

Open http://localhost:5001.

The page is read-only; reload to re-check status. The app is stateless — every page load re-queries all the services live.

## Dashboard page (`/`)

Tile grid (responsive 1-4 columns by viewport width) with one tile per integration. Each tile has a Lucide-style SVG icon, an uppercase label, a status dot (green/yellow/red), a one-line summary, and a collapsible `details` section with deeper diagnostic info.

The seven tiles:

1. **Telegram bot** — bot identity (`getMe`), chat reachability (`getChat`), webhook status (webhook set = bad, breaks `getUpdates`)
2. **Sheets** — sheet auth, title, `Applications` tab presence, all-tabs list
3. **Service account** — SA email + project for verifying you shared the sheet correctly
4. **GitHub Actions** — last 10 workflow runs, success rate, freshness, link to runs page
5. **Trigger cadence** — inferred from how recent the last GH run was (green if <90s, yellow ≤5min, red beyond)
6. **Watched sources** — for each upstream source (Simplify, Vansh): last snapshot SHA, row count, and whether upstream has advanced past us
7. **Bot state** — pending vs applied counts from `.bot_state.json`, last applied timestamp, Telegram offset

Tiles use CSS Grid with `align-items: start` so expanding one tile's details doesn't reflow the others.

## Tutorials page (`/tutorials`)

Long-scroll reference with a pill-style TOC at the top linking to per-integration sections. Each section has an icon-prefixed heading and a list of expandable how-to steps (numbered, with code blocks where useful). Currently covers ~29 tasks across the seven integrations — adding a new watched repo, finding section markers for a new source, rotating the SA key, debugging cron-job.org outages, manually fixing the state file, etc.

Reuses the same per-integration tutorial partials in `templates/tutorials/{slug}.html` that get embedded in the page.

## What it doesn't do (yet)

- No tables of individual pending/applied jobs
- No live sheet contents preview (you can just open the sheet)
- No funnel chart from your sheet's Status column
- No runs history table beyond the last-10 summary
- No interactive setup wizard / write operations (long-term: forms that validate as you fill them in)
- No auto-refresh — manual page reload

## Troubleshooting

**Dashboard won't start (ModuleNotFoundError):** you forgot `pip install -r requirements.txt` or you're in the wrong venv.

**Everything is red:** probably your `.env` didn't load. Check you copied `.env.example` to `.env` and you're running `python app.py` from the `web/` directory.

**GitHub API rate-limit errors:** add a `GITHUB_TOKEN` to `.env` for 5000 req/hr instead of 60. Any GitHub token works (even a fresh fine-grained one with no scopes).

**Sheets card red with HTTP 403:** you didn't share the sheet with the SA email. The GCP card shows the email — share the sheet with that address as Editor.

**Sheets card red with HTTP 404:** wrong sheet ID. Open your sheet, copy the ID from the URL (`docs.google.com/spreadsheets/d/<ID>/edit`).

**Watched-repos card shows "not bootstrapped":** the bot has never run for that source. Trigger the workflow once manually from the GitHub Actions tab.
