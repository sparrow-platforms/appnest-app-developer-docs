# $schedule — Schedule SDK

The `$schedule` module lets you create, get, update, pause, resume, and delete scheduled jobs.

**Schedule job kinds (`jobType`):** `ONE_TIME` (run once at a time), `CRON` (cron expression), `RECURRING` (fixed interval).

---

## Import / Usage

```javascript
const { AppnestFunctions } = require('@sparrowengg/appnest-app-sdk-utils');
const { $schedule } = AppnestFunctions;
```

---

## Common Fields

| Field | Type   | Required | Constraints |
|-------|--------|----------|-------------|
| jobName    | string | yes      | Non-empty, max 30 characters. |
| jobType    | string | yes      | `ONE_TIME` \| `CRON` \| `RECURRING` |
| jobPayload | object | yes (create/update) | At least one key; delivered to the scheduled handler as `data` on the wire. |
| jobConfig  | object | yes (create/update) | Type-specific scheduling fields (see below). |

The SDK maps `jobName` → `name`, `jobType` → `type`, and `jobPayload` → `data` on the wire. Fields inside `jobConfig` are sent as top-level `runAt`, `cronExpression`, and `repeat` (with `repeat` transformed to `{ every, unit }` for `RECURRING`).

---

## `jobConfig` (create / update)

Pass a single **`jobConfig`** object. Allowed keys depend on **`jobType`** (extras are rejected).

**ONE_TIME —** `jobConfig: { runAt }`

| Field  | Type   | Required | Description |
|--------|--------|----------|-------------|
| runAt  | string | yes      | Date-time string (e.g. ISO 8601: `YYYY-MM-DDTHH:mm:ssZ`). |

Do not include `cronExpression` or `repeat`.

**CRON —** `jobConfig: { cronExpression }`

| Field          | Type   | Required | Description |
|----------------|--------|----------|-------------|
| cronExpression | string | yes      | Valid cron expression (e.g. `0 * * * *` for hourly). |

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
| get    | `$schedule.get({ jobName, jobType })` | Get schedule by jobName and jobType. Returns nothing. |
| update | `$schedule.update({ jobName, jobType, jobPayload, jobConfig })` | Update existing schedule. Returns nothing. |
| pause  | `$schedule.pause({ jobName, jobType })` | Pause a schedule. Returns nothing. |
| resume | `$schedule.resume({ jobName, jobType })` | Resume a paused schedule. Returns nothing. |
| delete | `$schedule.delete({ jobName, jobType })` | Delete a schedule. Returns nothing. |

**Return:** All schedule methods return `Promise<void>` (no response body). They resolve when the operation completes.

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

**Get:** (returns nothing; use to ensure the schedule exists or to trigger a side effect)

```javascript
await $schedule.get({ jobName: 'hourlyReport', jobType: 'CRON' });
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
await $schedule.pause({ jobName: 'hourlyReport', jobType: 'CRON' });
await $schedule.resume({ jobName: 'hourlyReport', jobType: 'CRON' });
```

**Delete:**

```javascript
await $schedule.delete({ jobName: 'hourlyReport', jobType: 'CRON' });
```
