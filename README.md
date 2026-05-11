# Command Centre Dashboard

A self-contained HTML dashboard for projecting delivery, commercial, and team health onto a TV in the office. Pulls data from a Google Sheet over CSV and re-renders every 60 seconds. No backend, no build step — open `index.html` in a browser and it runs.

Built for AppliedAI Solutions Delivery to make the state of the portfolio visible from across the room: which engagements are red, what we're watching, where margins are slipping, who's at capacity, what's coming next.

## What it shows

The dashboard rotates between three views every 20 seconds, with a persistent headline strip across the top and a scrolling pipeline strip across the bottom.

**Headline strip (always visible)**
- Revenue Under Management — total contract value, with delivered-to-date underneath
- Portfolio GM — weighted gross margin vs target
- Portfolio CSAT — average satisfaction across active engagements
- Productive Utilisation — team average vs target

**View 01 — Delivery**
- Engagements list: each row shows RAG status, programme name, client, PM, SAR flag, CR count, current week, CSAT
- Watch List: Issues (red) and Risks (amber) across the portfolio, sorted with Issues first

**View 02 — Commercial**
- Margin slippage: per-engagement bars comparing sold GM% (or GM target if no quote) against actual GM%
- Portfolio GM% impact: the sold-vs-actual rollup with delivery slippage in points

**View 03 — Team**
- Utilisation: per-person bars with semantic colour bands (red = below target, amber = watch, green = healthy, amber-to-red = over-cooked)
- Hiring: open roles with stage pills and candidate counts

**Bottom scrolling strip** — upcoming projects from the Pipeline tab (Committed first, then Business Validation), client names only, refreshing as the pipeline changes.

## How it works

```
Google Sheet (your data)
       ↓
   Publish each tab → CSV URL
       ↓
   index.html fetches CSVs every 60s
       ↓
   Dashboard renders inline
```

The whole thing is one HTML file. CSV URLs live in the `SHEET_URLS` block at the top of `index.html` — set once, no other config needed. If no URLs are set, the dashboard renders against built-in dummy data so you can see what it looks like before wiring up a real sheet.

## Setup

### 1. Build the Google Sheet

Two options for getting started:

- **Empty template:** `docs/backend_template.xlsx` — same structure as the dashboard expects, dropdowns and formulas pre-wired, one greyed-out example row per tab showing the format
- **Populated template:** `docs/backend_template_populated.xlsx` — same template with realistic dummy data across all tabs, useful for testing before real data is ready

Upload either to Google Drive and open with Google Sheets. The dropdowns, formulas, and number formats survive the conversion.

### 2. Publish each tab to CSV

For each data tab (Engagements, Risks, Pipeline, Team, Hiring, Config), in Google Sheets:

`File → Share → Publish to web → choose the tab → Comma-separated values (.csv) → Publish`

Copy each URL. They look like `https://docs.google.com/spreadsheets/d/e/.../pub?gid=...&single=true&output=csv`.

You do not need to publish the CRs tab — it's read by formulas inside the Engagements tab, not directly by the dashboard. You also do not need to publish the README or Quotes tabs.

### 3. Wire the URLs into index.html

Open `index.html` and find the `SHEET_URLS` block near the top of the `<script>` section (around line 900):

```javascript
const SHEET_URLS = {
  engagements:  "<paste Engagements CSV URL>",
  risks:        "<paste Risks CSV URL>",
  pipeline:     "<paste Pipeline CSV URL>",
  team:         "<paste Team CSV URL>",
  hiring:       "<paste Hiring CSV URL>",
  config:       "<paste Config CSV URL>",
  // satisfaction, quotes, wins — leave empty unless used
};
```

Leave any unused slots as empty strings.

### 4. Open it

Open `index.html` in a browser. That's it. The dashboard fetches on load, then every 60 seconds. Refresh interval is configurable in the Config tab (`Refresh interval seconds`).

## Data structure

Each tab in the Google Sheet maps to a section of the dashboard. The column names must match exactly — the dashboard reads them by header text, so renaming a column will break it. The dropdowns enforce the enum values; using different values (e.g. "Critical" instead of "Issues") will cause the dashboard to miscount or render incorrectly.

**Foreign keys**
- `Engagement ID` on Engagements is the join key for Risks and CRs
- `Client name` on Engagements is the join key for Quotes (if used)

**Permanent flags**
- `SAR` on Engagements: once flagged at kickoff, leave it flagged for the lifetime of the engagement
- `Confidential` on Engagements: when TRUE and Config `Guest mode` is also TRUE, the client name is masked on the dashboard

**Computed columns**
- Engagements `Current week` is a formula based on Start date
- Engagements `CR count` and `CR value to date` are formulas reading the CRs tab
- Team `Utilisation %` is a formula (`Billable / Capacity`)
- Quotes `Discount %` is a formula

The pre-wired formulas extend down to row 50 — once you start adding real data, just type into row 2 onwards and the formulas fire automatically.

## Customisation

Brand colours, fonts, and refresh behaviour are all CSS variables and Config values:

- **Colours** — defined as CSS variables at the top of `<style>` (`--space-black`, `--safety-blue`, `--code-green`, `--amber`, `--red`)
- **Fonts** — Prompt (display), Open Sans (body), JetBrains Mono (metrics) — pulled from Google Fonts
- **Logo** — paste an `<img>` or inline SVG into the empty `<div class="opus-logo">` in the top bar; the divider and spacing reappear automatically
- **View rotation** — `VIEW_INTERVAL_MS` controls how long each view stays on screen (default 20s)
- **Refresh interval** — controlled by the Config tab's `Refresh interval seconds`

## Troubleshooting

**Dashboard shows "FETCH FAILED"** — usually means a CSV URL is stale or the tab isn't published. Re-publish from `File → Share → Publish to web` and confirm the URL still resolves in a browser.

**Numbers look wrong / dummy data still shows** — at least one CSV must respond successfully for the dashboard to switch out of dummy mode. Check the browser console for fetch errors.

**Status indicator stuck on "LIVE" with no real data** — means none of the CSV URLs were set, so the dashboard is rendering against built-in dummy data. Wire up at least one URL.

**Colours don't update when GM target changes** — the Config tab is read on each refresh; give it a minute or hit reload.

**Engagement appears on the dashboard but no CRs show up** — the CRs tab uses `Engagement ID` as the foreign key. The match has to be exact; check for trailing whitespace or capitalisation differences.

## File structure

```
.
├── index.html                          ← the dashboard
├── docs/
│   ├── backend_template.xlsx           ← empty template
│   └── backend_template_populated.xlsx ← dummy data version
├── README.md
└── .gitignore
```

## Hosting

The dashboard is a static HTML file with no build step. Any static host works:

- **GitHub Pages** — push to `main`, enable Pages in repo settings, point at the root
- **Netlify / Vercel** — drag the folder in
- **Local TV / laptop** — open `index.html` from disk

For TV display: open in fullscreen (F11), pin the tab, and let it run.

## Notes

The dashboard is read-only — it doesn't write back to the sheet. Updates happen in the sheet; the dashboard reflects them on next refresh.

Empty values are handled gracefully — engagements without a CSAT score show "—", missing dates skip cleanly, no formula errors propagate to the visual layer.

The match between Pipeline stage values and the bottom strip is exact: rows must use `Business Validation` or `Committed` (case-sensitive). Other values are filtered out.
