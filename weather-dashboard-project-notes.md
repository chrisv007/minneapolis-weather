# Weather Dashboard — Project Notes

Working reference for the Minneapolis weather triangulation dashboard. Read this
before changing `index.html`. It captures the architecture, data sources,
endpoints, thresholds, design decisions, and known limitations so they don't get
relitigated by accident.

## Overview

- **Single file:** everything lives in `index.html` at the repo root — markup,
  CSS (in one `<style>` block), and JS (in one `<script>` block). No build step,
  no frameworks, no bundler, no external JS/CSS dependencies.
- **Runs entirely client-side.** All data is fetched live from public weather
  APIs in the browser at page load / on Refresh. There is no server.
- **Location is hard-coded:** `LAT=44.9778, LON=-93.265`, `TZ="America/Chicago"`
  (Minneapolis / MSP). All time formatting is pinned to that timezone via
  `Intl.DateTimeFormat`, so the display is correct regardless of the viewer's
  device timezone.
- **Purpose:** not just "what's the weather" but *triangulation* — show what each
  source says, quantify where they agree or disagree, and always label data as
  **observed** vs **official** vs **model**.

## Sources

Five primary sources drive the comparison (`ALL` = the render order;
`FC` = the four forecast sources used for spread/consensus math):

| id      | name         | kind        | notes |
|---------|--------------|-------------|-------|
| `obs`   | MSP Actual   | observation | live station reading, KMSP |
| `nws`   | NWS Official | official    | the human-adjusted government forecast |
| `gfs`   | GFS (NOAA)   | model       | via Open-Meteo |
| `ecmwf` | ECMWF        | model       | via Open-Meteo |
| `icon`  | ICON (DWD)   | model       | via Open-Meteo |

- `ALL = ["obs","nws","gfs","ecmwf","icon"]`
- `FC  = ["nws","gfs","ecmwf","icon"]` — consensus/spread is over these four.
  `obs` is the ground truth, never part of the "forecast spread."

Context / secondary feeds:

- **GFS ensemble** (`ensemble-api.open-meteo.com`) — many GFS runs, used to draw
  the 7-day uncertainty cone (10th/50th/90th percentile of daily highs).
- **Air quality** (`air-quality-api.open-meteo.com`) — US AQI + PM2.5 for the
  Today strip.
- **Sub-hourly nowcast** (`NOW_SRC`): HRRR (`gfs_seamless`, radar-fed, genuinely
  15-min native), plus ICON and ECMWF interpolated from hourly. Powers the
  "Next 2 hours" nowcast.
- **Pirate Weather** — the only keyed source. True 1-minute / 60-minute nowcast.
  Key is user-entered, stored only in `localStorage` (`pw_api_key`) with an
  in-memory fallback (`PW_MEM_KEY`). Never hard-code, commit, or print it.

### Endpoints

- Open-Meteo forecast: `https://api.open-meteo.com/v1/forecast`
  (`current`, `hourly`, `daily`; `forecast_days=7`; units °F / mph / inch).
- Open-Meteo minutely (nowcast): same host, `forecast_minutely_15=8`
  (eight 15-minute steps → 2 hours), fields `precipitation,rain,snowfall`.
- Ensemble: `https://ensemble-api.open-meteo.com/v1/ensemble`
  (`models=gfs_seamless`, `hourly=temperature_2m`).
- Air quality: `https://air-quality-api.open-meteo.com/v1/air-quality`
  (`current=us_aqi,pm2_5`).
- NWS points → forecast: `https://api.weather.gov/points/{lat},{lon}` then the
  returned `forecastHourly` and `forecast` URLs.
- NWS observations: `https://api.weather.gov/stations/KMSP/observations/latest`.
- Pirate Weather: `https://api.pirateweather.net/forecast/{key}/{lat},{lon}?units=us&exclude=hourly,daily,alerts`.

All Open-Meteo requests use `timeformat=unixtime`; timestamps are converted to ms.

## Data flow

`loadAll()` orchestrates everything and is called on load and on Refresh:

1. Fetch the five primary sources in parallel (`Promise.allSettled`) so one
   failure never blocks the others; each source's status is tracked
   independently in `state.sources[id]` (`loading` / `ok` / `error`).
2. Then fetch ensemble + air quality, then the sub-hourly nowcast sources.
3. Then Pirate Weather, only if a key is present.
4. `render()` runs after each phase; every panel degrades gracefully to
   "loading" / "n/a" / "unavailable" rather than throwing.

`state` shape: `{ sources:{id:{status,data,error}}, ensemble, air, now, pw, updatedAt }`.
Helpers: `cur(id)` = current conditions, `okd(id)` = full data when status ok.

## Panels (top → bottom)

1. **Today strip** — sunrise, sunset, max UV, US AQI (with category label).
2. **Next 2 hours — precipitation nowcast** — per-source 15-min bars over a
   shared 2h timeline; only HRRR is native sub-hourly, the rest are interpolated
   and labeled as such. Anchored by an "Observed now at MSP" line when the
   station reports active precip. A source modeling no measurable precip
   renders as a compact one-line row (label + dotted baseline) instead of a
   tall empty chart; the timeline axis and type legend only appear when at
   least one source shows bars.
3. **How much do the sources disagree?** — a plain-language headline + the "now"
   temperature/big-number summary, then spread strips and categorical rows.
4. **Current conditions by source** — the full grid; units live in the row
   label so value cells stay narrow on a phone; bold ▲ (red) = highest / bold
   ▼ = lowest among forecasts; `Cons.` column = median of the four. The table
   scrolls sideways on narrow screens with CSS scroll-shadow cues.
5. **Next 24 hours** — temperature and precip-probability line charts with a
   min–max ribbon and a bold consensus (median) line.
6. **7-day high temperature & uncertainty** — see below.
7. **Pirate Weather — minute by minute** — 1-minute intensity + chance track for
   the next 60 minutes (only when a key is entered).
8. **Source status** — per-feed connection state.

> Panel order note: the Pirate Weather panel sits near the end, directly above
> Source status, so the minute-nowcast (which needs a user key and is optional)
> lives with the other end-of-page context rather than interrupting the main
> comparison flow.

## Spread / agreement logic

`level(range, lo, hi)` buckets a numeric spread into `low` / `med` / `high`,
mapped to labels ("High agreement" / "Some spread" / "High disagreement") and the
green / amber / red palette. Thresholds are per-metric and intentionally tuned:

The spread-strip *track* is threshold-anchored (`spreadTicks`): its full width
always represents at least 1.5× the high-disagreement threshold, centered on
the values' midpoint, so agreeing sources visibly cluster and only real
disagreement spreads the beads out (previously min–max always stretched to
full width, which made 0.01" of spread look like a split). Beads that would
overlap drop to a lower "lane" on a short gray stem — same horizontal
position means same value; this keeps sources with identical values visible
instead of hiding behind one another.

| metric                    | low < | med ≤ | unit |
|---------------------------|-------|-------|------|
| Temperature (now)         | 3     | 6     | °F   |
| Wind gusts (now)          | 8     | 15    | mph  |
| Precip total, next 24h    | 0.05  | 0.2   | in   |
| Daily high temp           | 3     | 6     | °F   |
| Daily precip chance        | 20    | 40    | %    |
| Daily gusts               | 8     | 15    | mph  |

Derived signals: `expects24` (any hour ≥50% chance or ≥0.01" in 24h),
`nextOnset` (first hour crossing that bar), `precipTotal24`, `ptype24`
(rain / snow / mixed classification from hourly snow vs liquid).

Precip **intensity tiers** for the bar charts:
- Nowcast (`nowcastTier`, inches/15-min step): >0 → 0.5, ≥0.005 → 1, ≥0.02 → 2, ≥0.06 → 3.
- Pirate (`pwTier`, in/hr): <0.002 → 0, <0.017 → 0.5, <0.1 → 1, <0.4 → 2, else 3.
- Type colors are shared: rain `#8BC34A`, snow `#4A90D9`, mix/sleet `#8E5FC4`.

## 7-day high temperature & uncertainty (current design)

Rendered as a **per-day range-bar list** (not a line chart), because the goal is
to read each day's high *and* its uncertainty at a glance on a phone:

- One **shared temperature scale** across all seven rows so bars are directly
  comparable; the scale endpoints are printed in a footer row.
- Each day: a horizontal bar spanning the **ensemble 10–90% range**, a tick at
  the **ensemble median** (the best-estimate high), and — when it diverges by
  ≥1° — a hollow marker for the **4-model consensus** high.
- The numeric high and the low–high range sit on the right of each row.
- Wider bar = less certain; the cone visibly widens toward the far-out days.
- Fallback when the ensemble feed is down: the bar shows the four-model
  lowest–highest spread and the consensus median, with an "ensemble unavailable"
  note.

Each day's row also carries, on a second line inside the same row, the
**per-source high/low readouts** (NWS 88/68 · GFS 90/76 · …) and the
**agreement squares** (T / P / W) showing which sources agree on high temp,
precip chance, and gusts (green / amber / red). This used to be a separate
per-day list below the bars; it was merged into one row per day so the reader
doesn't cross-reference two lists with duplicate day labels.

## Design conventions

- Design tokens live in CSS `:root`; each source has a fixed color reused
  everywhere (legend, ticks, chart lines, status dots) — keep them consistent.
- Numbers use a monospace font so columns align.
- **Mobile-first.** The user is almost always on an iPhone. Panels must stay
  scannable on a narrow screen; test at ~360px wide.
- Charts are hand-built inline SVG (`chartSVG`) with a viewBox and
  `preserveAspectRatio` so they scale fluidly — no chart library. The viewBox
  is 360 wide (≈ a phone's CSS width) so 10px axis text renders at true size
  on mobile instead of being scaled down; on wide screens the SVG is capped
  at 560px via CSS so text doesn't balloon.
- UI copy is honest about limits: label model vs observed, native vs
  interpolated, and don't overclaim precision.

## Known limitations (don't "fix" these blindly)

- **NWS forecast** publishes no pressure, visibility, or precip *amount* — those
  cells show `n/a` by design. It also has no sub-hourly feed.
- **ECMWF** (via Open-Meteo) carries no precip probability, so it's excluded from
  the precip-probability chart and some spread rows.
- **Models can read dry while it's actually raining.** A stale or coarse run may
  show ~0 amount during spotty/just-started precip. This is why nowcast panels
  surface an "Observed now at MSP" line to anchor on the real station.
- **ICON / ECMWF nowcast rows are interpolated** from hourly data, not native
  sub-hourly — they read smoother than HRRR and are labeled accordingly.
- **The ensemble cone is GFS-only**, so it reflects GFS run-to-run spread, not
  cross-model spread. The cone naturally widens with lead time; that's expected
  uncertainty growth, not "disagreement."
- **Pirate Weather is optional and keyed.** Without a key the panel just shows a
  key-entry prompt; the rest of the dashboard is fully functional without it.

## Working checklist

- Make targeted edits; preserve working functionality.
- Validate: extract the `<script>` block and run `node --check`; unit-test pure
  logic (tier classification, percentile math, window filtering, wording) with
  small mock inputs where practical.
- When sharing code in chat, give the **whole file or full copy-paste blocks** —
  never partial snippets (the user cannot merge fragments).
- Commit, push to `main`, confirm the push, and **stop** — Cloudflare Workers
  Builds deploys automatically on push. Never run a deploy command locally.
