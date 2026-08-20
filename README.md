# Rezultāti — Līgatne 2026

Mobile-first live leaderboard. One static file, no build step, no dependencies.

## How it works
The page fetches `KOPVĒRTĒJUMS` as CSV straight from Google Sheets and renders it.
The master already holds every team's per-task total, overall total, place and
attended count, so one request gives the whole picture.

    https://docs.google.com/spreadsheets/d/<ID>/export?format=csv&gid=<GID>

Refreshes every 30 s, and again whenever the phone comes back to the tab.
Last good response is kept in localStorage, so a dropout shows stale data with an
amber dot instead of an empty screen.

## Requirements — READ THIS
The page reads a **values-only public mirror**, NOT the master.

    REZULTĀTI (publiskais)  1L3Jw8csMEK3-du2xkBMA8yg6e32TWALuC7zG4d6FQHc  tab DATI

Why: the master's cells contain IMPORTRANGE formulas naming all 21 task file ids,
and those task files are deliberately "anyone with the link can EDIT" so referees
need no Google account. Publishing the master id would therefore have handed any
visitor a route to edit live scores. The mirror holds display values only — no
formulas, no file ids — and the master is private again (verified: anonymous read
returns 401).

`mirrorPublic()` in the Loquiz Apps Script copies `A1:AD60` into the mirror after
every sync, so it is never more than a minute behind. Sheet errors (`#REF!` etc.)
are blanked on the way across.

Publish-to-web does not work at all here: the aerones.com Workspace blocks
publishing outside the domain (401 anonymously). Plain link-sharing does work.

## Publishing to GitHub Pages
1. Create a repo, push `index.html` to the default branch.
2. Settings → Pages → Source: *Deploy from a branch*, branch `main`, folder `/ (root)`.
3. Wait ~1 min, open `https://<user>.github.io/<repo>/`.

## Configuration
Top of the `<script>` block in `index.html`:

    SHEET_ID    the KOPVĒRTĒJUMS file id
    GID         813088766  (the master tab; LOQUIZ tab is 582382323)
    REFRESH_MS  30000

## Column contract (must match the master)
    0   Nr
    1   Komanda
    2..22   the 21 task columns, header "SU4 Skulptūru dārzs"
    23  Sods (-)
    24  Kopā
    25  Vieta
    26  Apmeklēti uzd.

Task names come from row 2, so renaming a task in the master updates the page.
Adding or removing a task column shifts the last four — update the constants.

## Karte (map tab)
Coordinates live in **row 1 of KOPVĒRTĒJUMS**, one cell per task column (C1:W1),
written as `lat,lng` — e.g. `57.2318,25.0389`. They ride along in the same CSV,
so no extra request and no second file.

- A task with no coordinate simply does not appear on the map. Nothing breaks.
  SU21 BINGO is deliberately blank — it is scored remotely.
- **A task can have SEVERAL start points**: put them in the same cell separated
  by `|`, e.g. `57.239496,25.046452 | 57.231163,25.046028`. SU15 Upings uses
  this because teams may start from either end of the river.
  Every point gets its own pin carrying the SAME SU label, and because greying
  is decided per TASK (not per pin), one registered score greys all of that
  task's pins at once. The "x of 20 padarīti" count treats the task as one.
- Real coordinates were loaded 2026-08-20; 20 tasks, 21 pins.
- Map tiles come from OpenStreetMap and Leaflet loads from unpkg, both only when
  the Karte tab is first opened. No internet -> the tab says so, the rest of the
  page still works.
- "Rādīt, kur es esmu" uses the browser's own location. It needs HTTPS, which
  GitHub Pages gives you (localhost also counts). Nothing is uploaded anywhere —
  the position stays in the phone and is drawn locally. No other team sees it.

## Layout: two tabs, map on top of Uzdevumi
`Kopvērtējums` and `Uzdevumi un karte`. The map is a fixed block at the top of
the second tab with the task list under it — there is no separate Karte tab.

The map's own controls live **inside the map** (a `L.Control` in the top-right),
not as page buttons:
- `◎` centre on me. First tap asks for location and starts following; each later
  tap re-centres. **Dragging the map stops it following** (the Google Maps
  rule) and the button dims; tap again to resume. The blue dot keeps updating
  either way.
- `✚` fit all tasks.

`#map` height was `calc(100vh - 260px)`, which is why the map used to jump
around while scrolling: on a phone `100vh` changes as the address bar hides.
It is now `46svh` (small-viewport height, which does not change) with a
`46vh` fallback first for older browsers, clamped 260–440px. `#map` also gained
`position:relative; z-index:0; isolation:isolate` so Leaflet's internal
z-indexes (up to 1000) can no longer paint over the sticky header (z-index 20).

Known trade-off: with the map at the top of a scrolling tab, a one-finger drag
inside the map pans the map instead of scrolling the page. That is why the map
is kept to under half the screen — there is always list below it to scroll from.

## Team picker + greyed-out pins
On first open the page asks "Kura ir tava komanda?" and the choice is kept in
localStorage (`ligatne-myteam-v1`). On the map, every task that team has
already been scored for turns **grey with no shadow**; the pin stays put and its
popup shows the points. `Slēpt padarītos` hides them outright if a team prefers
the shorter map; that toggle is remembered too (`ligatne-hidedone-v1`).
The picked team's row is also outlined in the Kopvērtējums list, and the team
name sits as a tappable chip next to the "Rezultāti" title. The header is
`position:sticky`, so that chip stays visible while scrolling and is the way to
change team from any tab — the old map-only button is gone.

- "Visited" = the team's cell for that task is not empty. A **0 counts as
  visited** — that is the sheet's own rule (blank = never came, 0 = came and
  scored nothing).
- The pin only greys once a **referee types the score**, not when the team
  finishes. Self-service tasks (SU4, SU15, SU21) are scored from WhatsApp and
  can lag by hours; the three Loquiz tasks fill in within a minute. This is why
  pins are greyed and never removed by default — a stale map is then merely out
  of date instead of actively hiding a task the team still has to do.
- The map note prints the data age (`Dati pirms 2 min`) so a team on bad signal
  can see the greying may be behind. No extra polling was added — the page
  already refetches every 30 s, on tab focus, and via `Atjaunot tagad`.
- Team 20 exists as a number with no name, so it is filtered out of the picker
  (`team.named`). Anyone can pick any team; a static page cannot verify identity,
  so treat "which tasks has team X done" as public within the event.
- `Izlaist` skips the question and is remembered, so it is asked only once.

## Beigu laiks (cutoff)
Cell **`AD1` in KOPVĒRTĒJUMS** (label sits in `AC1`). Format `YYYY-MM-DD HH:MM`,
read as local Riga time. The cell's number format is pinned to `yyyy-mm-dd hh:mm`
so the CSV export always looks the same — do not change that format.

- Before the time: a live countdown sits under the tabs. It turns amber under
  15 minutes and red under 5.
- At the time: the results flip to "Rezultāti vairs nav pieejami · Tiekamies
  finišā!". The two results tabs, footer and countdown disappear — but the
  **map stays open indefinitely**, so teams can still navigate after the
  leaderboard closes. `render()` forces the view to the map tab, hides the
  standings tab and the task list, and leaves the map and the refresh button
  working. Only the standings are ever hidden, never the map.
- Empty cell = no cutoff, results stay visible forever.
- The page keeps polling after closing, so pushing the time out in the sheet
  brings the results back within 30 seconds. Same for pulling it in early.
- The clock is the phone's own. A phone with a badly wrong clock will flip at the
  wrong moment; nothing we can do from a static page.
