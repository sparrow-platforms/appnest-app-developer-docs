# $asset — App Assets SDK

The `$asset` module gives your backend the URLs of the static files your app ships in `app-assets/public/`: images, HTML pages, email templates, documents. It is read-only: assets ship with your code and never change after release. For files your app creates at runtime, use [`$file`](file.md).

**Import:** `const { $asset } = AppnestFunctions;` — see [Import and rules](README.md#import).

---

## Where the files go

```
my-app/
  manifest.json
  app-frontend/
  app-backend/
  app-assets/
    public/              ← put your files here; any file type, any nesting
      images/logo.png
      templates/welcome.html
      docs/guide.pdf
```

`app pack` adds `app-assets/public/` to your app. Every file in it is **public**: anyone with its URL can read it. Never put secrets, keys or customer data there.

---

## Paths

Every `path` is relative to `app-assets/public/`: write `images/logo.png`, not `public/images/logo.png` or `app-assets/public/images/logo.png`.

A path must not contain an empty, `.` or `..` folder (`a//b.png`, `./a`, `a/../b`) or `\`, start with `/`, or start with a URL scheme such as `https:`. Only the folder you pass to `list` may end in one `/` (`'images/'`). Such a path throws an `Error` before anything else happens.

---

## API

| Method | Signature | Description | Return |
|--------|-----------|-------------|--------|
| getUrl | `$asset.getUrl({ path })` | URL of one asset. Builds the URL only; it doesn't check that the file exists. | `string` |
| getBaseUrl | `$asset.getBaseUrl()` | Base URL of your assets, ending in `/`. Use it to hand your frontend one base URL, or as `<base href>` in a template. Treat it as an opaque prefix: don't parse it or build on its structure. | `string` |
| list | `$asset.list({ path? })` | Paths of the assets under a folder. `path` is a folder: `'templates'` and `'templates/'` both list `templates/` and its subfolders. `''` (the default) lists everything. | `Promise<{ keys: string[] }>` |
| exists | `$asset.exists({ path })` | Whether a file exists at exactly this path. A folder name such as `'images'` returns `false`; a path ending in `/` throws. | `Promise<{ exists: boolean }>` |

`getUrl` and `getBaseUrl` are **synchronous**: no `await`. `list` and `exists` are async.

`list` returns keys relative to `app-assets/public/` (for example `templates/welcome.html` when listing `templates/`), so each key can go straight into `getUrl`. This differs from `$file.list`, which returns keys relative to the listed folder.

---

## Errors

| When | What is thrown |
|------|----------------|
| Your app version has no `app-assets/public/` folder | `AssetsNotAvailableError`, from every method. Check `error.code === 'ASSETS_NOT_AVAILABLE'` (or `error.name === 'AssetsNotAvailableError'`). |
| `list` or `exists` aren't available for this installation | `AssetsListUnavailableError`. Check `error.code === 'ASSETS_LIST_UNAVAILABLE'`. `getUrl` and `getBaseUrl` still work. |
| The path breaks a rule above, or is missing | `Error` with a message describing the problem |
| The platform can't answer `list` or `exists`, or takes more than 10 seconds | `Error` (`$asset: could not load the assets index …`) with `status` (the HTTP status, or `null` when there was no response, as on a timeout) and `code` (the platform's error code when it sent one, otherwise the network error's, such as `'ECONNABORTED'` for a timeout, otherwise `null`). It carries no request details, so it is safe to log or return. Calling again retries. |

---

## Asset URLs belong to one release

Each release of your app gets its own copy of its assets, at its own URLs. When you release a new version, `getUrl` returns new URLs; the URLs of the previous release keep working for now, but they aren't guaranteed to last forever. Content that lives a long time, such as an email you send, can outlive them.

Within one release a URL never changes and its content never changes, so you can cache what you fetch from it.

---

## Examples

```javascript
const { AppnestFunctions } = require('@sparrowengg/appnest-app-sdk-utils');
const { $asset } = AppnestFunctions;

// URL of one image, e.g. to return to your frontend for an <img src>
const logoUrl = $asset.getUrl({ path: 'images/logo.png' });

// One base URL for the frontend to build its own asset URLs from
const assetsBaseUrl = $asset.getBaseUrl();

// Read a template's contents: fetch its URL with Node's built-in fetch (not $fetch, see below)
const res = await fetch($asset.getUrl({ path: 'templates/welcome.html' }));
if (!res.ok) throw new Error(`Template returned HTTP ${res.status}`);   // 404 if the file isn't in this release
const welcomeHtml = await res.text();

// Every file under templates/
const { keys } = await $asset.list({ path: 'templates/' });
// keys: [ 'templates/welcome.html', 'templates/reminder.html' ]

// Check a file before using it
const { exists } = await $asset.exists({ path: 'docs/guide.pdf' });
```

**Reading contents: use `fetch`, not `$fetch`.** `$fetch` only reaches the domains your app declares, and the asset host isn't one of them. Fetching a URL that `$asset.getUrl()` returned (or one joined to `getBaseUrl()`) is the only exception to the rule that backend I/O goes through Appnest functions ([Rules](README.md#rules)); for every other URL, keep using `$fetch`. Node's built-in `fetch` needs nothing in `app-backend/package.json`.

Handling an app version without assets:

```javascript
try {
  const url = $asset.getUrl({ path: 'images/logo.png' });
} catch (error) {
  if (error.code === 'ASSETS_NOT_AVAILABLE') {
    // this app version ships no app-assets/public/ folder
  } else {
    throw error;
  }
}
```

---

## Local development (`app start`)

With `appnest-development-engine app start`, `$asset` uses your project's local `app-assets/public/` folder, served through your development app URL (your tunnel to the local proxy, for example ngrok):

- For events the platform sends to your development app, `getUrl` and `getBaseUrl` return `<development app URL>/app-assets/public/...` URLs, so they work in your frontend, in emails and for other users while `app start` and the tunnel are running.
- `http://localhost:<port>/app-assets/public/...` URLs are only a fallback, for a request that didn't come from the platform (for example one you send to `/app-backend/<function>` by hand). They work only on your machine.
- `list` and `exists` read the folder as it is, so files you add show up right away. They see only the files `app validate` accepts and the local server serves: hidden files and folders (names starting with `.`), symlinks, files in an `app-runtime` folder, and paths that break a rule above or are over 256 characters are left out. Run `app validate` to see why a file is missing.
- Reading contents with `fetch` works the same way locally and after deploy.
