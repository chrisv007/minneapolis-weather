# minneapolis-weather

A personal weather triangulation dashboard for Minneapolis, MN, comparing forecasts from NWS, GFS, ECMWF, ICON, HRRR, and Pirate Weather.

This is a static, mostly-keyless, client-side-only app: it fetches public weather APIs (NWS, Open-Meteo) directly from the browser at runtime, with no server and no build step. The optional Pirate Weather data source requires an API key that you supply yourself in the page; it is stored only in your browser's `localStorage` and is never committed to or transmitted through this repo.

Live URL: https://minneapolis-weather.chris-vierig.workers.dev
