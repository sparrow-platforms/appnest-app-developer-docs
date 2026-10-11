# $file — File Storage SDK

The `$file` module provides file storage helpers: signed URLs for upload/download, delete, list, and existence check.

**Import:** `const { $file } = AppnestFunctions;` — see [Import and rules](README.md#import).

---

## Common Parameters

| Parameter   | Type   | Required | Default  | Description        |
|------------|--------|----------|----------|--------------------|
| path       | string | yes      | —        | File key/path      |
| visibility | string | no       | `PRIVATE` | `PRIVATE` or `PUBLIC` |

Files are **private by default**. Pass `visibility: 'PUBLIC'` only for files that anyone with the URL may read, and use the same `visibility` for every call on a file — a `PUBLIC` file and a `PRIVATE` file are stored separately even at the same `path`.

**Return shape:** All methods return the response object directly (no `{ data, status }` wrapper). See the Return column below.

---

## API

| Method        | Signature | Description | Return |
|---------------|-----------|-------------|--------|
| getUploadUrl  | `$file.getUploadUrl({ path, visibility })` | Get a signed URL for uploading a file. | `Promise<{ preSignedUrl: string }>` |
| getDownloadUrl| `$file.getDownloadUrl({ path, visibility })` | Get a signed URL for reading/downloading a file. | `Promise<{ preSignedUrl: string }>` |
| delete        | `$file.delete({ path, visibility })` | Delete a file at the given path. | `Promise<{ deleted: boolean }>` |
| list          | `$file.list({ path, visibility })` | List object keys for the installation; path filters what is listed (empty path uses the default listing scope). | `Promise<{ keys: string[] }>` |
| exists        | `$file.exists({ path, visibility })` | Check if a file exists at the path. | `Promise<{ exists: boolean }>` |

---

## Examples

```javascript
// Get upload URL (private by default); returns { preSignedUrl }
const upload = await $file.getUploadUrl({ path: 'reports/2026-02-09.csv' });

// Get download URL for the same private file; returns { preSignedUrl }
const read = await $file.getDownloadUrl({ path: 'reports/2026-02-09.csv' });
// read === { preSignedUrl: "https://..." }

// Public file: pass visibility on every call for that file
const logo = await $file.getUploadUrl({ path: 'assets/logo.png', visibility: 'PUBLIC' });

// Delete file; returns { deleted: true } or { deleted: false }
await $file.delete({ path: 'reports/old.csv' });

// List keys under prefix; returns { keys: ["reports/2026-02-09.csv", ...] } (relative paths)
const listResult = await $file.list({ path: 'reports/' });

// Check existence; returns { exists: true } or { exists: false }
const existsResult = await $file.exists({ path: 'reports/2026-02-09.csv' });
```
