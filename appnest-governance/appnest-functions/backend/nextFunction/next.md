# $next — Invoke Function SDK

The `$next` module lets you invoke another function by name, optionally after a delay (in seconds).

---

## Import / Usage

```javascript
const { AppnestFunctions } = require('@sparrowengg/appnest-app-sdk-utils');
const { $next } = AppnestFunctions;
```

---

## API

### run

| Method | Signature | Description |
|--------|-----------|-------------|
| run | `$next.run({ functionName, functionPayload, delay })` | Invoke a function by name. Optionally delay execution. |

**Parameters:**

| Parameter       | Type   | Required | Default | Description |
|-----------------|--------|----------|---------|-------------|
| functionName    | string | yes      | —       | Target function name to run. |
| functionPayload | object | yes      | —       | Data passed to the function. |
| delay           | number | no       | 0       | Delay in seconds before execution (>= 0). |

**Return:** `Promise<object>` — the API response body directly (e.g. `{ success: boolean }`). No `{ data, status }` wrapper.

---

## Examples

**Run immediately:**

```javascript
await $next.run({
  functionName: 'syncContacts',
  functionPayload: { accountId: 12345 }
});
```

**Run after delay (e.g. 10 seconds):**

```javascript
await $next.run({
  functionName: 'syncContacts',
  functionPayload: { accountId: 12345 },
  delay: 10
});
```
