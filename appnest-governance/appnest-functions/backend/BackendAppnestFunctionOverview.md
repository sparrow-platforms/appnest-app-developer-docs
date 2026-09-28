# Backend Appnest functions — overview

**Backend Appnest functions** mean the **AppnestFunctions** SDK: platform-backed modules you import in **server-side** Appnest code (for example HTTP or event handlers). They are not separate deployable units—they are the `$db`, `$fetch`, `$file`, `$next`, `$schedule`, and `getTraceId` objects you load with `require('@sparrowengg/appnest-app-sdk-utils')` and call with `await`; Appnest supplies the implementation at runtime.

**What they are for:** persisting and reading app state, calling outbound APIs with supported options, uploading and downloading files via signed URLs, handing off work to another function (immediately or after a delay), managing scheduled jobs (one-time, cron, or recurring), and reading a **trace ID** so logs and support can follow a single request.

Together they cover **storage** (`$db`), **outbound HTTP** (`$fetch`), **files** via signed URLs (`$file`), **chaining work** to another function (`$next`), **scheduled jobs** (`$schedule`), and **request correlation** (`getTraceId`). Pick only the modules you need from `AppnestFunctions`.

Use this page for orientation; for **exact signatures, parameters, return types, and constraints**, use [BackendAppnestFunctions.md](./BackendAppnestFunctions.md) as the single source of truth.

## Import

```javascript
const { AppnestFunctions } = require('@sparrowengg/appnest-app-sdk-utils');
const { $db, $fetch, $file, $next, $schedule, getTraceId } = AppnestFunctions;
```

- Destructure from `AppnestFunctions`, not from the package root.
- The SDK is provided at runtime by the Appnest platform — **do not** add `@sparrowengg/appnest-app-sdk-utils` to `app-backend/package.json`.

## Modules at a glance

| Module | Purpose |
|--------|---------|
| **$db** | Key/value by type: string, number (with increment/decrement), list, map, boolean. |
| **$fetch** | Outbound HTTP via `request({ url, method, headers, body, options? })`. |
| **$file** | File storage with signed URLs: upload, download, delete, list, exists (`path`, optional `visibility`). |
| **$next** | Invoke another function: `run({ functionName, payload, delay? })` (delay in seconds). |
| **$schedule** | Scheduled jobs: `create`, `get`, `update`, `pause`, `resume`, `delete` — types `ONE_TIME`, `CRON`, `RECURRING`. |
| **getTraceId** | String trace ID for logging and correlation. |

