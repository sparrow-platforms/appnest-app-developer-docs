# App assets (`app-assets/public/`)

Ship static files with your app — images, HTML pages and fragments, email templates, PDFs — and get a URL for each one from your **backend** with **`$asset`**.

Files in `app-assets/public/` are **public**: anyone with the URL can read them. Never put secrets, keys, tokens or customer data there.

**On this page:** [The folder](#the-folder) · [Rules and limits](#rules-and-limits) · [URLs change on every deploy](#urls-change-on-every-deploy) · [`$asset` API](#asset-api) · [Showing assets in the frontend](#showing-assets-in-the-frontend) · [Local development and packing](#local-development-and-packing)

---

## The folder

Put your files under **`app-assets/public/`** at the root of your app project, next to `app-backend/` and `app-frontend/`. Any file type and any folder nesting is allowed.

```text
your-app/
├── manifest.json
├── app-backend/
├── app-frontend/
└── app-assets/
    └── public/              ← everything here is published as-is
        ├── images/logo.png
        ├── templates/welcome.html
        ├── auth/redirect.html
        └── docs/guide.pdf
```

- There is **no manifest key** for assets: the folder is enough. If `app-assets/public/` is missing or empty, the app simply has no assets.
- **Paths are relative to `app-assets/public/`.** The file above is `images/logo.png`, never `public/images/logo.png` or `app-assets/public/images/logo.png`.
- Files are published **exactly as you ship them**. There is no build step, minification or image processing.
- Assets are **read-only** at runtime and fixed for a release. To store files your app creates or users upload, use [`$file`](appnest-functions/backend/file.md).
- **HTML files render as web pages** in the browser (they are not downloaded), so you can ship pages that run on their own. SVG files render too. Every file is served with the content type that matches its extension, and browsers are told not to guess a different type.

### When to use `app-assets/` and when to use `app-frontend/`

| Use | For |
|-----|-----|
| `app-frontend/` (import the file in your code, e.g. `import logo from './logo.png'`) | Images and styles used **only** by your UI. The frontend build bundles them. |
| `app-assets/public/` | Files your **backend** needs a URL for: email images and templates, documents to link to, standalone HTML pages. |
| [`$file`](appnest-functions/backend/file.md) | Files created or changed **at runtime**, or that must stay private. |

---

## Rules and limits

`app validate` and `app pack` check these locally, and the platform checks them again when you upload. An upload that breaks any rule is rejected.

| Rule | Value |
|------|-------|
| Size of one file | At most **5 MB** |
| Total size of `app-assets/public/` | At most **25 MB** |
| Number of files | At most **500** |
| File types and nesting | Any |
| Path length | At most **256** characters (relative to `app-assets/public/`) |
| Path contents | No `..`, no leading `/`, and no `app-runtime` folder anywhere in the path |
| Hidden files (names starting with `.`, e.g. `.env`, `.DS_Store`) | Rejected |
| Symbolic links | Rejected |
| Empty folders | Ignored |
| Names that differ only in case (`Logo.png` and `logo.png`) | Rejected |
| Directly under `app-assets/` | Only the `public/` folder. Loose files and any other folder (including `private/`) are rejected |

- The CLI checks the default limits above. The platform may allow higher limits for some apps; the check at upload is the one that counts.
- Large media such as video doesn't belong in assets. Host it elsewhere and link to it.
- `private/` is reserved for private assets, which are [not available yet](#not-available-yet).

---

## URLs change on every deploy

Every deploy of your app publishes a **fresh copy** of `app-assets/public/` at a **new, unguessable URL**. Your code and its assets always change together: the running backend only ever gets URLs for the assets of the release that is live.

- **Never hard-code an asset URL**, in code, config, `$db` or your frontend. Always get it from [`$asset.getUrl()`](#geturl) when you need it.
- **Don't try to build or parse asset URLs yourself.** Treat the value of [`getBaseUrl()`](#getbaseurl) as an opaque prefix; its format can change.
- **URLs from earlier releases keep working for now, but aren't guaranteed to last.** Content that lives a long time — emails already sent, documents users saved — may outlive the release its asset URLs belong to. If an asset must stay reachable for a long time, keep that in mind (for example, host it yourself, or keep the email self-contained).
- A release without `app-assets/public/` has no assets, even if an earlier release had some: every `$asset` call throws [`AssetsNotAvailableError`](#errors).

---

## `$asset` API

**Backend only.** `$asset` is one of the [backend Appnest functions](appnest-functions/backend/README.md). Full reference: [asset.md](appnest-functions/backend/asset.md).

```javascript
const { AppnestFunctions } = require('@sparrowengg/appnest-app-sdk-utils');
const { $asset } = AppnestFunctions;
```

| Method | Signature | Returns | Sync? |
|--------|-----------|---------|-------|
| [getUrl](#geturl) | `$asset.getUrl({ path })` | `string` — the file's URL | Sync |
| [getBaseUrl](#getbaseurl) | `$asset.getBaseUrl()` | `string` — the base URL, ending in `/` | Sync |
| [list](#list) | `$asset.list({ path? })` | `Promise<{ keys: string[] }>` | Async |
| [exists](#exists) | `$asset.exists({ path })` | `Promise<{ exists: boolean }>` | Async |

The shapes follow [`$file`](appnest-functions/backend/file.md) (`{ path }` in, `{ keys }` / `{ exists }` out), but `$asset` has no `visibility`: every asset is public. Results are returned directly, with no `{ data, status }` wrapper.

**Path rules (every method):** `path` is a string relative to `app-assets/public/`. A path containing `..` or `\`, starting with `/`, or starting with a URL scheme (e.g. `https://`) throws an `Error`.

### getUrl

```javascript
const logoUrl = $asset.getUrl({ path: 'images/logo.png' });
// 'https://…/images/logo.png'
```

- Returns the public URL of one file, synchronously. No network call is made.
- It **doesn't check that the file exists**: a typo gives a URL that returns 404. Use [`exists`](#exists) when the path comes from user input or data.

### getBaseUrl

```javascript
const baseUrl = $asset.getBaseUrl();   // ends in '/'
```

- Returns the URL that every asset URL starts with, synchronously.
- Use it to hand your frontend **one** value instead of many URLs, or as `<base href>` in an HTML template so relative links resolve to your assets.
- It's opaque: join a relative path to it, but never take it apart or store it.

### list

```javascript
const { keys } = await $asset.list({ path: 'templates/' });
// keys: ['templates/welcome.html', 'templates/reminder.html']

const { keys: all } = await $asset.list({ path: '' });   // every file
```

- `path` is a **folder prefix**. A non-empty path without a trailing `/` gets one (`'templates'` lists `templates/…` and its subfolders). `''`, the default, lists every file.
- Keys are **relative to `app-assets/public/`** (not to the folder you listed), so you can pass any key straight to `getUrl`. This differs from `$file.list`, which returns keys relative to the listed folder.
- Only files are listed, not folders. An empty or unknown folder gives `{ keys: [] }`.

### exists

```javascript
const { exists } = await $asset.exists({ path: 'images/logo.png' });
```

- Exact match on the file path. Returns `{ exists: false }` for a file that isn't in this release, and for a folder path (it doesn't throw).

### Errors

| Error | When |
|-------|------|
| `AssetsNotAvailableError` (`error.code === 'ASSETS_NOT_AVAILABLE'`) | The running release has no `app-assets/public/` folder. Thrown by **every** method, including `getUrl` and `getBaseUrl`. |
| `Error` (path message) | `path` is missing, not a string, or breaks the path rules above. |
| Rejected promise from `list` / `exists` | The platform couldn't answer. `error.response.status` is the HTTP status. Calling again retries. |

If your app can run without assets, catch `AssetsNotAvailableError` and fall back:

```javascript
let logoUrl = null;
try {
  logoUrl = $asset.getUrl({ path: 'images/logo.png' });
} catch (error) {
  if (error.code !== 'ASSETS_NOT_AVAILABLE') throw error;
}
```

### Reading a file's contents

`$asset` has no "get content" method. Fetch the URL like any other HTTP resource with [`$fetch`](appnest-functions/backend/fetch.md):

```javascript
const { $asset, $fetch } = AppnestFunctions;

const loadTemplate = async (path) => {
  const res = await $fetch.request({ url: $asset.getUrl({ path }), method: 'GET' });
  if (res.status !== 200) throw new Error(`Asset ${path} returned ${res.status}`);
  return res.body;   // the file's contents
};

const welcomeHtml = await loadTemplate('templates/welcome.html');
```

The contents of a URL never change within a release, so you may cache what you fetched for as long as your function runs.

### Example: an email with a logo

```javascript
const { AppnestFunctions } = require('@sparrowengg/appnest-app-sdk-utils');
const { $asset, $fetch } = AppnestFunctions;

const buildWelcomeEmail = async ({ payload }) => {
  const res = await $fetch.request({
    url: $asset.getUrl({ path: 'templates/welcome.html' }),
    method: 'GET',
  });
  const html = res.body
    .replace('{{logoUrl}}', $asset.getUrl({ path: 'images/logo.png' }))
    .replace('{{name}}', payload.name);
  return { html };
};

module.exports = { buildWelcomeEmail };
```

Remember that an email outlives the release it was sent from; see [URLs change on every deploy](#urls-change-on-every-deploy).

### Not available yet

- **Frontend access.** There is no `$asset` in `window.AppnestFunctions`. See [Showing assets in the frontend](#showing-assets-in-the-frontend).
- **File metadata** (`getMetadata`: size, content type, hash).
- **Private assets** (`app-assets/private/`, signed URLs, `visibility`). `private/` is rejected at upload until then.

---

## Showing assets in the frontend

The frontend can't call `$asset`. Return the URLs from a backend function and use them in your UI:

```javascript
// app-backend: exported from server.js and declared in backend_api_functions
const getBranding = async () => ({
  logoUrl: $asset.getUrl({ path: 'images/logo.png' }),
  guideUrl: $asset.getUrl({ path: 'docs/guide.pdf' }),
});
```

```jsx
// app-frontend
const { body } = await window.AppnestFunctions.$app.backend({ functionName: 'getBranding', functionPayload: {} });
// <img src={body.logoUrl} />
```

To pass many files, return `$asset.getBaseUrl()` once and join relative paths to it in the frontend. Fetch the URLs fresh each time the UI loads; don't save them.

---

## Local development and packing

### `app start`

`appnest-development-engine app start` serves your local `app-assets/public/` folder through the local proxy:

```text
http://localhost:<port>/app-assets/public/<path>
```

(`<port>` is `APPNEST_ENGINE_PORT`, default 3456; see [configuration.md](appnest-development-engine/configuration.md#local-urls).)

- `$asset.getUrl` and `getBaseUrl` return these local URLs, and `list` and `exists` read the local folder as it is, so files you add show up right away. The same backend code works locally and after deploy.
- Locally there is no deploy, so the URL of a file stays the same between edits. After deploy it won't: don't let a URL you saw locally end up in your code.

### `app validate` and `app pack`

- `app validate` checks every [rule](#rules-and-limits), lists **every** problem, and exits with code **1** if there are any.
- `app pack` runs the same checks and adds **`app-assets/public/`** to the zip **as-is**, next to `app-frontend/`, `app-backend/` and `manifest.json`. You don't need `PACK_FOLDER_PATHS` for it.
- Upload the zip as usual. Assets go live together with the code of that release; there is no separate asset upload.
