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
        ├── pages/help.html
        └── docs/guide.pdf
```

- There is **no manifest key** for assets: the folder is enough. If `app-assets/public/` is missing or empty, the app simply has no assets.
- **Paths are relative to `app-assets/public/`.** The file above is `images/logo.png`, never `public/images/logo.png` or `app-assets/public/images/logo.png`.
- Files are published **exactly as you ship them**. There is no build step, minification or image processing.
- Assets are **read-only** at runtime and fixed for a release. To store files your app creates or users upload, use [`$file`](appnest-functions/backend/file.md).
- **HTML files render as web pages** in the browser (they are not downloaded), so you can ship pages that run on their own. SVG files render too. Every file is served with the content type that matches its **file extension**, so give each file the right extension. Text files (HTML, CSS, JavaScript, JSON, plain text and so on) are served as UTF-8 (`; charset=utf-8`), so save them in UTF-8.

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
| Path contents | No `.` or `..` folder (`./a`, `a/../b`; a name like `v1..2.png` is fine), no `\` (use `/` between folders), no leading `/`, and no `app-runtime` folder anywhere in the path |
| Paths that start like a URL scheme (`v1:logo.png`) | Rejected (also by `$asset`). Don't put `:` in the first folder or file name |
| Hidden files and folders (names starting with `.`, e.g. `.env`) | Rejected |
| `.DS_Store` files | Ignored: never packed or uploaded |
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

**Path rules (every method):** `path` is a string relative to `app-assets/public/`. A path containing a `.` or `..` folder, an empty folder name (`images//logo.png`) or `\`, starting with `/`, or starting like a URL scheme (`v1:logo.png`, `https://…`) throws an `Error`. For `list`, a single trailing `/` on the folder prefix (`'templates/'`) is fine.

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
| `AssetsListUnavailableError` (`error.code === 'ASSETS_LIST_UNAVAILABLE'`) | `list` / `exists` aren't available for this installation. `getUrl` and `getBaseUrl` still work. |
| `Error` (path message) | `path` is missing, not a string, or breaks the path rules above. |
| `Error` from `list` / `exists` (`error.status`, `error.code`) | The list of this release's files couldn't be loaded. `error.status` is the HTTP status, or `null` if there was no response (the request times out after **10 seconds**); `error.code` is an error code, or `null`. Calling again retries. |

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

`$asset` has no "get content" method. Fetch the URL from `getUrl` yourself, with Node's built-in `fetch` (nothing to add to `package.json`):

```javascript
const { AppnestFunctions } = require('@sparrowengg/appnest-app-sdk-utils');
const { $asset } = AppnestFunctions;

const readAsset = async (path) => {
  const res = await fetch($asset.getUrl({ path }));
  if (!res.ok) throw new Error(`Asset ${path} returned HTTP ${res.status}`);
  return res.text();   // res.json() for JSON; Buffer.from(await res.arrayBuffer()) for binary files
};
```

- **Use `fetch` here, not [`$fetch`](appnest-functions/backend/fetch.md).** `$fetch` only reaches the domains your app declares, and the asset host isn't one of them. Fetching a URL that `$asset.getUrl()` returned (or one joined to `getBaseUrl()`) is the **only** exception to the rule that backend I/O goes through Appnest functions. For every other URL, keep using `$fetch`.
- `getUrl` doesn't check that the file exists, so a wrong path gives **HTTP 404**. Check `res.ok` as above, or call [`exists`](#exists) first.
- The same code works under `app start`, where the URL points at your local folder.
- **Caching:** a URL's contents never change, and a new release gets new URLs, so you can cache what you read **by URL** (for example in a module-level `Map`). Locally a file keeps its URL while you edit it, so a cached copy can be out of date until you restart `app start`.

### Example: an email with a logo

```javascript
const { AppnestFunctions } = require('@sparrowengg/appnest-app-sdk-utils');
const { $asset } = AppnestFunctions;

const escapeHtml = (text) => String(text).replace(/[&<>"']/g, (c) => (
  { '&': '&amp;', '<': '&lt;', '>': '&gt;', '"': '&quot;', "'": '&#39;' }[c]
));

const buildWelcomeEmail = async ({ payload }) => {
  const res = await fetch($asset.getUrl({ path: 'templates/welcome.html' }));
  if (!res.ok) throw new Error(`Welcome template returned HTTP ${res.status}`);
  const html = (await res.text())
    .replaceAll('{{logoUrl}}', () => $asset.getUrl({ path: 'images/logo.png' }))
    .replaceAll('{{name}}', () => escapeHtml(payload.name));
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
const { AppnestFunctions } = require('@sparrowengg/appnest-app-sdk-utils');
const { $asset } = AppnestFunctions;

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

To pass many files, return `$asset.getBaseUrl()` once and join relative paths to it in the frontend. Encode each folder and file name with `encodeURIComponent` when you join (`'images/' + encodeURIComponent('my logo.png')`), as `getUrl` does, so names with spaces, `#` or `?` still work. Fetch the URLs fresh each time the UI loads; don't save them.

---

## Local development and packing

### `app start`

`appnest-development-engine app start` serves your local `app-assets/public/` folder through the local proxy, on the same port as `/app-backend` and `/app-frontend`:

```text
http://localhost:<port>/app-assets/public/<path>
```

(`<port>` is `APPNEST_ENGINE_PORT`, default 3456; see [configuration.md](appnest-development-engine/configuration.md#local-urls).)

Your development app's frontend is opened by other users, on other machines, through the development app URL you registered (your tunnel to the local proxy, for example ngrok). So for events the platform sends to a development app, `$asset` URLs use that URL, not `localhost`:

```text
<development app URL>/app-assets/public/<path>
```

- `$asset.getUrl` and `getBaseUrl` return these tunnel URLs, so they work in your frontend, in emails and for other users, while `app start` and the tunnel are running. The engine prints the base URL in use the first time a backend function runs and whenever it changes.
- `localhost` URLs are only a fallback, for a request that didn't come from the platform (for example one you send to `/app-backend/<function>` by hand); they work only on your machine.
- `list` and `exists` read the local folder as it is, so files you add show up right away. They return only files that pass the [rules](#rules-and-limits), the same files `app start` serves, so every key `list` returns can be fetched. The same backend code works locally and after deploy.
- Local serving supports HTTP Range requests, so audio and video play locally.
- Locally there is no deploy, so the URL of a file stays the same between edits. After deploy it won't: don't let a URL you saw locally end up in your code.

Keep in mind while you develop:

- **The URLs change when your tunnel URL changes.** Re-register the new URL, as you already do for the backend; URLs handed out before that (in an email you sent, a page that is still open) stop working.
- **Anyone with a URL can fetch your local assets while the tunnel is up**, the same as anyone with a URL can fetch public assets after deploy. Don't put anything in `app-assets/public/` that you wouldn't publish.
- **A tunnel's browser warning page can get in the way.** ngrok's free plan, for example, shows a warning page to browsers before the first request goes through, so an image in an email or an `<img>` on another site may get that page instead of the file. Use a tunnel or plan without that page when you test such cases.

### `app validate` and `app pack`

- `app validate` checks every [rule](#rules-and-limits), lists **every** problem, and exits with code **1** if there are any.
- `app pack` runs the same checks and adds **`app-assets/public/`** to the zip **as-is**, next to `app-frontend/`, `app-backend/` and `manifest.json`. You don't need `PACK_FOLDER_PATHS` for it.
- Upload the zip as usual. Assets go live together with the code of that release; there is no separate asset upload.
