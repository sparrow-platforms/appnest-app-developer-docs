# Appnest Development Engine

Local development CLI for Appnest apps. It runs your **`app-backend`**, **`app-frontend`** and optional **`app-install-frontend`** behind one local proxy, validates your code and **`manifest.json`**, and packs the app into a zip for upload.

**Package:** `@sparrowengg/appnest-development-engine` · **CLI:** `appnest-development-engine` · **Requires:** Node.js 22 or later

| Doc | What's in it |
|-----|--------------|
| **[commands.md](commands.md)** | Every `app` action: what it does, what it needs, what it writes |
| **[configuration.md](configuration.md)** | App project layout, `.env` keys, local URLs and file watching |
| **[Manifest-Rules.md](../app-configuration/Manifest-Rules.md)** | `manifest.json` schema and the rules `app validate` checks |

---

## Install

```bash
npm install -g @sparrowengg/appnest-development-engine
appnest-development-engine --version
```

## Quick start

Run every command from your **app project folder** (the folder that contains `app-backend/`, `app-frontend/` and `manifest.json`).

```bash
mkdir my-app && cd my-app
appnest-development-engine app init               # download the starter app into this folder
appnest-development-engine app install-packages   # install the engine's dependencies and app-frontend's
appnest-development-engine app start              # run the app locally
```

`app start` serves everything on one port: **`APPNEST_ENGINE_PORT`** from your `.env`, or **3456** if it isn't set. The starter's `.env` sets it to `3001`, so open `http://localhost:3001/app-frontend/`. See [configuration.md](configuration.md#local-urls).

Before you upload:

```bash
appnest-development-engine app validate          # lint + manifest checks; exits 1 on errors
appnest-development-engine app pack              # build the frontends, then zip the app into app-pack-zip/
```

To build without packing (e.g. to check the production build), run `app bundle-frontend`.

## What the engine provides

- **Backend:** your `app-backend/server.js` runs locally under `/app-backend`, and the functions it exports are what gets called. Declare each one in `manifest.json` (see [Manifest-Rules.md](../app-configuration/Manifest-Rules.md)); `app validate` fails if a declared function isn't exported. `app-backend` must be **CommonJS** (`require` / `module.exports`); `app validate` rejects `import` / `export`.
- **Frontend:** a Vite + React dev server with hot reload for `app-frontend`, served under `/app-frontend`. The engine loads **`app-frontend/src/App.jsx`** (default export) as the root component and provides React and React DOM itself.
- **Install UI (optional):** the same for `app-install-frontend` (root component `app-install-frontend/src/App.jsx`), served under `/app-install-frontend`. Leave the folder out if you don't need one.
