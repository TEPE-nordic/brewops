# BrewOps

Telemetry and operations app for the office coffee machines. Python (FastAPI) + stdlib `sqlite3` (no ORM), vanilla JS frontend (no build step).

## Architecture

Data flows one direction: **ingest → db → api → frontend**.

- **ingest** (`src/brewops/ingest/`) — `loader.py` parses and validates CSV files, `cli.py` provides the `ingest`/`seed` console scripts.
- **db** (`src/brewops/db/`) — `schema.py` defines tables + seed reference data (`machines`, `drink_types`); `connection.py` opens the SQLite file (`$BREWOPS_DB` or `./brewops.db`); `queries.py` is the only place that writes SQL — no query building elsewhere.
- **api** (`src/brewops/api/main.py`) — FastAPI routes under `/api/*`, thin wrappers around `queries.py`. GET endpoints return raw dicts from queries (no Pydantic response models); POST bodies use Pydantic models (`BrewIn`, `MaintenanceIn`) for validation. Also serves the frontend as static files at `/`.
- **frontend** (`src/brewops/frontend/`) — `index.html` + `app.js`, fetches `/api/*` and renders dashboard, machine cards, and the two logging forms. No framework, no build step.

## Ingestion paths

Every `brew_events` row has `source` = `'csv'` or `'manual'`:

1. **CSV** — `uv run ingest [path]` (default `data/inbox`) or `uv run seed` (wipes + reseeds, then ingests `data/inbox`). Filename prefix selects the row type: `brews_*.csv` (source=csv), `manual_*.csv` (source=manual — exports of the paper log kept next to machines without telemetry, e.g. Old Faithful), `maintenance_*.csv`. Bad rows are rejected and reported per-line; the rest of the file still loads.
2. **Manual entry** — the frontend's "Log a brew" / "Log maintenance" forms POST to `/api/brews` / `/api/maintenance`, always recorded with `source='manual'`.

Both paths validate against the same reference data (`machines`, `drink_types`) and go through the same `queries.py` insert functions.

## Running and testing

```
uv run seed     # rebuild the db from scratch and load data/inbox
uv run start    # serve the app at http://localhost:8123
uv run ingest [path]   # load more CSVs without wiping existing data
uv run pytest   # run the test suite (tests/test_db.py, test_api.py, test_ingest.py, test_frontend.py)
```

API tests use a custom in-process ASGI client (`tests/asgi_client.py`), not `TestClient`/`httpx`. Each test gets an isolated DB via the `db` fixture (`tmp_path` + `BREWOPS_DB` env var).

Known environment quirk: on some Windows setups `uv run start` fails to spawn with `Access is denied (os error 5)`. Workaround: `uv run python -c "from brewops.api.main import run; run()"`.

## Conventions

- Timestamps are naive local time, always `'YYYY-MM-DD HH:MM:SS'` in storage; the API also accepts HTML `datetime-local` format (`YYYY-MM-DDTHH:MM`) and normalizes it. Future timestamps are rejected.
- Drinks and maintenance types are closed vocabularies: `drink_types.name` (machine key, e.g. `espresso`) vs `.label` (display text, e.g. `Espresso`) — use the label in anything user-facing. Maintenance `type` is a DB `CHECK` constraint (`descale`, `refill`, `repair`, `error`), duplicated as a validation list in `main.py`.
- `queries.py` functions take a `sqlite3.Connection` as their first argument and return plain dicts/lists (via `dict(row)`), never raw `sqlite3.Row` objects, past the query layer.
