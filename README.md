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
- Right now those are MOCK coordinates around Līgatne village, ~1.5 x 1.1 km.
  Replace them with the real ones in the sheet; the page picks them up on the
  next 30-second refresh.
- Map tiles come from OpenStreetMap and Leaflet loads from unpkg, both only when
  the Karte tab is first opened. No internet -> the tab says so, the rest of the
  page still works.
- "Rādīt, kur es esmu" uses the browser's own location. It needs HTTPS, which
  GitHub Pages gives you (localhost also counts). Nothing is uploaded anywhere —
  the position stays in the phone and is drawn locally. No other team sees it.

## Beigu laiks (cutoff)
Cell **`AD1` in KOPVĒRTĒJUMS** (label sits in `AC1`). Format `YYYY-MM-DD HH:MM`,
read as local Riga time. The cell's number format is pinned to `yyyy-mm-dd hh:mm`
so the CSV export always looks the same — do not change that format.

- Before the time: a live countdown sits under the tabs. It turns amber under
  15 minutes and red under 5.
- At the time: the whole page flips to "Rezultāti vairs nav pieejami · Tiekamies
  finišā!". Tabs, footer and countdown disappear.
- Empty cell = no cutoff, results stay visible forever.
- The page keeps polling after closing, so pushing the time out in the sheet
  brings the results back within 30 seconds. Same for pulling it in early.
- The clock is the phone's own. A phone with a badly wrong clock will flip at the
  wrong moment; nothing we can do from a static page.
