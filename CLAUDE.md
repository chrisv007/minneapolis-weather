# Minneapolis Weather Triangulation — Project Context

## Purpose

This is a personal weather dashboard for Minneapolis, MN. The core idea is
triangulation, not "show the weather." The dashboard shows what each
forecast source says, where they agree or disagree, and whether a given
number is an official forecast, a raw model output, or a measured
observation. Every feature should serve that comparison purpose. If a
change would blur the distinction between sources, or make the dashboard
look more precise or authoritative than the underlying data actually is,
don't make it without flagging the tradeoff.

The user has zero coding experience and directs this project entirely
through conversation with Claude Code. Explain nontrivial changes in
plain language. Don't assume familiarity with how the code works, and
don't make sweeping structural changes (new framework, new backend,
major dependency) without explaining what changed and why in terms a
non-coder can follow.

Read `weather-dashboard-project-notes.md` in this repo before making any
changes. It has the architecture, data flow, endpoints, panel layout,
thresholds, and known limitations at the code level. Do not relitigate
decisions documented there without a specific reason.

## How to read this document

Two tiers below. **Product principles** are the actual point of the
project and should hold regardless of how the code is built or
deployed. **Implementation approach** is intentionally flexible: use
normal, current software engineering practice, whatever serves the
product principles best, and defer to whatever the repo already does
rather than forcing it back toward how this project was originally
delivered.

History for context: this dashboard was originally built by hand as a
single self-contained HTML file, because it was first developed in a
Claude.ai chat conversation, where Claude can't run a build step or
maintain a repo, and Claude's artifact sandbox can't fetch external
APIs at all. Those were constraints of that delivery method, not
properties the product needs. It has since been moved into a GitHub
repo deployed via Cloudflare Workers, built with Claude Code. Do not
treat "single file," "no build step," "no frameworks," or "no
dependencies" as rules to enforce or restore. If the repo now has
multiple files, a build step, a framework, or dependencies, that's
fine and expected. Use whatever structure the repo currently has as
the baseline, and evolve it using your own judgment the way you would
on any repo.

As of now, the repo is still a single `index.html` at the root with no
build step, deployed as static assets via Cloudflare Workers (see
Deployment below). That's the current state, not a constraint to
preserve for its own sake.

## Product principles (hold regardless of implementation)

- **Triangulation is the point.** Every forecast value shown should be
  traceable to a specific source, and the dashboard's job is comparison
  and disagreement, not a single "best guess" number. Don't average
  sources into a single display value unless the comparison itself is
  still visible somewhere.
- **Label provenance honestly.** Always distinguish official forecast
  vs raw model output vs measured observation, and native sub-hourly
  data vs interpolated/coarser data. Don't let a feature imply more
  precision or timeliness than the underlying source actually has.
- **Keep this free to run.** Use free, keyless data sources wherever
  possible. The one exception is Pirate Weather, which needs a free
  API key. That key is entered by the user in the page itself and
  stored only in the browser's local storage on their device. It must
  never be hard-coded, committed, put in an env file that ships to the
  client, or logged anywhere. If a change would require a paid API or
  an ongoing bill, don't make it without surfacing that cost to the
  user first.
- **Mobile-first.** The user is almost always on an iPhone. Check every
  visual and layout decision against a narrow screen before considering
  it done.
- **Favor clarity over cleverness.** If a design choice trades a small
  amount of clarity for a more polished or "clever" look, default to
  clarity. This project has already gone back and forth on this
  (see Known limitations below) and clarity has won every time.

### Open question, not a settled rule: does this need a backend?

The original "no backend" stance was a byproduct of being a single
static file with no server. Cloudflare Workers supports serverless
functions now, so a lightweight backend is genuinely possible, for
example to proxy the Pirate Weather key so the user doesn't have to
re-enter it per device, or to cache API responses. That's a real
architecture option now, not something ruled out. But it's also a
meaningful change: it adds a server component to maintain, a possible
cost if usage ever scaled, and moves away from the "everything happens
in your own browser, nothing is sent anywhere but the weather APIs
themselves" property the user has valued. Default to keeping data
fetches client-side unless a specific feature clearly benefits from a
backend, and if you do think one is warranted, explain the tradeoff to
the user in plain language before building it rather than adding it
quietly.

## Implementation approach (flexible, use your judgment)

- File structure, build tooling, frameworks, and dependencies are all
  fair game. Use whatever is normal and maintainable for a small
  Cloudflare Workers project. There's no requirement to keep this
  minimal for its own sake.
- One real caution though: the user cannot personally read or debug
  the code. That means unnecessary complexity has a higher cost here
  than on a typical project, since the user's only way to change
  anything is by asking an AI to do it. Prefer dependencies and
  patterns that a future Claude Code session can understand quickly
  over ones that are trendy or marginally more elegant. Simplicity is
  still a tiebreaker, just not a hard rule.
- Validate changes using whatever the repo's actual tooling is (lint,
  typecheck, build, test scripts). Currently that means: extract the
  `<script>` block from `index.html` and run `node --check` on it, and
  unit-test pure logic functions (tier classification, percentile
  math, window filtering, wording) with small mock inputs where
  practical. If the repo gains real tooling (a test runner, linter,
  build script), use that instead and don't assume this manual check
  still applies.
- Unit-test pure logic functions where practical, especially anything
  computing agreement/disagreement, percentiles or medians, time-window
  filtering, or onset timing. These are the functions most likely to
  fail silently and produce a confidently wrong answer on the page.
- When editing, make targeted, well-scoped changes for the task at
  hand rather than opportunistically rewriting unrelated parts of the
  codebase, unless the task is explicitly a refactor.
- When responding to the user in chat, always give the full file or
  full copy-paste blocks, never partial snippets. The user has no
  coding experience and cannot merge fragments themselves.

## Data sources

All free and keyless except Pirate Weather. Treat this section as the
historical intent; if the actual code does something different,
follow the code, but flag it in your response if the difference looks
like a bug rather than an intentional change.

**NWS (api.weather.gov)** — official US forecast + observations. No
key required. CORS from a browser only works if no custom `User-Agent`
header is sent (the browser's own default works; overriding it breaks
the CORS preflight).
- `/points/{lat},{lon}` returns `forecast` and `forecastHourly` URLs.
- `forecastHourly` periods give `temperature` (F), `windSpeed` /
  `windGust` (strings like "10 mph", parse the number out),
  `probabilityOfPrecipitation.value` (%), `dewpoint.value` (C, convert
  to F), `relativeHumidity.value`, `shortForecast`.
- `forecast` periods are day/night pairs; daily high comes from the
  daytime period, low from the night period of the same date.
- `/stations/KMSP/observations/latest` is the measured ground-truth
  anchor. Units are SI (temp/dewpoint in C, wind km/h, pressure Pa,
  visibility m); convert to imperial. `textDescription` is the human
  conditions string used to detect active precipitation.

**Open-Meteo (api.open-meteo.com/v1/forecast)** — model data. No key,
CORS supported. Query one model at a time via `&models=`. Standard
params: `temperature_unit=fahrenheit`, `wind_speed_unit=mph`,
`precipitation_unit=inch`, `timezone=America/Chicago`,
`timeformat=unixtime`, `forecast_days=7`.
- Models in use: `gfs_seamless` (GFS/HRRR, NOAA), `ecmwf_ifs025`
  (ECMWF), `icon_seamless` (DWD ICON).
- Sub-hourly: `&minutely_15=precipitation,rain,snowfall` with
  `&forecast_minutely_15=8` and `models=gfs_seamless`. For North
  America this is the genuine NOAA HRRR model at true 15-minute
  resolution. For ICON/ECMWF the 15-minute series is interpolated
  from hourly data, not native sub-hourly, and must stay labeled as
  such wherever it's shown.

**Open-Meteo ensemble (ensemble-api.open-meteo.com/v1/ensemble)** —
`models=gfs_seamless`, `hourly=temperature_2m`, roughly 31 GFS
ensemble members. Used for the 7-day daily-high uncertainty cone
(p10/p50/p90 of each member's daily max). Falls back gracefully to
plain deterministic model spread if the ensemble call fails.

**Open-Meteo air quality (air-quality-api.open-meteo.com/v1/air-quality)**
— `current=us_aqi,pm2_5`. Single-source context, not part of the
triangulation comparison.

**Pirate Weather (api.pirateweather.net/forecast/{key}/{lat},{lon})**
— Dark Sky-compatible, `minutely` gives 60 one-minute steps
(`precipIntensity` in/hr, `precipProbability` 0-1, `precipType`).
Requires the user-supplied key described above. CORS is enabled on
the production endpoint.

Location constants: `LAT = 44.9778`, `LON = -93.265`,
`TZ = "America/Chicago"`, NWS station `KMSP`.

## Design system

Treat these as historical intent; if the repo's actual CSS or theme
differs, the repo wins.

- Instrument feel, not a consumer weather app. Cool off-white
  background, deep slate ink, monospace tabular numerals for data
  readouts so digits align and spread is scannable at a glance.
- Signature visual pattern: an agreement strip, a horizontal track
  with one tick per source, used anywhere sources are compared
  directly (temperature spread, wind gust spread, etc).
- Source colors: Actual `#11181C`, NWS `#1F6FB2`, GFS/HRRR `#C9821F`,
  ECMWF `#2E8B7F`, ICON `#7A5CC4`, consensus `#3A4A52`.
- Agreement-level colors: good `#2F9E6E`, medium `#D9982B`,
  bad `#D9583B`.
- Precipitation type colors: rain `#8BC34A`, snow `#4A90D9`,
  mix/sleet `#8E5FC4`. Color encodes type only, never intensity or
  source. Bar height encodes amount/intensity.

## Known limitations and prior decisions (don't relitigate without reason)

- **Color must encode type only, not intensity.** A shaded ramp within
  a color (pale to saturated, or green-to-red by intensity) was tried
  twice and both times read as broken, faded, or contradicted the
  legend. The rule now is one flat color per precipitation type, with
  height carrying amount. Don't reintroduce intensity-by-color without
  discussing it first.
- **Honesty comes from labels, not from degrading the visual.** An
  attempt to show interpolated (non-native) sub-hourly data as coarse
  hourly blocks instead of fine bars badly hurt readability and was
  reverted. The working solution is a uniform fine-bar layout with an
  explicit tag per row ("native 15-min" vs "interpolated") and an
  honest footnote. Resolution differences should be disclosed in text,
  not by making some rows visually worse.
- **HRRR and Pirate Weather are model forecasts, not live radar.** Both
  derive their sub-hourly precipitation from the HRRR model. When
  HRRR's run misses or underplays a developing or spotty cell, both
  can read near-zero while it is actively raining, confirmed during a
  real event where ECMWF, ICON, and the observed station all showed
  rain but HRRR did not. This is a genuine model limitation, not a
  bug, and cannot be fixed in code. The mitigation in place is an
  "Observed now at MSP" anchor line pulled from the live station
  observation, shown in both nowcast panels, plus an explicit
  "no measurable precip modeled" label instead of an empty chart when
  a source reads dry.
- **NWS browser CORS** requires sending no custom `User-Agent` header.
- **iOS `localStorage`** for a `file://`-origin page may not persist
  between sessions; this was part of the motivation for moving to
  Cloudflare hosting.
- **Snowfall units** from Open-Meteo are assumed centimeters and
  converted to inches (÷2.54). This was never verified against a real
  snow event; sanity-check it the first time snow actually falls.
- **Verification habit:** when a model's real-world behavior matters
  (does HRRR's onset match reality, is the snowfall conversion right),
  the right test is empirical, comparing against the observed station
  or actual conditions during a live event, not just reading docs.

## Working conventions

- Treat any request phrased as "find X" or "show evidence Y is true"
  as a neutral instruction to investigate, not to confirm the premise.
  If the evidence is weak, missing, mixed, or contradicts what the
  user assumed, say so plainly rather than building the response
  around the most satisfying or agreeable answer.
- Disagree with the user's assumptions, framing, or requested approach
  whenever the evidence or reasoning warrants it, even if they haven't
  invited disagreement. Be direct without being needlessly blunt; keep
  a warm, helpful tone rather than a clinical one.
- When a change involves a decision, recommendation, or judgment call,
  not a simple or purely mechanical fix, end with a brief, honest note
  on what tradeoffs, risks, or better alternatives the user may be
  overlooking.
- Do the work rather than redirecting the user to do it themselves.
  Don't tell the user to go check something, look something up, or
  test something in a place you have access to (the repo, the live
  site, an API, search). If there's a genuine reason you can't act, say
  why instead of just pointing them elsewhere.
- The user has zero coding experience. When you do show code in
  conversation rather than only editing it directly in the repo, give
  the full relevant block, not an isolated fragment they're expected
  to merge by hand. Explain what changed and why in plain language.
- Avoid em dashes in any text shown to the user (commit messages, PR
  descriptions, chat responses). Only use one if it's truly
  unavoidable.

## Deployment: read this before touching git or Cloudflare

This repo deploys through Cloudflare Workers Builds, connected directly to this GitHub repository. It works like this:

1. The Cloudflare project (`minneapolis-weather`, live at `minneapolis-weather.chris-vierig.workers.dev`) is already connected to this repo through Cloudflare's dashboard.
2. Cloudflare watches the `main` branch. Any push to `main` automatically triggers a build and deploy on Cloudflare's own servers.
3. The deploy command Cloudflare runs is `npx wrangler deploy`, using a Cloudflare API token stored inside the Cloudflare dashboard, scoped to this project. That token lives on Cloudflare's side, not in this local environment, not in this repo, and not in any environment variable available here.
4. Static assets are served from the repo root per `wrangler.jsonc` (`assets.directory: "./"`). `.assetsignore` excludes non-site files (`README.md`, `wrangler.jsonc`, `CLAUDE.md`, `weather-dashboard-project-notes.md`, `.gitignore`, `.git`) from being served as public assets.

**What this means for you, Claude Code:**
- Never run `wrangler deploy`, `wrangler pages deploy`, or any deploy command yourself in this local environment.
- Never ask the user to set `CLOUDFLARE_API_TOKEN` or any other Cloudflare credential locally. It is not needed and should not exist in this environment.
- Your job after making a change is: commit it, push it to `main`, confirm the push succeeded, and stop. Do not attempt to deploy.
- If a deploy-related error appears (missing credentials, "not logged in," etc.), that is expected and correct. It means you tried to deploy locally, which is not how this project works. Stop and just push instead.
- The user can verify a deploy happened by checking the Deployments tab in the Cloudflare dashboard for the `minneapolis-weather` project. That is a manual browser step only the user can do. Don't attempt to check it yourself.
