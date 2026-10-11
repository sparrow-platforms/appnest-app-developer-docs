# Commands

All app actions are run as:

```bash
appnest-development-engine app <action>
```

Run them from your **app project folder** (the folder that contains `app-backend/`, `app-frontend/` and `manifest.json`): every action treats the current folder as the app project. Action names are case-insensitive.

Back to the **[engine overview](README.md)** · `.env` keys: **[configuration.md](configuration.md)**

---

## Summary

| Action | What it does |
|--------|--------------|
| [`init`](#init) | Downloads the starter app into the current folder. |
| [`precheck`](#precheck) | Checks Node.js 22+ and that the engine and app folders exist. |
| [`install-packages`](#install-packages) | Installs the engine's dependencies and your `app-frontend` dependencies. |
| [`start`](#start) | Runs the app locally behind one port. |
| [`validate`](#validate) | Lints your app code and checks `manifest.json`. |
| [`bundle-frontend`](#bundle-frontend) | Builds `app-frontend/dist` and `app-install-frontend/dist`. |
| [`pack`](#pack) | Builds the frontends, then zips the app into `app-pack-zip/`. |
| [`ai-context`](#ai-context) | Downloads the Appnest developer docs into `appnest-ai-context/`. |

Other invocations:

| Invocation | What it does |
|------------|--------------|
| `appnest-development-engine --version` (`-V`) | Prints the engine version. |
| `appnest-development-engine help` | Prints the list of app actions. |
| `appnest-development-engine app`, `app help` | Same list. |
| `appnest-development-engine app -h` (`--help`) | Usage for `app`, followed by the same list. |
| `appnest-development-engine -h` (`--help`), or no arguments | Top-level usage. |

An unknown action (e.g. `app strat`) prints the list of actions.

---

## `init`

Downloads the starter app ([sparrow-platforms/appnest-app-basecode](https://github.com/sparrow-platforms/appnest-app-basecode), `master`) and copies its contents into the current folder.

- Refuses to run (exit code 1) if the folder already contains `app-backend/`, `app-frontend/` or `app-install-frontend/`. Other files are allowed; starter files with the same name overwrite them.
- Needs `curl` and `unzip` on your `PATH`, and network access to GitHub.
- The starter includes `app-backend/`, `app-frontend/`, `manifest.json` and a `.env`.

## `precheck`

Checks that:

- Node.js **22 or later** is installed;
- the engine's own folders exist, and your project has `app-backend/` and `app-frontend/`. A missing `app-install-frontend/` is reported but doesn't fail the check.

Exits with code 1 if a check fails.

## `install-packages`

Runs `npm install` in each part of the engine, then in your **`app-frontend/`**. Run it once after installing or upgrading the engine, and again when `app-frontend/package.json` changes.

- Run it from your app project folder: it fails (exit code 1) if `app-frontend/package.json` doesn't exist.
- It does **not** install `app-backend` or `app-install-frontend` dependencies. Run `npm install` in those folders yourself if they have a `package.json` with dependencies.

## `start`

Starts the local proxy, which then starts your backend, your frontend and (if the folder exists) your install frontend, and serves them all on one port. Stop it with `Ctrl+C`.

- Port: `APPNEST_ENGINE_PORT` from `.env`, default **3456**.
- **Requires a `.env` file** in the project folder (it may be empty); `start` fails without one.
- Routes: `/app-backend/…`, `/app-frontend/…`, `/app-install-frontend/…`.
- Changes under `app-backend/` restart the backend. Frontend edits hot-reload; changing `vite.config.js` or `.env` inside `app-frontend/` (or `app-install-frontend/`) restarts that dev server.

Details: [configuration.md](configuration.md#local-urls).

## `validate`

Lints your code and checks `manifest.json`, then exits with code **1** if there are any errors (warnings alone pass), so you can use it in CI.

- **`manifest.json`:** checked against [Manifest-Rules.md](../app-configuration/Manifest-Rules.md), including that every function in `backend_api_functions` and every `handler` in `event_listener_functions` is exported from `app-backend/server.js`. Older manifest shapes the platform still accepts are reported as warnings. A missing `manifest.json` is an error.
- **`app-backend/`** (`.js` files): ESLint recommended rules, Node globals, and **CommonJS only**: `import`, `export` and dynamic `import()` are errors.
- **`app-frontend/`** and **`app-install-frontend/`** (`.js`, `.jsx`, `.css`): ESLint recommended rules with browser globals and JSX; stylelint checks for invalid hex colours, duplicate font names and unknown units.
- `node_modules/` and `dist/` are skipped. `// eslint-disable` comments are ignored.
- Unused variables are warnings. Ignored: arguments starting with `_`, a caught error named `exception` (`catch (exception)`), and, in frontends, names starting with a capital letter (components used as `<Component />`). If you don't need the error, use `catch { }`.

## `bundle-frontend`

Production build (Vite) of your frontends:

- `app-frontend/` → `app-frontend/dist/`
- `app-install-frontend/` → `app-install-frontend/dist/`

A folder that doesn't exist is skipped with a warning. Each `dist/` is emptied before the build. The build reads `CLIENT_SCRIPT_URL` from your **shell environment**, not from `.env`; see [configuration.md](configuration.md#env-keys).

## `pack`

Packs the app for upload.

1. Builds your frontends, exactly like [`bundle-frontend`](#bundle-frontend). If a build fails, `pack` stops with exit code 1.
2. In `app-frontend/dist/index.html`, replaces `/frontend/` with `./`. If that file doesn't exist after the build (e.g. there is no `app-frontend/`), `pack` stops with exit code 1.
3. Copies into a temporary `appnest-app-pack/` folder:
   - `app-frontend/`, `app-backend/` and `app-install-frontend/`, whole (including `src/` and any `node_modules/`); missing folders are skipped with a warning;
   - `manifest.json`;
   - the extra folders and files listed in `PACK_FOLDER_PATHS` / `PACK_FILE_PATHS` in `.env` ([configuration.md](configuration.md#env-keys)).
4. Zips it to **`app-pack-zip/app-pack-<timestamp>.zip`** (the zip contains the `appnest-app-pack/` folder) and deletes the temporary folder.

**Env files are never packed.** `.env` and `.env.*` (e.g. `.env.local`, `.env.production`) are left out wherever they are, with a `🔒 Skipping …` warning. Templates (`.env.example`, `.env.sample`, `.env.template`) are packed.

## `ai-context`

Downloads [sparrow-platforms/appnest-app-developer-docs](https://github.com/sparrow-platforms/appnest-app-developer-docs) (`master`) into **`appnest-ai-context/`** in the current folder, for AI coding tools to read. An existing `appnest-ai-context/` is replaced, but only after the download succeeds. Needs `curl`, `unzip` and network access to GitHub.
