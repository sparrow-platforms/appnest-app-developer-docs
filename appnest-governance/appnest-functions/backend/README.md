# Backend Appnest Functions

> **Public developer docs.** Everything in this folder is published to the public `appnest-app-developer-docs` repository. Keep it to what app developers write and get back — no SDK internals.

**Backend Appnest functions** are the platform-backed modules you call from your app's **backend** code (`app-backend/server.js` handlers): `$db`, `$fetch`, `$file`, `$next`, `$schedule`, and `getTraceId`. They are not separate deployable units — Appnest provides the implementation at runtime, and you call them with `await`.

This page is the **index**: the import, the rules, and every method with its signature. Each module page has the full parameter rules, limits, and examples.

---

## Import

```javascript
const { AppnestFunctions } = require('@sparrowengg/appnest-app-sdk-utils');
const { $db, $fetch, $file, $next, $schedule, getTraceId } = AppnestFunctions;
```

## Rules

- **Destructure from `AppnestFunctions`**, not from the package root. `const { $db } = require('@sparrowengg/appnest-app-sdk-utils')` is incorrect.
- **Do not add `@sparrowengg/appnest-app-sdk-utils` to `app-backend/package.json`.** The platform provides it at runtime.
- **Use these modules for all backend I/O** — no `axios`, `fetch`, or other HTTP/storage clients.
- Import only the modules you need.

---

## Modules

| Module | Purpose | Reference |
|--------|---------|-----------|
| **$db** | Key/value storage by type: string, number, list, map (with atomic nested-path operations), boolean | [db.md](db.md) |
| **$fetch** | Outbound HTTP requests, with installation-parameter and OAuth token placeholders | [fetch.md](fetch.md) |
| **$file** | File storage via signed URLs: upload, download, delete, list, exists | [file.md](file.md) |
| **$next** | Invoke another function by name, now or after a delay | [next.md](next.md) |
| **$schedule** | Scheduled jobs: one-time, cron, or recurring | [schedule.md](schedule.md) |
| **getTraceId** | Trace ID of the current request, for logging and correlation | [below](#gettraceid) |

---

## Method index

### $db — [db.md](db.md)

`key` is required on every call: a non-empty string, max 1000 characters.

| Method | Signature | Returns |
|--------|-----------|---------|
| get | `$db.get({ key })` | `Promise<string>` |
| set | `$db.set({ key, value })` — `value`: string | `Promise<void>` |
| delete | `$db.delete({ key })` | `Promise<void>` |
| typeOf | `$db.typeOf({ key })` | `Promise<string>` — `'map' \| 'list' \| 'number' \| 'string' \| 'boolean' \| 'null' \| 'absent'` |
| byteSize | `$db.byteSize({ key })` | `Promise<number>` |
| string.get | `$db.string.get({ key })` | `Promise<string>` |
| string.set | `$db.string.set({ key, value })` — `value`: string | `Promise<void>` |
| string.delete | `$db.string.delete({ key })` | `Promise<void>` |
| number.get | `$db.number.get({ key })` | `Promise<number>` |
| number.set | `$db.number.set({ key, value })` — `value`: finite number | `Promise<void>` |
| number.delete | `$db.number.delete({ key })` | `Promise<void>` |
| number.increment | `$db.number.increment({ key, value? })` | `Promise<void>` |
| number.decrement | `$db.number.decrement({ key, value? })` | `Promise<void>` |
| list.get | `$db.list.get({ key })` | `Promise<array>` |
| list.set | `$db.list.set({ key, value })` — `value`: array | `Promise<void>` |
| list.append | `$db.list.append({ key, value })` — `value`: array | `Promise<void>` |
| list.prepend | `$db.list.prepend({ key, value })` — `value`: array | `Promise<void>` |
| list.updateItemAtIndex | `$db.list.updateItemAtIndex({ key, value, index })` | `Promise<void>` |
| list.removeItemAtIndex | `$db.list.removeItemAtIndex({ key, index })` | `Promise<void>` |
| map.get | `$db.map.get({ key })` | `Promise<object>` |
| map.set | `$db.map.set({ key, value })` — `value`: JSON object | `Promise<void>` |
| map.delete | `$db.map.delete({ key })` | `Promise<void>` |
| map.updateItem | `$db.map.updateItem({ key, path, value })` | `Promise<void>` |
| map.removeItem | `$db.map.removeItem({ key, path, recursive? })` | `Promise<void>` |
| map.increment | `$db.map.increment({ key, path, value })` | `Promise<void>` |
| map.decrement | `$db.map.decrement({ key, path, value })` | `Promise<void>` |
| map.append | `$db.map.append({ key, path, value })` — `value`: non-empty array | `Promise<void>` |
| map.prepend | `$db.map.prepend({ key, path, value })` — `value`: non-empty array | `Promise<void>` |
| map.apply | `$db.map.apply({ key, ops })` — atomic batch | `Promise<void>` |
| map.typeOf | `$db.map.typeOf({ key, path })` | `Promise<string>` |
| map.byteSize | `$db.map.byteSize({ key, path })` | `Promise<number>` |
| boolean.get | `$db.boolean.get({ key })` | `Promise<boolean>` |
| boolean.set | `$db.boolean.set({ key, value })` — `value`: boolean | `Promise<void>` |
| boolean.delete | `$db.boolean.delete({ key })` | `Promise<void>` |

`path` is an array of map keys (strings) and, as the last segment only, a list index (non-negative integer) — see [Nested paths](db.md#nested-paths).

### $fetch — [fetch.md](fetch.md)

| Method | Signature | Returns |
|--------|-----------|---------|
| request | `$fetch.request({ url, method, headers?, body?, options? })` — `method`: `GET` \| `POST` \| `PUT` \| `DELETE` \| `PATCH`; `options`: `{ isOauth?, maxAttempts?, additionalOptions? }` | `Promise<{ headers, body, status }>` |

Use `<%=iparams.<key>%>` for installation parameters and `<%=user_oauth.access_token%>` for the OAuth token — never hardcode them.

### $file — [file.md](file.md)

`visibility` is optional on every call: `'PRIVATE'` (default) or `'PUBLIC'`.

| Method | Signature | Returns |
|--------|-----------|---------|
| getUploadUrl | `$file.getUploadUrl({ path, visibility? })` | `Promise<{ preSignedUrl: string }>` |
| getDownloadUrl | `$file.getDownloadUrl({ path, visibility? })` | `Promise<{ preSignedUrl: string }>` |
| delete | `$file.delete({ path, visibility? })` | `Promise<{ deleted: boolean }>` |
| list | `$file.list({ path, visibility? })` | `Promise<{ keys: string[] }>` |
| exists | `$file.exists({ path, visibility? })` | `Promise<{ exists: boolean }>` |

### $next — [next.md](next.md)

| Method | Signature | Returns |
|--------|-----------|---------|
| run | `$next.run({ functionName, functionPayload, delay? })` — `delay` in seconds, default `0` | `Promise<object>` — e.g. `{ success: boolean }` |

### $schedule — [schedule.md](schedule.md)

`jobName` (max 30 characters) identifies a job. `jobType` (`ONE_TIME` \| `CRON` \| `RECURRING`) is required for create and update, and optional for get, pause, resume and delete.

| Method | Signature | Returns |
|--------|-----------|---------|
| create | `$schedule.create({ jobName, jobType, jobPayload, jobConfig })` | `Promise<void>` |
| get | `$schedule.get({ jobName, jobType? })` | `Promise<{ jobName, jobType, jobPayload, status }>` |
| update | `$schedule.update({ jobName, jobType, jobPayload, jobConfig })` | `Promise<void>` |
| pause | `$schedule.pause({ jobName, jobType? })` | `Promise<void>` |
| resume | `$schedule.resume({ jobName, jobType? })` | `Promise<void>` |
| delete | `$schedule.delete({ jobName, jobType? })` | `Promise<void>` |

`jobConfig` by type: `ONE_TIME` → `{ runAt }`, `CRON` → `{ cronExpression }`, `RECURRING` → `{ repeat: { frequency, timeUnit } }`.

### getTraceId

| Function | Signature | Returns |
|----------|-----------|---------|
| getTraceId | `getTraceId()` | `string` — the current request's trace ID |

Include it in your logs so a single request can be followed end to end, and quote it when contacting support.

---

## Return values

All methods return the result directly — there is no `{ data, status }` wrapper.

| Module | Returns |
|--------|---------|
| **$db** | Reads return the stored value (string, number, array, object, boolean); `typeOf` returns a type label and `byteSize` a number. Writes and deletes return nothing. |
| **$fetch** | `{ headers, body, status }` of the outbound response. |
| **$file** | A small result object, e.g. `{ preSignedUrl }`, `{ deleted }`, `{ keys }`, `{ exists }`. |
| **$next** | The response body, e.g. `{ success: true }`. |
| **$schedule** | Nothing. |

On invalid input every method throws an `Error` with a message describing the problem (e.g. `Key must be a non-empty string`).

---

## Example: using it in a backend function

```javascript
const { AppnestFunctions } = require('@sparrowengg/appnest-app-sdk-utils');
const { $db, $fetch, $next, getTraceId } = AppnestFunctions;

// Exported from app-backend/server.js and registered in manifest.json
const syncContacts = async ({ payload }) => {
  const traceId = getTraceId();
  console.log(`syncContacts started, traceId=${traceId}`);

  const res = await $fetch.request({
    url: '<%=iparams.api_url%>/contacts',
    method: 'GET',
    headers: { Authorization: 'Bearer <%=iparams.api_key%>' },
  });

  // atomic update of one field inside a stored object
  await $db.map.increment({ key: 'syncState', path: ['runs'], value: 1 });
  await $db.string.set({ key: 'lastSyncAt', value: new Date().toISOString() });

  // continue in another function, 10 seconds from now
  await $next.run({ functionName: 'processContacts', functionPayload: { status: res.status }, delay: 10 });

  return { synced: true, traceId };
};

module.exports = { syncContacts };
```
