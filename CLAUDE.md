# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository purpose

This repo is a Home Assistant **add-on repository** (declared by `repository.json` at the root) that ships a customized fork of the upstream [matt8707/ha-fusion](https://github.com/matt8707/ha-fusion) dashboard. Home Assistant supervised installations consume this repo URL and install the add-on defined under `ha_fusion_custom/`.

All real code lives in `ha_fusion_custom/`. The root only contains `repository.json` plus the add-on directory.

## Add-on layout (`ha_fusion_custom/`)

The directory is **both** a Home Assistant add-on (driven by `config.yaml` + `Dockerfile` + `run.sh`) **and** a SvelteKit application (Svelte 4 + Vite, adapter-node, pnpm). Three deployment paths share the same source tree:

- **HA add-on (production)**: `config.yaml` declares `arch: amd64`, exposes `8080/tcp`, `startup: application`. The supervisor builds the `Dockerfile` (base `ghcr.io/hassio-addons/base/amd64`), then `run.sh` (`bashio` shebang) `cd /app && node server.js`. The CMD in `Dockerfile` (`npm run serve`) is overridden by `run.sh` in the addon context.
- **Standalone Docker**: `docker-compose.yml` (production image reference) and `compose-dev.yml` (local build) — used when running outside HA Supervisor.
- **Local dev**: `pnpm install && npm run dev -- --open` against an `HASS_URL` set in `.env` (copy from `.env.example`).

`ha-fusion-custom.tar` is a checked-in prebuilt image artifact — do not regenerate or modify it casually.

## Common commands

Run all of these from `ha_fusion_custom/`:

```bash
pnpm install              # install deps (preferred over npm)
npm run dev -- --open     # vite dev server, hot reload
npm run build             # SvelteKit build via adapter-node → ./build
npm run check             # svelte-kit sync + svelte-check (typecheck)
npm run lint              # prettier --check + eslint
npm run format            # prettier --write
```

There is no test runner configured — `npm run check` (type checking) is the closest thing to a test suite.

## Architecture

### Runtime entry point: `server.js`

`server.js` is the production server that wraps SvelteKit's `./build/handler.js` in an Express app and adds a **proxy middleware**:

- It proxies `/api/*` and `/local/*` to a Home Assistant instance.
- The proxy `target` is computed per-request by `entryMiddleware`:
  - In **add-on mode** (`ADDON=true`), it derives the target from headers: `x-hass-source` + `x-forwarded-proto` + `x-forwarded-host` for ingress, or rewrites the host's exposed port to `HASS_PORT` when accessed via direct port.
  - Otherwise it falls back to the `HASS_URL` env var.
- It stamps the resolved target onto `req.headers['X-Proxy-Target']` so SvelteKit's `+page.server.ts` can read it.

This is why the SvelteKit API endpoints live under `src/routes/_api/` (with leading underscore) — `/api/*` is reserved for the proxy to Home Assistant.

### SvelteKit app: `src/`

- `src/routes/+page.server.ts` — single-page server load. Reads `data/configuration.yaml` and `data/dashboard.yaml`, then loads the active theme from `static/themes/` (or `build/client/themes/` in prod) and translation JSON from `static/translations/`. Resolves `hassUrl` from `HASS_URL` or the `X-Proxy-Target` header set by `server.js`.
- `src/routes/_api/` — server endpoints (`save_dashboard`, `save_config`, `load_theme`, `get_calendar`, `get_translation`, `youtube`, `version`, etc.). All under `_api` to avoid collision with the HA proxy at `/api`.
- `src/lib/Socket.ts` — Home Assistant WebSocket connection (`home-assistant-js-websocket`). Supports both long-lived access tokens (from `configuration.yaml`) and the OAuth flow; falls back to a `TokenModal` when running inside the HA companion app/ingress where redirects fail.
- `src/lib/Stores.ts`, `Types.ts`, `Utils.ts` — central Svelte stores, type definitions, shared helpers.
- `src/lib/Main/` — dashboard view rendering (`Index.svelte`, `Views.svelte`, sections, picture elements).
- `src/lib/Sidebar/`, `src/lib/Drawer/`, `src/lib/Modal/` — sidebar widgets, edit-mode drawer, and per-domain modal editors. The Modal directory is large — there's one `*Modal.svelte` per HA entity domain (Light, Climate, Cover, Camera, Calendar, etc.) plus `*Config.svelte` editors.
- `src/lib/Components/` — generic UI primitives (sliders, color picker, code editor, theme).
- `src/lib/Settings/` — global settings panes (token, language, custom JS, version, etc.).
- `src/hooks.server.ts` — strips the SvelteKit `link` header so the app survives behind nginx/reverse proxies that reject oversized headers.

### Data & state

- `data/dashboard.yaml` — user dashboard layout (committed).
- `data/configuration.yaml` — user configuration including HA token (gitignored).
- `data/custom_javascript.js` — optional user-provided JS injected at runtime.
- `static/themes/*.yaml`, `static/translations/*.json` — theme and i18n bundles. After build these are also copied to `build/client/`; `+page.server.ts` switches between the two paths via `dev`.

`PROGRESS.md` tracks the upstream project's per-domain implementation status. It's a roadmap, not a spec for this fork.

## Things to know when modifying this codebase

- This is a **fork**, not a from-scratch project. When making changes, check whether you're touching upstream code or genuine local customization. The git log on this repo (only a handful of commits) won't tell you — the upstream history isn't preserved here.
- New API routes go under `src/routes/_api/` (not `/api/`), or they will be intercepted by the HA proxy.
- Anything that reads/writes `./data/*` runs server-side — those paths are relative to the working directory, which is `/app` in production. Don't assume CWD elsewhere.
- The Vite config has explicit `optimizeDeps.include` and `exclude` lists — when adding new dependencies that pull in dynamic imports or codemirror modules, you may need to update these to prevent dev-server reloads or codemirror state duplication.
- `arch: amd64` only — the add-on does not target arm64/armv7. If multi-arch is needed, `config.yaml` and the `Dockerfile`'s `BUILD_FROM` need to change in lockstep.
