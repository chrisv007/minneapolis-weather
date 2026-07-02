# Minneapolis Weather Dashboard: Claude Code Instructions

## What this project is
A personal weather triangulation dashboard for Minneapolis, MN. The deliverable is a single self-contained HTML file, `index.html`, that runs directly in a browser with no build step, no frameworks, and no backend. Vanilla JS, inline SVG charts, no external dependencies.

Core purpose: don't just show the weather. Show what each source says, where sources agree or disagree, and which data is official versus model versus observed.

Read `weather-dashboard-project-notes.md` in this repo before making any changes. It has the full architecture, data sources, endpoints, design decisions, thresholds, and known limitations. Do not relitigate decisions documented there without a specific reason.

## Data sources
All free and keyless except one:
- NWS (api.weather.gov): official forecast and observations, no key
- Open-Meteo: GFS, ECMWF, ICON model data, no key
- Pirate Weather: minute-by-minute nowcast, requires a user-entered API key

The Pirate Weather key is entered by the user directly in the page and stored only in browser localStorage. Never hard-code this key, never put it in a commit, never put it in chat output.

## Hard constraints
- Must stay a standalone HTML file, `index.html` at the repo root. This is a Cloudflare Workers static-assets project, not a build pipeline.
- Never deliver this as a Claude.ai artifact. Artifacts cannot fetch external APIs.
- Free, keyless sources only, with the one user-key exception above.
- Mobile-first. The user is almost always on an iPhone. Keep panels scannable on a small screen.

## How to work
- Make targeted edits to the existing file. Preserve working functionality.
- When responding to the user in chat, always give the full file or full copy-paste blocks, never partial snippets. The user has no coding experience and cannot merge fragments themselves.
- Validate before considering a change done: extract the script block and run `node --check`, and unit-test pure logic functions (tier classification, percentile math, window filtering, wording) with small mock inputs where practical.
- Be honest about data limits in the UI copy: label model vs observed, native vs interpolated. Don't overclaim precision.
- Favor clarity over cleverness in visuals.

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

## Repo structure
