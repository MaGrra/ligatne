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

## Requirements
The master must be shared **"Anyone with the link → Viewer"**. Publish-to-web does
NOT work here — the aerones.com Workspace blocks publishing outside the domain
(returns 401 anonymously). Link-sharing does work and was verified.

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
