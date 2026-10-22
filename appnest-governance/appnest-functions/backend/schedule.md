# $schedule — Schedule SDK

The `$schedule` module lets you create, get, update, pause, resume, and delete scheduled jobs.

**Schedule job kinds (`jobType`):** `ONE_TIME` (run once at a time), `CRON` (cron expression), `RECURRING` (fixed interval).

**Import:** `const { $schedule } = AppnestFunctions;` — see [Import and rules](README.md#import).

**Handling a run:** each time a job fires, the platform calls the handler you registered for the **`onScheduledEvent`** event under `event_listener_functions` in `manifest.json` (see Manifest Rules in `appnest-governance`). Register it before creating schedules.

---

## Common Fields

| Field | Type   | Required | Constraints |
|-------|--------|----------|-------------|
| jobName    | string | yes      | Non-empty, max 30 characters. |
| jobType    | string | yes (create/update) | `ONE_TIME` \| `CRON` \| `RECURRING`. Optional for `get`, `pause`, `resume`, `delete`; validated if passed. |
| jobPayload | object | yes (create/update) | At least one key. Delivered to your `onScheduledEvent` handler on each run. |
| jobConfig  | object | yes (create/update) | Type-specific scheduling fields (see below). |

A job is identified by **`jobName`** alone: names are unique per installation, whatever the type. `update` still needs `jobType`, and it must match the job's type. If you pass `jobType` to `delete`, it must match too.

---

## `jobConfig` (create / update)

Pass a single **`jobConfig`** object. Allowed keys depend on **`jobType`** (extras are rejected).

**ONE_TIME —** `jobConfig: { runAt }`

| Field  | Type   | Required | Description |
|--------|--------|----------|-------------|
| runAt  | string | yes      | Date-time string, ISO 8601 recommended (e.g. `2026-02-05T10:00:00Z`). |

Do not include `cronExpression` or `repeat`.

**CRON —** `jobConfig: { cronExpression }`

| Field          | Type   | Required | Description |
|----------------|--------|----------|-------------|
| cronExpression | string | yes      | Valid 5-field cron expression (e.g. `0 * * * *` for hourly). |

Do not include `runAt` or `repeat`.

**RECURRING —** `jobConfig: { repeat }`

| Field  | Type   | Required | Description |
|--------|--------|----------|-------------|
| repeat | object | yes      | `{ frequency: number, timeUnit: string }` |

| repeat field | Type   | Required | Constraints |
|--------------|--------|----------|-------------|
| frequency    | number | yes      | Integer >= 1; if `timeUnit` is `MINUTES`, >= 10. |
| timeUnit     | string | yes      | `MINUTES` \| `HOURS` \| `DAYS` \| `WEEKS` \| `MONTHS` \| `YEARS` |

Do not include `runAt` or `cronExpression`.

---

## API

| Method | Signature | Description |
|--------|-----------|-------------|
| create | `$schedule.create({ jobName, jobType, jobPayload, jobConfig })` | Create a schedule. Returns nothing. |
| get    | `$schedule.get({ jobName, jobType? })` | Get a schedule by jobName. Returns `{ jobName, jobType, jobPayload, status }`. |
| update | `$schedule.update({ jobName, jobType, jobPayload, jobConfig })` | Update existing schedule. Returns nothing. |
| pause  | `$schedule.pause({ jobName, jobType? })` | Pause a schedule. Returns nothing. |
| resume | `$schedule.resume({ jobName, jobType? })` | Resume a paused schedule. Returns nothing. |
| delete | `$schedule.delete({ jobName, jobType? })` | Delete a schedule. Returns nothing. |

**Return:** `get` returns `Promise<{ jobName, jobType, jobPayload, status }>`, where `status` is `ACTIVE` \| `PAUSED` \| `COMPLETED`. The other methods return `Promise<void>` and resolve when the operation completes.

**Errors:** `get`, `update`, `pause`, `resume` and `delete` reject with a 404 error if no job with that `jobName` exists.

---

## Examples

**ONE_TIME:**

```javascript
await $schedule.create({
  jobName: 'dailyReportOnce',
  jobType: 'ONE_TIME',
  jobConfig: { runAt: '2026-02-05T10:00:00Z' },
  jobPayload: { reportType: 'daily' }
});
```

**CRON:**

```javascript
await $schedule.create({
  jobName: 'hourlyReport',
  jobType: 'CRON',
  jobConfig: { cronExpression: '0 * * * *' },
  jobPayload: { reportType: 'hourly' }
});
```

**RECURRING:**

```javascript
await $schedule.create({
  jobName: 'everyTwoHours',
  jobType: 'RECURRING',
  jobConfig: { repeat: { frequency: 2, timeUnit: 'HOURS' } },
  jobPayload: { task: 'sync' }
});
```

**Get:**

```javascript
const job = await $schedule.get({ jobName: 'hourlyReport' });
// { jobName: 'hourlyReport', jobType: 'CRON', jobPayload: { reportType: 'hourly' }, status: 'ACTIVE' }
```

**Update:**

```javascript
await $schedule.update({
  jobName: 'hourlyReport',
  jobType: 'CRON',
  jobConfig: { cronExpression: '0 */2 * * *' },
  jobPayload: { reportType: 'hourly' }
});
```

**Pause / Resume:**

```javascript
await $schedule.pause({ jobName: 'hourlyReport' });
await $schedule.resume({ jobName: 'hourlyReport' });
```

**Delete:**

```javascript
await $schedule.delete({ jobName: 'hourlyReport' });
```
