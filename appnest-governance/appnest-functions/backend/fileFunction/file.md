# $file — File Storage SDK

The `$file` module provides file storage helpers: signed URLs for upload/download, delete, list, and existence check.

---

## Import / Usage

```javascript
const { AppnestFunctions } = require('@sparrowengg/appnest-app-sdk-utils');
const { $file } = AppnestFunctions;
```

---

## Common Parameters

| Parameter   | Type   | Required | Default  | Description        |
|------------|--------|----------|----------|--------------------|
| path       | string | yes      | —        | File key/path      |
| visibility | string | no       | `PUBLIC` | `PUBLIC` or `PRIVATE` |

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
// Get upload URL (default PUBLIC); returns { preSignedUrl }
const upload = await $file.getUploadUrl({ path: 'reports/2026-02-09.csv' });

// Get download URL (private); returns { preSignedUrl }
const read = await $file.getDownloadUrl({ path: 'reports/2026-02-09.csv', visibility: 'PRIVATE' });
// read === { preSignedUrl: "https://...s3.amazonaws.com/...?X-Amz-Security-Token=..." }

// Delete file; returns { deleted: true } or { deleted: false }
await $file.delete({ path: 'reports/old.csv' });

// List keys under prefix; returns { keys: ["reports/2026-02-09.csv", ...] } (relative paths)
const listResult = await $file.list({ path: 'reports/' });

// Check existence; returns { exists: true } or { exists: false }
const existsResult = await $file.exists({ path: 'reports/2026-02-09.csv' });
```
