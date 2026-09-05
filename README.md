# Rezultāti — Swedbank Līgatne 05.09.2026

Mobile-first live leaderboard. One static file, no build step, no dependencies.

## How it works
The page fetches `KOPVĒRTĒJUMS` as CSV straight from Google Sheets and renders it.
The master already holds every team's per-task total, overall total, place and
attended count, so one request gives the whole picture.

    https://docs.google.com/spreadsheets/d/<ID>/export?format=csv&gid=<GID>

Refreshes every 30 s, and again whenever the phone comes back to the tab.
Last good response is kept in localStorage, so a dropout shows stale data with an
amber dot instead of an empty screen.

## Three groups (2026-09-05)
One URL, a switcher in the header, one mirror tab per group. The default group
comes from the saved team number (1–82 Tautas, 101–113 Sporta, 201–232
Baudītāji), so a team lands on its own board without touching anything.

A phone fetches **only the group on screen**. Switching shows the cached copy
at once and refreshes it behind that. TV mode fetches all three, because it
cycles them.

**Baudītāji are not scored.** Their board shows attended tasks and bingo ticks,
ranked by attended then bingo — no points anywhere. The place is computed in
the page, not read from the sheet, because the master's Bauda tab has no Kopā
or Vieta column at all.

## Requirements — READ THIS
The page reads a **values-only public mirror**, NOT the master.

    REZULTĀTI (publiskais)  1c8sSEoA28Ki13tOdA8XdzBJ7dhc42AjJtKDHCHfYZAU
      Sporta  gid 0
      Tautas  gid 110833847
      Bauda   gid 404359574

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

    SHEET_ID     the public mirror file id
    GROUPS[]     one entry per group: gid + every column index it uses
    REFRESH_MS   30000
    TV_CYCLE_MS  20000   how long each group holds the TV
    TV_TOP       20      82 Tautas teams will not fit one screen

## Only the SU tasks are listed
`tFirst..tLast` covers **only the 22 numbered SU tasks**. BINGO and the three
Sporta race tasks (velo posms, svara uzdevums, trases laiks) are scored but are
not listed anywhere on the public page — not in the team breakdown, not in the
list under the map, not on the map.

**Their points ARE still in Kopā**, because Kopā is read straight from the
sheet's own column. That was deliberate: the online order then always matches
the real standings. The trade-off is that the listed task scores add up to less
than the Kopā shown — on live data by about 185 for Sporta and 11 for Tautas —
so the team sheet carries a line saying so rather than leaving them to wonder.

The internal top-3 page still shows all of them; that is the point of it.

## Column contract (must match the master) — DIFFERENT PER GROUP
This is the thing most likely to break. Sporta carries 25 task columns (22 SU +
BINGO + two reserved for the race), Tautas 23, Bauda 22 ticks plus a bingo
count, so the summary columns sit at different indexes in each tab:

    group    task cols   Sods  Kopā  Vieta  Apmeklēti  Bingo   rows
    Sporta   2..26        27    28     29      30        --    3..30
    Tautas   2..24        25    26     27      28        --    3..95
    Bauda    2..23        --    --     --      25        24    3..50

Every one of those lives in `GROUPS[]` and nowhere else. Adding or removing a
task column shifts the summary columns for that group — update the entry.
Run `python3 v3/audit_page.py` to assert them against the live sheet headers.

Cutoff moved to **AH1 (index 33)**, and is written to **all three** master tabs,
because each group's board only ever reads its own tab.

Task names come from row 2, so renaming a task in the master updates the page.
Adding or removing a task column shifts the last four — update the constants.

## The task list under the map
Counts are **scoped to the class on screen** — a Sporta team sees how many of
the 13 have been to a task, not how many of all 127. `DATA` is already just
that group, so this is true by construction.

Tasks the picked team has finished move to the **bottom** under a `PADARĪTS`
heading with a `✓` and the team's own score; what is still to do sits at the top
under `VĒL JĀDARA`. With no team picked it says so rather than guessing.

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
- **Swedbank 2026-09-05: coordinates live in `v3/coords.csv` in the scoring
  project, NOT typed into the sheet.** Row 1 is inside the range a master
  rebuild clears, so hand-entered coordinates would vanish the next time a task
  column was added. `build_master.coords()` reads the CSV, rejects anything
  outside Latvia as a transcription error, and joins several points for one task
  with `|`. 22 tasks, 23 pins (SU15 has an upstream and a downstream start).
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

## TV mode
Same link plus `#tv`:  https://magrra.github.io/ligatne/#tv
There is also a **TV režīms** button in the footer, and an **Iziet no TV**
button bottom-right while in TV mode.

**It rotates the three groups every 20 s, unattended** — Sporta (all 13 fit),
then Tautas TOP 20, then Baudītāji TOP 20. Dots in the header line show which
group is up. A group with no data yet is skipped rather than showing an empty
screen for 20 seconds, so it behaves sensibly before the IMPORTRANGE clicks are
done. Entering `#tv` fetches all three immediately instead of waiting for the
next 30 s tick.

Teams on one screen, no scrolling: a CSS grid filled column-first,
`ceil(n/2)` rows, so ranks 1-13 run down the left column and 14-26 down the
right. Each row carries the same facts as a mobile row — place, team name,
`Nr. X · N no 21 uzd.`, penalty if any, and points. Teams on 0 stay listed.
Banner, tabs, map, task list and footer are hidden; the header line keeps the
countdown and the data age.

- Sizes use `clamp()` with `vh` units, so it fills a 1080p TV and still works on
  a laptop. `vh` is safe here because a TV browser has no address bar that
  hides.
- Padding is `2.4vh 2vw` so TV overscan (some sets crop the edges) cannot eat a
  row.
- The grid is only rebuilt when the numbers actually change (a signature string
  is compared), so it does not blink every 30 seconds.
- **TV obeys the same cutoff** as mobile. At the cutoff the grid is replaced by
  the closed message.
- A TV screensaver will still blank a static page. Turn it off on the TV;
  browser-side prevention is unreliable on TV browsers.

Nothing about the mobile view changed. `render()` returns to `renderTV()` on the
first line only when the flag is present; with no flag the original path runs
untouched.

## Internal top-3 page (unlisted)
`iekseja-be6f4cff51bc.html` — top 3 per task, for picking shout-outs at the
ceremony. Reads the SAME mirror as the board and ranks in the browser, so there
is no second data path.

**Mobile-first accordion.** A list of tasks, collapsed, each row showing the two
class leaders on one line so it can be scanned without tapping. Opening a task
shows Sporta's top 3 and then Tautas' underneath. `Atvērt visus` /
`Aizvērt visus` for a laptop at the finish.

**Only the 17 tasks that award a bonus appear.** On the other nine every team
that completes the task scores the same, so a top 3 would be sixty tied names —
SU1 and SU3 have ~60 Tautas teams tied for first. The list is generated from the
one task table, and `groups` is per task: **SU5 shows Tautas only**, because
Sporta scores it 5p per answer with no ranking. `SVARS` and `LAIKS` show Sporta
only.

**It is UNLISTED, not private.** GitHub Pages serves it to anyone with the URL,
and the filename is visible in the public repo listing. The page says so on
itself. For real access control the master spreadsheet is already private to the
three organisers.

- Baudītāji are absent: they have no scores, only attendance ticks.
- **It ignores the cutoff** — the shout-outs happen after the board closes.
- Ties share a place, as the scoring does; a long tie caps at 6 rows with
  "+ vēl N ar tādu pašu rezultātu", and a task where every team tied says so.
- Do NOT name a top-level function `top`: it collides with the read-only
  `window.top` and the SyntaxError kills the whole script silently.

## Team picker + greyed-out pins
On first open the page asks in **two steps**: which class (Sporta / Tautas /
Baudītāji), then which team within it. 127 names in one list is not a choice
anyone can make on a phone; the class narrows it to 13, 82 or 32 and the second
step is a normal scroll. `← Cita klase` goes back. The choice is kept in
localStorage (`swd-myteam-v1`, `swd-group-v1`). On the map, every task that team has
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
- A numbered row with no NAME is not a team at all — `shape()` skips it. That
  drops the phantom "Komanda Nr.20" from the standings, the picker and the map
  counts. 26 teams everywhere. Anyone can pick any team; a static page cannot verify identity,
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
