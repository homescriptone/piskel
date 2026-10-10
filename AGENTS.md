# AGENTS.md — Piskel contributor guide

This file helps AI coding agents (and humans) work in this repo quickly and safely.

## Project overview

Piskel is a browser-based pixel-art / sprite editor (the editor behind piskelapp.com).
Stack: plain JavaScript + HTML + CSS, no framework, no bundler ESM imports for app code.
Build system: custom Vite plugins concatenate files into `dest/prod/`.
Desktop apps: NW.js via `nw-builder` (see `scripts/desktop.js`).

Maintainer policy (see README): only small, straightforward fixes are likely to be
merged. Large refactors, complex multi-area bug fixes, and UX/feature changes are
unlikely to be reviewed. Keep PRs minimal.

## Essential commands

```bash
npm ci                    # install (Node 20, see .github/workflows/ci.yml)
npm run build             # sprites + vite build + partials -> dest/prod/
npm start                 # build + serve with live reload at http://localhost:9001
npm run start:test        # build + serve WITHOUT live reload (used by Playwright)
npm run lint              # biome check src/
npm run lint:fix          # biome check --write src/
npm run unit-tests        # karma + jasmine (needs Chrome)
npm run e2e               # playwright (spawns npm run start:test automatically)
npm run test-ci           # bash ./run-test-ci.sh (unit + e2e, CI entry point)
npm run build:desktop     # NW.js build for Windows (ia32+x64) + Linux (ia32+x64)
npm run build:mac         # NW.js build for macOS (osx x64)
npm run start:nw          # build + run in NW.js locally
```

Outputs:
- `dest/prod/` — web build (gitignored). Entry: `dest/prod/index.html`.
- `dest/desktop/` — NW.js builds, one folder per `platform-arch` (gitignored).
- `cache/` — downloaded NW.js binaries (gitignored, cached in CI).

## Build pipeline details

1. `node scripts/sprite.js` — builds `src/img/icons.png`, `icons@2x.png`, `src/css/icons.css`
   from individual icons (spritesmith). These generated files are gitignored.
2. `vite build` with custom plugins in `vite-plugins/`:
   - `html-include` — resolves `@@include` directives in `src/index.html`.
   - `concat-scripts` — reads `src/piskel-script-list.js` (single source of truth
     for JS load order) and emits one minified `js/piskel-packaged-{version}.js` via terser.
   - `css-replace` — concatenates `src/piskel-style-list.js` CSS in order.
3. `node scripts/partials.js` — post-processing of the prod HTML.
4. `node scripts/serve.js` — static server for `dest/prod` on port 9001, with
   SSE live reload (CSS hot-swap, full reload for JS/HTML). `--test` disables
   watching/reload and additionally serves `tests/e2e/data`.

## Architecture

- Entry: `src/index.html` → `src/piskel-boot.js` → `src/js/app.js`.
- Namespacing: single global `pskl` object + `$.publish/$.subscribe` event bus
  (`src/js/lib/pubsub.js`, events in `src/js/Events.js`, constants in `src/js/Constants.js`).
- Load order matters: `src/piskel-script-list.js` defines it. New files MUST be
  added there in dependency order, otherwise the build silently breaks at runtime.
- Key directories under `src/js/`:
  - `controller/` — UI controllers (settings, dialogs, panels).
  - `model/` — frame/layer/palette/piskel document model.
  - `service/` — import/export (GIF via gifjs, PNG, ZIP, .piskel files), keyboard, etc.
  - `tools/` — drawing tools (pen, bucket, shapes, dithering, …).
  - `rendering/` — canvas frame/layer renderers.
  - `widgets/`, `selection/`, `utils/`, `database/`, `worker/`.
- `src/js/lib/` and `src/css/{bootstrap,spectrum}/` are vendored third-party code:
  Biome lint/format is disabled there (see `biome.jsonc` overrides). Do not edit.
- `src/index.html` uses `@@include` template directives: Biome is disabled for it.
- Desktop specifics:
  - `package.json` `"window"` field + `"main": "./dest/prod/index.html"` configure NW.js.
  - `src/js/utils/FileUtilsDesktop.js` holds NW-only file access. Keep web/desktop
    code paths separated; guard NW APIs (`require('nw.gui')` style) so the web build never breaks.
  - `misc/desktop/` holds `nw.icns`, `logo.ico`, `Info.plist`, Enigma Virtual Box config.
  - NW.js version is pinned in `scripts/desktop.js` (`NW_VERSION`). Bump it there
    and update the `nwjs-*` cache key in `.github/workflows/desktop.yml`.

## Tests

- Unit: `tests/unit-tests/` run with Karma + Jasmine + Chrome (`karma.conf.js`).
  Run `npm run unit-tests`. Watch mode: `npm run unit-tests:watch`.
- E2E: `tests/e2e/playwright/` run with Playwright (`playwright.config.ts`).
  Run `npm run e2e`. Snapshots live next to specs; `tests/e2e/playwright/tmp` is gitignored.
- CI (`.github/workflows/ci.yml`): `biome ci src/` on ubuntu, then unit+e2e matrix
  on `ubuntu-latest` + `macos-latest`, Node 20.
- Desktop (`.github/workflows/desktop.yml`): builds Windows+Linux on `ubuntu-latest`,
  macOS on `macos-latest`. Manual dispatch with platform filter, auto-build on
  `master` pushes and `v*` tags, artifacts uploaded per platform. Releases are
  automatic after the build: `master`/manual runs update the rolling
  `desktop-latest` prerelease, `v*` tags create a versioned GitHub Release —
  both with zips (+ `.dmg` for macOS, built via staging dir + `hdiutil`) and
  `SHA256SUMS.txt` via `softprops/action-gh-release`. The DMG is unsigned —
  users on recent macOS must right-click > Open on first launch unless you add
  codesigning/notarization (requires Apple Developer secrets).

## Code style

- Biome v2 (`biome.jsonc`): 2-space indent, double quotes, semicolons always,
  no trailing commas, LF line endings. Run `npm run lint:fix` before committing.
- Lint rules enforced: `noUndeclaredVariables`, `noInvalidUseBeforeDeclaration`,
  `useLiteralKeys`, `useBlockStatements` (always use braces), `noWith`.
- Globals (`$`, `pskl`, `Events`, `Constants`, Jasmine fns, …) are declared in
  `biome.jsonc` — do not add `import`/`require` for them in app code.
- Follow existing file conventions: constructor functions / prototype style in
  older modules, plain script concatenation (no ES modules in `src/js`).
- Husky runs on `prepare`; keep changes biome-clean so hooks/CI pass.

## Safe-change checklist for agents

1. Read `src/piskel-script-list.js` if adding/renaming JS files — register new files there.
2. Run `npm run build` then `npm run lint` after any `src/` change.
3. For UI changes, run `npm run start` and verify at http://localhost:9001.
4. For behavior changes, run relevant unit tests (`karma`) and, if UI-facing,
   the Playwright suite (`npm run e2e`).
5. For desktop changes, test `npm run start:nw` locally if possible; the full
   matrix only runs in CI. Never commit `dest/`, `cache/`, or `node_modules/`.
6. Keep diffs small and focused per the maintainer PR policy.
