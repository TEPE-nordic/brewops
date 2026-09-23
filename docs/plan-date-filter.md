# Implementation plan: dashboard date-range filter (ticket 005)

Read `tickets/005-date-filter.md` first. This plan assumes zero prior context —
every file, function, and parameter name below is exact. Do not implement
anything beyond what's described here without checking back.

## Summary

Let the dashboard be filtered to a date range. Three layers change:

1. `src/brewops/db/queries.py` — `get_stats` gains optional `start`/`end` args.
2. `src/brewops/api/main.py` — `/api/stats` gains optional `start`/`end` query params.
3. `src/brewops/frontend/index.html` + `app.js` — two date inputs + apply/reset,
   wired into `loadDashboard()`.

Dates are plain `YYYY-MM-DD` strings (no time component). `end` is inclusive —
"2026-06-02" includes all brews on June 2nd through 23:59:59.

Machine cards (`get_machine_health`, `/api/machines/{id}`, `renderMachineCards`)
are **out of scope**. They keep showing lifetime stats (specialty, busiest day,
last brew). Do not touch `get_machine_health`.

---

## 1. `src/brewops/db/queries.py` — `get_stats`

Current signature (line 70):
```python
def get_stats(conn: sqlite3.Connection) -> dict[str, Any]:
```

New signature:
```python
def get_stats(
    conn: sqlite3.Connection,
    start: str | None = None,
    end: str | None = None,
) -> dict[str, Any]:
```

`start`/`end` are `YYYY-MM-DD` strings or `None`. `None` means unbounded on
that side (matches current all-time behavior when both are `None`).

Build a `WHERE` fragment and params list once, reuse across the three queries:

```python
conditions = []
params: list[str] = []
if start is not None:
    conditions.append("timestamp >= ?")
    params.append(f"{start} 00:00:00")
if end is not None:
    conditions.append("timestamp <= ?")
    params.append(f"{end} 23:59:59")
where_clause = f"WHERE {' AND '.join(conditions)}" if conditions else ""
```

Apply to all three sub-queries:

- **`total`** (line 72): add `where_clause` after `FROM brew_events`, pass `params`.
- **`per_drink`** (lines 73-84): the filter **must go in the `ON` clause of the
  `LEFT JOIN`, not a `WHERE`**. If you put it in `WHERE`, drink types with zero
  brews in the range get filtered out of the result entirely (the `LEFT JOIN`
  row still exists with NULL `be.id`, but `WHERE be.timestamp >= ?` on a NULL
  fails and drops the row). Rewrite the `ON` clause:
  ```sql
  LEFT JOIN brew_events be ON be.drink_type = dt.name
      AND be.timestamp >= ? AND be.timestamp <= ?
  ```
  Only append the conditions that are non-None, same pattern as above but as
  part of the `ON` clause, with its own params list built the same way.
  Verify every drink type still appears with `count: 0` when the range excludes
  all its brews.
- **`per_day`** (lines 85-95): add `where_clause` after `FROM brew_events`, pass
  `params`. An empty range naturally produces `per_day: []` — do not add special
  handling here, the frontend deals with the empty list (see section 3).

Return value shape is unchanged: `{"total_brews": ..., "per_drink": [...], "per_day": [...]}`.

### Edge cases to handle in this function

- **Empty range (no brews match):** `total_brews` = 0, every entry in
  `per_drink` has `count: 0`, `per_day` = `[]`. This should fall out naturally
  from the query changes above — no extra code needed, just verify it with a
  test (section 5).
- **`start` after `end` (reversed range):** do **not** validate this here.
  SQLite will just return zero rows for a reversed range (`timestamp >= start
  AND timestamp <= end` where `start > end` is never true) — that's silently
  wrong, not an error, so validation belongs in the API layer where you can
  return a 400. Leave `get_stats` dumb: it trusts its inputs.
- **`start`/`end` in the future:** no special handling needed here either. A
  future `end` just means nothing is excluded on the high side; a future
  `start` just means everything is excluded (empty result, same as any other
  empty range). This function does not know "today" and should not care.

---

## 2. `src/brewops/api/main.py` — `/api/stats`

Current (lines 73-75):
```python
@app.get("/api/stats")
def stats(conn: sqlite3.Connection = Depends(get_db)):
    return queries.get_stats(conn)
```

New:
```python
@app.get("/api/stats")
def stats(
    start: str | None = None,
    end: str | None = None,
    conn: sqlite3.Connection = Depends(get_db),
):
    start, end = validate_date_range(start, end)
    return queries.get_stats(conn, start, end)
```

FastAPI will bind `?start=...&end=...` query params automatically from these
function parameters — no Pydantic model needed (they're simple optional
strings on a GET, consistent with how `machine_id` is a path param elsewhere).

Add a new helper function near `parse_timestamp` (after line 54):

```python
DATE_FORMAT = "%Y-%m-%d"


def validate_date_range(start: str | None, end: str | None) -> tuple[str | None, str | None]:
    """Validate optional YYYY-MM-DD query params for /api/stats.
    Does not reject future dates — only malformed strings and start > end."""
    parsed_start = parsed_end = None
    if start is not None:
        try:
            parsed_start = datetime.strptime(start.strip(), DATE_FORMAT)
        except ValueError:
            raise HTTPException(400, f"unparsable start date {start!r}")
    if end is not None:
        try:
            parsed_end = datetime.strptime(end.strip(), DATE_FORMAT)
        except ValueError:
            raise HTTPException(400, f"unparsable end date {end!r}")
    if parsed_start is not None and parsed_end is not None and parsed_start > parsed_end:
        raise HTTPException(400, "start date is after end date")
    return start, end
```

### Edge cases to handle in this layer

- **Malformed date string** (e.g. `?start=banana`): 400 with
  `"unparsable start date 'banana'"` — mirrors the existing wording style of
  `parse_timestamp`'s `"unparsable timestamp {value!r}"` (line 51).
- **`start` after `end`:** 400 with `"start date is after end date"`. This is
  the one case that must be caught here, not left to SQL, because a silent
  empty result would look identical to "no coffee that week" and mislead the
  user into thinking the range was valid.
- **Date in the future:** allowed, no error. A user picking "last 7 days" from
  a widget that defaults `end` to today is normal, not a mistake. Do **not**
  reuse `parse_timestamp`'s future-rejection logic here — that check exists to
  stop someone from *logging* a brew that hasn't happened yet; it does not
  apply to *querying* a range. If `start` is in the future, that's just an
  empty range, not an error (see section 1).
- **Only one of `start`/`end` given:** valid. Means "from the beginning" or
  "through today" respectively. No error.
- **Neither given:** valid, identical to current all-time behavior. This must
  not regress — run the existing `test_stats` test in `tests/test_api.py`
  (line 26) unmodified and confirm it still passes with no query params.

---

## 3. Frontend — `src/brewops/frontend/index.html` and `app.js`

### `index.html`

Inside `<section id="dashboard">`, before `<div class="stat-tiles">` (line 18),
add a small filter bar:

```html
<div class="date-filter">
  <label for="filter-start">From</label>
  <input type="date" id="filter-start">
  <label for="filter-end">To</label>
  <input type="date" id="filter-end">
  <button type="button" id="filter-apply">Apply</button>
  <button type="button" id="filter-reset">Reset</button>
  <p id="filter-message" class="message" role="status"></p>
</div>
```

`type="date"` inputs give browser-native pickers and already emit
`YYYY-MM-DD`, which matches the API param format exactly — no conversion
needed (contrast with the existing `datetime-local` handling in
`localNow()`, which normalizes `T`-separated values; you do not need
anything like that here).

### `app.js`

Current `loadDashboard` (lines 78-90) always calls `fetchJSON("/api/stats")`
with no params. Change it to accept an optional range and build a query
string:

```js
async function loadDashboard(range = {}) {
  const params = new URLSearchParams();
  if (range.start) params.set("start", range.start);
  if (range.end) params.set("end", range.end);
  const query = params.toString() ? `?${params}` : "";
  const stats = await fetchJSON(`/api/stats${query}`);
  ...
}
```

Everything below that first line in `loadDashboard` (lines 79-90) stays as-is
— `renderDrinkBars`, `renderTimeline`, and the machine-cards fetch don't need
to know about the range, they just render whatever `stats` comes back with.

Add a new function to wire up the filter controls, called once at startup
alongside `setupForms()` (near line 161-162):

```js
function setupDateFilter() {
  const startInput = document.getElementById("filter-start");
  const endInput = document.getElementById("filter-end");
  const message = document.getElementById("filter-message");

  document.getElementById("filter-apply").addEventListener("click", async () => {
    message.textContent = "";
    message.className = "message";
    try {
      await loadDashboard({ start: startInput.value, end: endInput.value });
    } catch (error) {
      message.textContent = error.message;
      message.classList.add("error");
    }
  });

  document.getElementById("filter-reset").addEventListener("click", async () => {
    startInput.value = "";
    endInput.value = "";
    message.textContent = "";
    message.className = "message";
    await loadDashboard();
  });
}
```

Call `setupDateFilter();` at the bottom of the file next to the existing
`loadDashboard().catch(...)` and `setupForms().catch(...)` calls (lines
157-161).

### Handling "brews on last active day" tile (lines 81-82)

Current code:
```js
const lastDay = stats.per_day[stats.per_day.length - 1];
document.getElementById("brews-today").textContent = lastDay ? lastDay.count : 0;
```

This already degrades safely to `0` when `per_day` is empty — no code change
required. But relabel the tile in `index.html` (line 25) when a filter is
active, since "last active day" is misleading once the user has picked a
range that may not include today. Simplest approach: leave the label static
as `"brews on last active day in range"` (edit line 25's text) so it reads
correctly whether or not a filter is applied — do not add conditional label
logic in JS for this, it's not worth the complexity.

### Edge cases to handle in the frontend

- **Empty range (no brews match):** `per_day` comes back `[]`. `renderTimeline`
  (line 32) already has `if (perDay.length === 0) return;` and leaves the SVG
  empty — that's a silent blank panel, which reads as broken rather than
  "no data." Add one line inside `renderTimeline`, right after the existing
  early-return check, to show a message instead of just returning:
  ```js
  const svg = document.getElementById("timeline");
  svg.innerHTML = "";
  if (perDay.length === 0) {
    document.getElementById("filter-message").textContent = "No brews in this range.";
    return;
  }
  ```
  Clear that message at the top of `setupDateFilter`'s apply handler (already
  done above via `message.textContent = ""` before the fetch) so it doesn't
  persist once a non-empty range is applied.
- **Reversed range (`start` after `end`):** the API returns 400 with detail
  `"start date is after end date"`. `fetchJSON` (line 3-10) already throws
  `Error(body.detail || ...)` on non-OK responses, and the `filter-apply`
  click handler above already catches that and writes it into `#filter-message`
  with the `error` class. No extra frontend code needed beyond the try/catch
  already shown — just confirm manually that the message surfaces correctly.
- **Future dates:** no special frontend handling. The date input allows
  picking any date; the API accepts it and returns whatever data exists
  (likely empty, handled by the case above).
- **Only `start` or only `end` filled in:** `URLSearchParams` only sets the
  params that are truthy, so this works with no extra logic.

---

## 4. Do not change

- `get_machine_health` and `/api/machines/{id}` — machine cards stay lifetime-only.
- `parse_timestamp` in `main.py` — that's for POST body validation (rejects
  future timestamps for *logging*), unrelated to query-range validation. Do
  not merge or reuse it for `validate_date_range`; they have different rules
  (future is invalid for one, valid for the other).
- CSV ingestion (`src/brewops/ingest/`) — unaffected by this ticket.

---

## 5. Verification

Run `uv run pytest` after each layer. Add these specific cases:

### `tests/test_db.py`

Add to (or near) `test_stats_math` (line 38), using the same seeded rows:

- `get_stats(conn, start="2026-06-01", end="2026-06-01")` → `total_brews == 2`,
  `per_day == {"2026-06-01": 2}` (2026-06-02 excluded).
- `get_stats(conn, start="2026-06-02", end="2026-06-02")` → confirm `per_drink`
  still lists `espresso` with `count: 0` (not missing from the list) even
  though all espresso brews are on 06-01, outside this range. This is the
  regression test for the `LEFT JOIN ON`-vs-`WHERE` bug described in section 1.
- `get_stats(conn, start="2026-07-01", end="2026-07-31")` (empty range, no
  brews in July) → `total_brews == 0`, `per_day == []`, every `per_drink`
  entry has `count: 0`.
- `get_stats(conn, start="2099-01-01", end="2099-01-31")` (future range) →
  same as the empty-range case above; confirms no exception is raised for
  future dates at the query layer.

### `tests/test_api.py`

Add near `test_stats` (line 26), reusing the `db` fixture's seeded rows
(2026-06-01 and 2026-06-02):

- `GET /api/stats?start=2026-06-01&end=2026-06-01` → 200, `total_brews == 2`.
- `GET /api/stats?start=2026-07-01&end=2026-07-31` → 200, `total_brews == 0`,
  `per_day == []` (empty range, not an error).
- `GET /api/stats?start=2026-06-05&end=2026-06-01` → 400, detail contains
  `"after"` (reversed range).
- `GET /api/stats?start=banana` → 400, detail contains `"unparsable"`.
- `GET /api/stats?start=2099-01-01` → 200, not an error (future start date is
  valid, just yields an empty/partial result — this is the test that would
  catch someone wrongly copy-pasting the future-rejection logic from
  `parse_timestamp` into `validate_date_range`).
- `GET /api/stats` with no params → still 200 with the same totals as before
  this change (regression guard on default behavior).

### Manual verification (frontend)

`uv run seed` then `uv run python -c "from brewops.api.main import run; run()"`
(per the Windows workaround noted in `CLAUDE.md`), open `http://localhost:8123`:

1. Confirm the dashboard loads all-time totals by default, unchanged from
   before this change.
2. Pick a `From`/`To` range that covers only part of the seeded data, click
   Apply — confirm tiles, drink bars, and timeline update to match.
3. Pick a range with no brews in it — confirm tiles show 0, drink bars show
   all zero, and the timeline panel shows "No brews in this range." instead
   of silently rendering nothing.
4. Pick `From` after `To` — confirm an error message appears near the filter
   controls (not a silent failure, not a JS console error, not a stale chart).
5. Pick a `To` date in the future — confirm no error, and results reflect
   data up through today.
6. Click Reset — confirm it returns to the same all-time view as step 1.
7. Log a new brew via the "Log a brew" form while a filter is active — confirm
   `loadDashboard()` is called with no args after a manual log (existing
   behavior, `submitForm` line 150), i.e. logging a brew resets the view to
   all-time rather than silently re-applying a stale filter. If that's
   undesirable, flag it — it's a product decision, not a bug, and out of
   scope for this plan to decide unilaterally.

---

## 6. Follow-up: "Apply does nothing" + unstyled filter bar

Reported after the above was implemented: clicking **Apply** appeared to do
nothing. Root cause when investigated — **not a code bug**: a stale
`uv run python -c "from brewops.api.main import run; run()"` process from an
earlier session was still bound to port 8123, serving the pre-filter build of
`main.py`/`app.js`. Requests to `/api/stats?start=...&end=...` were landing on
that old process, which ignored the query params entirely and never applied
the fix from section 2. Confirmed by checking `netstat -ano` for the PID
holding port 8123, killing it, and restarting — the same curl requests then
returned filtered totals and the 400 on a reversed range as designed.

There is no code change required to fix "Apply does nothing" itself. What
this section adds:

### 6a. Guard against this class of confusion in future manual verification

When manual-verifying section 5's steps, first check nothing else is already
bound to port 8123 before starting the server:
```
netstat -ano | findstr :8123
```
If a PID is listed, stop it first (`Stop-Process -Id <pid> -Force` in
PowerShell) — otherwise the new server fails to bind, silently exits or errors
in its own log, and the *previous* process keeps answering requests with old
code, which looks exactly like "my change didn't take effect."

### 6b. Style the `.date-filter` bar to match the rest of the frontend

The bar currently renders with zero custom CSS — unstyled `<label>`/`<input
type="date">`/`<button>` elements, which is why it visually clashes with the
warm coffee-house theme in `style.css` (the `:root` custom properties defined
lines 3–33: `--panel`, `--line`, `--brass`, `--brass-deep`, `--crema`,
`--radius-sm`, `--serif`, `--sans`, etc.).

Add a `.date-filter` block to `style.css`, near the other dashboard rules
(after `.stat-tiles` / before `.bar-row`, or wherever reads cleanest next to
`.panel`). Requirements, matching existing conventions used elsewhere in the
file:

- Wrap the bar in the same visual language as `.panel` (`--panel` background,
  `--line` border, `--radius` or `--radius-sm` corners) so it reads as part of
  the dashboard rather than a bare HTML form — compare how `.form-panel
  select, .form-panel input` (lines 403–420) style the existing date/datetime
  inputs and reuse that same look for `#filter-start` / `#filter-end`
  (`border: 1px solid var(--line)`, `border-radius: var(--radius-sm)`, `font:
  inherit`, focus ring via `border-color: var(--brass)` +
  `box-shadow: 0 0 0 3px rgb(185 129 47 / 0.22)` — the same values, not
  reinvented).
- Style `#filter-apply` as the primary action using the same gradient as
  `.form-panel button` (lines 422–451): `linear-gradient(135deg, var(--brass),
  var(--brass-deep))`, `color: #fff8ee`, same hover/active/focus-visible
  treatment (`filter: brightness(1.07)` on hover, `translateY(1px)` on active,
  brass focus outline).
- Style `#filter-reset` as a secondary/quiet action — do not duplicate the
  brass gradient on both buttons or Apply loses its primacy as the one
  affirmative action in the bar. A plain bordered button (`background:
  transparent` or `var(--panel)`, `border: 1px solid var(--line)`, `color:
  var(--ink-soft)`) is enough; give it a hover state (e.g. `border-color:
  var(--brass)`) so it doesn't look disabled.
- Lay the label/input/button row out with flexbox (`display: flex;
  align-items: center; gap: ...`), wrapping on narrow widths — this bar sits
  above `.stat-tiles` at the top of `#dashboard`, which spans full width, so
  it should flow naturally at the `@media (max-width: 820px)` and
  `@media (max-width: 560px)` breakpoints already defined (lines 482–517)
  rather than needing new breakpoints. Verify it doesn't overflow or overlap
  at 560px.
- `#filter-message` reuses the existing `.message` / `.message.error` classes
  (lines 461–475) for the error-red text on a reversed range — no new color
  needed, just make sure it isn't given the form's `grid-column: 2` rule
  (that only applies inside `.form-panel form`'s grid; confirm `.date-filter`
  doesn't accidentally inherit or need it, since it's a flex container, not a
  grid).
- Match spacing to the rest of the dashboard: same outer margin/gap as
  `.stat-tiles` uses relative to the panels below it (`gap: 1.1rem` /
  `1.35rem` pattern already used throughout), so the filter bar doesn't sit
  flush against the tiles.

Do not introduce new CSS custom properties — every color/radius/shadow needed
already exists in `:root` (lines 3–33). Do not add a new font — `--sans` for
labels/buttons, matching `.form-panel label` and `.form-panel button`.

Verify by reloading `http://localhost:8123` after a *fresh* server start (see
6a) and confirming: the bar looks like a natural extension of the panel below
it, Apply is visually the primary action, Reset is visually secondary, and
the layout doesn't break at narrow widths.
