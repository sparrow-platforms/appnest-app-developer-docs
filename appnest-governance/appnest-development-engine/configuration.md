# Configuration

App project layout, the `.env` keys the engine reads, and what `app start` serves.

Back to the **[engine overview](README.md)** · All actions: **[commands.md](commands.md)**

---

## Project layout

| Path | Required? | Role |
|------|-----------|------|
| `app-backend/server.js` | Yes | Backend entry. Export the functions declared in `manifest.json`. CommonJS only. |
| `app-frontend/src/App.jsx` | Yes | Root React component (default export) of the main UI. |
| `app-frontend/package.json` | Yes | Your frontend dependencies; installed by `app install-packages`. |
| `app-install-frontend/src/App.jsx` | Optional | Root component of the install UI. Leave the whole folder out if you don't need one. |
| `manifest.json` | Yes | App declaration for the platform. See [Manifest-Rules.md](../app-configuration/Manifest-Rules.md). |
| `.env` | Yes, for `app start` | Local settings (below). Never packed. |

Folders the engine creates: `app-frontend/dist/` and `app-install-frontend/dist/` (`bundle-frontend`), `app-pack-zip/` (`pack`), `appnest-ai-context/` (`ai-context`). You may want to add them to `.gitignore`.

## `.env` keys

`.env` lives in the app project folder. Format: one `KEY=VALUE` per line; spaces around `=` and surrounding quotes are removed; lines starting with `#` are ignored. Flags are on only when the value is exactly `true`.

| Key | Used by | Default | Effect |
|-----|---------|---------|--------|
| `APPNEST_ENGINE_PORT` | `start` | `3456` | Port the local proxy listens on. |
| `CLIENT_SCRIPT_URL` | `start` | built-in URL | URL of the Appnest client script loaded by your frontend's page. |
| `PROXY_DEBUG_LOGS` | `start` | off | `true` logs every proxied request and proxy errors. |
| `DEBUG_FILE_CHANGES` | `start` | off | `true` logs every file change the engine sees in your app folders. |
| `PACK_FOLDER_PATHS` | `pack` | none | Extra folders to pack. JSON array (`["assets","docs"]`) or comma-separated (`assets,docs`). Relative to the project folder, or absolute. |
| `PACK_FILE_PATHS` | `pack` | none | Extra files to pack, same format. Env files listed here are still left out. |

- During `app start`, **every** key in `.env` is also set as an environment variable for your **backend** (`process.env.MY_KEY`). Frontends get only `CLIENT_SCRIPT_URL`.
- `app bundle-frontend` doesn't read `.env`. To build with a specific client script, set `CLIENT_SCRIPT_URL` in your shell, e.g. `CLIENT_SCRIPT_URL=https://… appnest-development-engine app bundle-frontend`.
- `.env` is never packed, so nothing in it reaches the platform. Keep secrets there for local runs only.

## Local URLs

With `app start` running (port = `APPNEST_ENGINE_PORT`, default 3456):

| URL | Serves |
|-----|--------|
| `http://localhost:<port>/app-frontend/` | Your main UI (Vite dev server, hot reload) |
| `http://localhost:<port>/app-install-frontend/` | Your install UI, if `app-install-frontend/` exists |
| `http://localhost:<port>/app-backend/<functionName>` | Your backend functions (`POST`, JSON body) |

The backend and frontends run on their own free ports behind the proxy; always use the proxy port.

**File watching:**

- Any change under `app-backend/` (except `node_modules/`) restarts the backend.
- Frontend source edits hot-reload in the browser.
- Changing `vite.config.js` or `.env` **inside** `app-frontend/` or `app-install-frontend/` restarts that frontend's dev server. Changing the project's root `.env` takes effect on the next `app start`.
