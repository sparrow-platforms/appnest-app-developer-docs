# $db — Key/Value Database SDK

The `$db` module provides typed key/value storage operations. Use the namespaced APIs (`$db.string`, `$db.number`, `$db.list`, `$db.map`, `$db.boolean`) for full create/update support, and [`batchGet` / `batchSet` / `batchDelete`](#batch-operations--batchget-batchset-and-batchdelete) to read, write or delete many keys in one call.

**Import:** `const { $db } = AppnestFunctions;` — see [Import and rules](README.md#import).

---

## Common Rules

| Parameter | Type   | Required | Constraints                          |
|----------|--------|----------|--------------------------------------|
| `key`    | string | yes      | Non-empty, max 255 characters        |

**Return shape:** Read methods (`get`, `typeOf`, `byteSize`) return the value directly (string, number, array, object, or boolean). Create, update, and delete methods return nothing (`Promise<void>`), except `increment` / `decrement`, which return the new value. The [conditional writes](#conditional-writes--setifabsent-and-compareandset) also return nothing; when their condition fails they reject with `code: 'CONDITION_FAILED'`. The [batch methods](#batch-operations--batchget-batchset-and-batchdelete) return one result per key, or `{ count }` for an atomic write.

**Size limit:** a stored value can be at most **380KB**. Writes over the limit throw an error.

**Create before you modify:** only `set` and `setIfAbsent` create a key. Operations that change an existing value need the key to exist already, with the right type:
- `$db.number.increment` / `decrement`
- `$db.list.append` / `prepend` / `updateItemAtIndex` / `removeItemAtIndex`

On a key that doesn't exist yet they reject with a **400** whose reason is `Key '<key>' does not exist`. On a key holding another type they reject with a **400** whose reason is `Key '<key>' value is not of type <type>`. These errors keep the usual shape: `e.message` is `Request failed with status code 400` and the reason is in `e.response.data.message` (see [Return values](README.md#return-values)). Create the key first, or check it with `$db.typeOf`.

Create it with `setIfAbsent` when the code runs on every invocation. A plain `set` there resets the value each time, and two invocations racing on first install can lose an update:

```javascript
// only the first caller writes 0; later callers get CONDITION_FAILED, which here just means "already created"
await $db.number.setIfAbsent({ key: 'counter', value: 0 }).catch((e) => {
  if (e.code !== 'CONDITION_FAILED') throw e;
});
await $db.number.increment({ key: 'counter' });
```

---

## Conditional writes — setIfAbsent and compareAndSet

Both are a single atomic write with a condition: there is no separate read, so two callers can't both win. Each one resolves with nothing when it writes, and **rejects with `code: 'CONDITION_FAILED'`** (HTTP 412) when the condition fails. That's the same error `map.apply` throws for a failed condition, so one check covers every conditional write.

| Method | Available on | Writes only if |
|--------|--------------|----------------|
| `setIfAbsent({ key, value })` | `string`, `number`, `boolean`, `list`, `map` | nothing is stored at the key yet |
| `compareAndSet({ key, expected, value })` | `string`, `number`, `boolean` | the stored value equals `expected` |
| `$db.map.setIfAbsent({ key, path?, value })` | `map` | nothing is stored at the leaf (or, without `path`, at the key) |
| `$db.map.compareAndSet({ key, path, expected, value })` | `map` | the leaf equals `expected` |

**When the condition fails**, nothing is written and the promise rejects with an `Error` carrying:

| Field | Value |
|-------|-------|
| `code` | `'CONDITION_FAILED'` |
| `message` | `Key '<key>' already holds a value` / `Key '<key>' does not hold the expected value`, or `Path '<path>' …` for a map leaf |
| `key` | the key |
| `path` | the leaf path (map calls with a `path` only) |
| `current` | what is stored there now |
| `status` | `412` |

- `current` is read just after the refused write, so another caller may already have changed it. It's left out when nothing is stored there, or when the re-read finds the condition satisfied again (another caller changed it in between). Either way, re-read and retry if you need to.
- A stored `null` counts as present for `setIfAbsent`; `current` is then `null` (`typeOf` reports it as `'null'`, not `'absent'`).
- `expected` must be a string, number or boolean. Numbers compare by value (`1` equals `1.0`); a different type never matches (`'5'` doesn't match `5`).
- Other failures are not `CONDITION_FAILED`, so don't retry them. `compareAndSet` on a missing key rejects with a **400** whose reason is `Key '<key>' does not exist`. With a `path`, the key must exist and every parent on the path must already be a map (parents are not created); otherwise the call rejects with a **400** whose reason is `Key '<key>' does not exist` or `Path '<path>' cannot be updated: a parent segment is missing or is not a map`. A 400 keeps the usual error shape: `e.message` is `Request failed with status code 400`, `e.response.status` is `400` and the reason is in `e.response.data.message`. A `setIfAbsent` path must end in a map key, not a list index.
- `value` follows the same rules as the matching `set`, including the 380KB limit.

**Example — claim a job once:**

```javascript
try {
  await $db.string.compareAndSet({ key: 'job:42', expected: 'pending', value: 'processing' });
} catch (e) {
  if (e.code === 'CONDITION_FAILED') return; // another invocation claimed it
  throw e;
}
runJob();
```

**Example — read, change, write back, retry on a lost race:**

```javascript
for (let attempt = 0; attempt < 5; attempt++) {
  const { status } = await $db.map.get({ key: 'order' });
  if (status !== 'paid') break;
  try {
    await $db.map.compareAndSet({ key: 'order', path: ['status'], expected: status, value: 'shipped' });
    break; // written
  } catch (e) {
    if (e.code !== 'CONDITION_FAILED') throw e; // a real error: don't retry
  }
}
```

**Example — a simple lock:**

```javascript
try {
  await $db.string.setIfAbsent({ key: 'lock:sync', value: invocationId });
} catch (e) {
  if (e.code !== 'CONDITION_FAILED') throw e;
  // current can be absent: the lock may have been released between the write and the re-read
  console.log(e.current !== undefined ? `sync already running in ${e.current}` : 'sync lock was held by another invocation');
  return;
}
```

---

## Batch operations — batchGet, batchSet and batchDelete

Read, write or delete many keys of any type in one call, instead of `Promise.all` over single calls.

| Method | Signature | Default return | `atomic: true` return |
|--------|-----------|----------------|-----------------------|
| batchGet | `$db.batchGet({ keys, atomic? })` | `Promise<Array<{ key, found, type?, value? } \| { key, error }>>` | the same array |
| batchSet | `$db.batchSet({ items, atomic? })` — `items`: `[{ key, type, value, condition? }]` | `Promise<Array<{ key, ok, error? }>>` | `Promise<{ count }>` |
| batchDelete | `$db.batchDelete({ keys, atomic? })` — `keys`: strings, or `{ key, condition? }` when atomic | `Promise<Array<{ key, ok, error? }>>` | `Promise<{ count }>` |

Every method takes `atomic` (default `false`):

|  | Default (`atomic: false`) | `atomic: true` |
|--|---------------------------|----------------|
| Guarantee | Best effort: each key succeeds or fails on its own | All or nothing; reads come from one consistent snapshot |
| Max keys | 100 per `batchGet`, 25 per `batchSet` / `batchDelete` | 100 |
| Max size | `batchGet` response about 5MB (the rest comes back `UNPROCESSED`) | 4MB of values per `batchSet`; a `batchGet` over the limit fails |
| Conditions | Not allowed | Allowed on `batchSet` items and `batchDelete` keys |
| Result | One result per key, in request order | Resolves only if everything succeeds; otherwise rejects naming `index` and `key` |
| Cost | 1× | 2× |

- Results come back in request order, one per key.
- Over-limit batches (keys, or a request over 5MB), duplicate keys, a missing `type` or `value`, and a `condition` without `atomic: true` throw before anything is sent. The SDK never splits a batch: call it several times for more keys.
- Each key follows the same rules as its single-key call: `key` max 255 characters and not blank (a key of only spaces or other whitespace throws `Key cannot be blank`), and each `batchSet` value the same validation as the matching typed `set`, including the 380KB limit. `batchSet` replaces the whole value, like `set`.
- In default mode, a per-key `error` is `{ code, message }`. `code` is `UNPROCESSED` (not read or written this time, e.g. throttled; send that key again), `VALIDATION_ERROR` (e.g. the value doesn't match its `type`, or is over 380KB), `RATE_LIMITED`, or a `STORAGE_*` code.

### batchGet

```javascript
const results = await $db.batchGet({ keys: ['config', 'counter', 'tags'] });
// [
//   { key: 'config',  found: true,  type: 'MAP',    value: { theme: 'dark' } },
//   { key: 'counter', found: true,  type: 'NUMBER', value: 3 },
//   { key: 'tags',    found: false },
// ]

const snapshot = await $db.batchGet({ keys: ['job:123:cursor', 'job:123:migrated'], atomic: true });
```

- A missing key is `{ key, found: false }` and doesn't fail the call. A key that stores `null` is `{ key, found: true, type: 'NULL', value: null }`.
- `type` is `'STRING' | 'NUMBER' | 'LIST' | 'BOOLEAN' | 'MAP' | 'NULL'`.
- Default mode stops adding values at about 5MB. Keys past that, or keys the server couldn't read, come back as `{ key, error: { code: 'UNPROCESSED', message } }` with no `found`; read them again in another call.
- `atomic: true` reads every key from one consistent snapshot. A read whose values exceed the size limit fails with a **400** asking you to read fewer keys.

### batchSet

Each item names its `type`: `'STRING' | 'NUMBER' | 'LIST' | 'BOOLEAN' | 'MAP'`. `value` can't be `null` or missing, and can't be something JSON stores as `null` or drops (`NaN`, `Infinity`, a function): any of these throws `value is required for key: <key>` before anything is sent, in either mode, because the whole request would be rejected.

```javascript
const results = await $db.batchSet({
  items: [
    { key: 'status',  type: 'STRING', value: 'active' },
    { key: 'counter', type: 'NUMBER', value: 0 },
    { key: 'profile', type: 'MAP',    value: { name: 'A' } },
  ],
});
// [{ key: 'status', ok: true }, { key: 'counter', ok: true }, { key: 'profile', ok: false, error: { code, message } }]

const failed = results.filter((r) => !r.ok); // retry or report these

await $db.batchSet({ items, atomic: true }); // resolves { count: 3 }, or rejects and writes nothing
```

In default mode, an item whose value is invalid (wrong type, over 380KB) fails on its own with `VALIDATION_ERROR`; the rest are written. With `atomic: true` the same item fails the whole batch with a **400** whose reason names it, e.g. `Data size exceeds 380KB limit (key: profile)`, and nothing is written. The values in an atomic `batchSet` can total at most **4MB** (each value measured as its JSON text in UTF-8). The server writes some decimal numbers differently from JavaScript (`0.000001` as `1.0E-6`), so a batch of many such numbers close to 4MB may be rejected by the server with a **400** rather than before it is sent. In either mode, a whole batch request can be at most **5MB** (5,242,880 bytes of JSON, keys and values included); a larger one throws `Request body exceeds the 5242880 byte limit for batch storage calls` before anything is sent, so 25 values near the 380KB limit need two calls.

### batchDelete

```javascript
const results = await $db.batchDelete({ keys: ['tmp:1', 'tmp:2'] });
// [{ key: 'tmp:1', ok: true }, { key: 'tmp:2', ok: true }]

await $db.batchDelete({
  keys: [{ key: 'lock:order:42', condition: { exists: true } }, 'order:42:draft'],
  atomic: true,
}); // resolves { count: 2 }
```

A key that doesn't exist counts as deleted: `ok: true` means the key is gone.

### Conditions (atomic only)

With `atomic: true`, each `batchSet` item and each `batchDelete` key can carry a `condition`. If any condition fails, the whole batch is rolled back.

| Condition | Holds when |
|-----------|------------|
| `{ exists: true }` / `{ exists: false }` | the key holds a value (a stored `null` counts) / it doesn't. `exists: false` is setIfAbsent |
| `{ equals: v }` / `{ notEquals: v }` | the stored value equals / doesn't equal `v` (string, number or boolean). `notEquals` holds when the key is missing |
| `{ gt \| gte \| lt \| lte: n }` | numeric comparison with the stored number |
| `{ type: 'NUMBER' }` | the stored value has that type |
| `{ path: [...], <any of the above> }` | the same checks on a leaf inside a stored map (see [Nested paths](#nested-paths)) |

Several checks in one condition must all hold. There is no OR / NOT. A `condition` without `atomic: true` throws.

### Errors in atomic mode

An atomic batch either succeeds completely or rejects, and nothing is written or deleted:

| Status | `code` | Meaning | Fields |
|--------|--------|---------|--------|
| 412 | `'CONDITION_FAILED'` | A condition failed | `index`, `key`, `current` (left out when nothing is stored there) |
| 409 | `'TRANSACTION_CONFLICT'` | Concurrent writes to the same keys kept cancelling the batch; retry it | `index`, `key` |
| 400 | — | Invalid request or item; the reason is in `e.response.data.message` | — |

`index` is the position of the failing item in `items` / `keys`, and `key` its key. Both errors also carry `message` and `status`, like the other [conditional writes](#conditional-writes--setifabsent-and-compareandset). Other errors keep the usual shape (`e.response.status`, `e.response.data.message`).

**Example — take a lock and set a status together:**

```javascript
try {
  await $db.batchSet({
    items: [
      { key: 'lock:sync', type: 'MAP', value: { owner: runId, at: Date.now() }, condition: { exists: false } },
      { key: 'sync:status', type: 'STRING', value: 'running' },
    ],
    atomic: true,
  });
} catch (e) {
  if (e.code === 'CONDITION_FAILED') return; // another run holds the lock (e.index === 0)
  if (e.code === 'TRANSACTION_CONFLICT') return retryLater();
  throw e;
}
```

---

## Data Types

| Type     | Description                    |
|----------|--------------------------------|
| STRING   | string value                   |
| NUMBER   | number value                   |
| LIST     | array value                    |
| MAP      | JSON object (non-array)        |
| BOOLEAN  | boolean value                  |

---

## Top-level (string-oriented)

| Method | Signature | Description | Return |
|--------|-----------|-------------|--------|
| get    | `$db.get({ key })` | Get value by key (string) | `Promise<string>` |
| set    | `$db.set({ key, value })` | Write a string value at key (same as `$db.string.set`). | `Promise<void>` |
| delete | `$db.delete({ key })` | Delete by key | `Promise<void>` |

**Example:**

```javascript
await $db.set({ key: 'welcome', value: 'hello' });
const result = await $db.get({ key: 'welcome' }); // result is the string value
await $db.set({ key: 'welcome', value: 'hello world' });
await $db.delete({ key: 'welcome' });
```

---

## Cross-type helpers

These inspect any key regardless of its stored type.

| Method   | Signature | Description | Return |
|----------|-----------|-------------|--------|
| typeOf   | `$db.typeOf({ key })` | Stored type of the whole key | `Promise<string>` |
| byteSize | `$db.byteSize({ key })` | Serialized byte size of the whole key | `Promise<number>` |

- `typeOf` returns one of `'map' | 'list' | 'number' | 'string' | 'boolean' | 'null' | 'absent'` (`'absent'` means the key does not exist). Handy to inspect a key before acting on it — e.g. deciding whether to migrate a string key to a map.
- `byteSize` uses the same measure as the 380KB payload limit, so it's a useful gauge before a write. A missing key (or null value) is `0`.

**Example:**

```javascript
const kind = await $db.typeOf({ key: 'profile' });   // e.g. 'map'
const bytes = await $db.byteSize({ key: 'profile' }); // e.g. 73
```

---

## $db.string

| Method | Signature | Parameters | Return |
|--------|-----------|------------|--------|
| get    | `$db.string.get({ key })` | `key` (string, required) | `Promise<string>` |
| set    | `$db.string.set({ key, value })` | `key`, `value` (string, required) | `Promise<void>` |
| delete | `$db.string.delete({ key })` | `key` (string, required) | `Promise<void>` |
| setIfAbsent | `$db.string.setIfAbsent({ key, value })` | `key`, `value` (string, required) | `Promise<void>`; rejects with `CONDITION_FAILED` if a value is present — see [Conditional writes](#conditional-writes--setifabsent-and-compareandset) |
| compareAndSet | `$db.string.compareAndSet({ key, expected, value })` | `key`, `expected` (string, number or boolean), `value` (string) | `Promise<void>` |

**Example:**

```javascript
await $db.string.set({ key: 'welcome', value: 'hello' });
const res = await $db.string.get({ key: 'welcome' });
await $db.string.set({ key: 'welcome', value: 'hello world' });
await $db.string.delete({ key: 'welcome' });
```

---

## $db.number

| Method    | Signature | Parameters | Return |
|-----------|-----------|------------|--------|
| get       | `$db.number.get({ key })` | `key` (string, required) | `Promise<number>` |
| set       | `$db.number.set({ key, value })` | `key`, `value` (number, required) | `Promise<void>` |
| delete    | `$db.number.delete({ key })` | `key` (string, required) | `Promise<void>` |
| increment | `$db.number.increment({ key, value })` | `key` (string, required), `value` (finite number, optional, default `1`) | `Promise<number>` — the value after the increment |
| decrement | `$db.number.decrement({ key, value })` | `key` (string, required), `value` (finite number, optional, default `1`) | `Promise<number>` — the value after the decrement |
| setIfAbsent | `$db.number.setIfAbsent({ key, value })` | `key`, `value` (number, required) | `Promise<void>` |
| compareAndSet | `$db.number.compareAndSet({ key, expected, value })` | `key`, `expected` (string, number or boolean), `value` (number) | `Promise<void>` |

`increment` / `decrement` need the key to exist as a number. Create it with `set` or `setIfAbsent` first (see [Create before you modify](#common-rules)).

Both resolve with the number stored after the write, read back from the same atomic update. Use it instead of a follow-up `get`, which another caller's write can race.

**Example:**

```javascript
await $db.number.setIfAbsent({ key: 'counter', value: 0 }) // or set, to reset it
  .catch((e) => { if (e.code !== 'CONDITION_FAILED') throw e; }); // already exists: fine
const res = await $db.number.get({ key: 'counter' });
const next = await $db.number.increment({ key: 'counter' }); // the new value (1 on a fresh key)
await $db.number.decrement({ key: 'counter' });                // resolves with the new value too
await $db.number.delete({ key: 'counter' });
```

---

## $db.list

| Method            | Signature | Parameters | Return |
|-------------------|-----------|------------|--------|
| get               | `$db.list.get({ key })` | `key` (string, required) | `Promise<array>` |
| set               | `$db.list.set({ key, value })` | `key`, `value` (array, required) | `Promise<void>` |
| append            | `$db.list.append({ key, value })` | `key`, `value` (array, required) | `Promise<void>` |
| prepend           | `$db.list.prepend({ key, value })` | `key`, `value` (array, required) | `Promise<void>` |
| updateItemAtIndex | `$db.list.updateItemAtIndex({ key, value, index })` | `key`, `value` (array), `index` (number, required) | `Promise<void>` |
| removeItemAtIndex | `$db.list.removeItemAtIndex({ key, index })` | `key`, `index` (number, required) | `Promise<void>` |
| setIfAbsent       | `$db.list.setIfAbsent({ key, value })` | `key`, `value` (array, required) | `Promise<void>` |

`append`, `prepend`, `updateItemAtIndex` and `removeItemAtIndex` need the key to exist as a list. They don't create it. Create it with `set` first, as in the example below (see [Create before you modify](#common-rules)).

**Example:**

```javascript
await $db.list.set({ key: 'logs', value: [] });   // create the list before appending
await $db.list.append({ key: 'logs', value: ['entry-1'] });
const res = await $db.list.get({ key: 'logs' });
await $db.list.removeItemAtIndex({ key: 'logs', index: 0 });
await $db.delete({ key: 'logs' });
```

---

## $db.map

Store and manipulate JSON objects. Beyond whole-object `get`/`set`/`delete`, the map API supports **atomic nested-path operations** that read or write a single value *inside* a stored object without rewriting the whole map.

### Nested paths

Nested-path methods address a target with a `path` — an array of segments walked from the root of the map.

| Rule          | Detail |
|---------------|--------|
| Non-empty     | `path` must be a non-empty array |
| Max depth     | At most 31 segments |
| Segment types | Each segment is a non-empty string (map key) or a non-negative integer (list index) |
| First segment | Must be a map key (string) |
| List index    | A numeric index is only allowed as the **final** segment |

Example path: `['profile', 'contacts', 0]` → the first element of the `contacts` list inside the `profile` object.

### Whole-object

| Method | Signature | Parameters | Return |
|--------|-----------|------------|--------|
| get    | `$db.map.get({ key })` | `key` (string, required) | `Promise<object>` |
| set    | `$db.map.set({ key, value })` | `key`, `value` (object, required, JSON-serializable) | `Promise<void>` |
| delete | `$db.map.delete({ key })` | `key` (string, required) | `Promise<void>` |

### Nested-path operations

Each operates atomically on a single leaf addressed by `path`.

| Method     | Signature | Parameters | Return |
|------------|-----------|------------|--------|
| updateItem | `$db.map.updateItem({ key, path, value })` | `path` (array, required), `value` (any JSON type, required, non-null) | `Promise<void>` |
| removeItem | `$db.map.removeItem({ key, path, recursive })` | `path` (array, required), `recursive` (boolean, optional, default `false`) | `Promise<void>` |
| increment  | `$db.map.increment({ key, path, value })` | `path` (array, required), `value` (number, required) | `Promise<number>` — the leaf's value after the increment |
| decrement  | `$db.map.decrement({ key, path, value })` | `path` (array, required), `value` (number, required) | `Promise<number>` — the leaf's value after the decrement |
| append     | `$db.map.append({ key, path, value })` | `path` (array, required), `value` (array, required, non-empty) | `Promise<void>` |
| prepend    | `$db.map.prepend({ key, path, value })` | `path` (array, required), `value` (array, required, non-empty) | `Promise<void>` |
| typeOf     | `$db.map.typeOf({ key, path })` | `path` (array, required) | `Promise<string>` |
| byteSize   | `$db.map.byteSize({ key, path })` | `path` (array, required) | `Promise<number>` |
| setIfAbsent | `$db.map.setIfAbsent({ key, path, value })` | `path` (array, optional — omit to target the whole key), `value` (required, non-null; an object when there is no `path`) | `Promise<void>` |
| compareAndSet | `$db.map.compareAndSet({ key, path, expected, value })` | `path` (array, required), `expected` (string, number or boolean), `value` (required, non-null) | `Promise<void>` |

- `updateItem` requires a non-null `value` — use `removeItem` to delete.
- `removeItem` by default removes only a leaf (a scalar, or an already-absent key); a path landing on a nested object or list is rejected. Pass `recursive: true` to remove such a node together with its whole subtree.
- `increment`/`decrement` require the leaf to be a number and resolve with the leaf's new value, read back from the same atomic update; `append`/`prepend` require it to be a list and `value` to be a non-empty array.
- `setIfAbsent` / `compareAndSet` follow the [conditional write](#conditional-writes--setifabsent-and-compareandset) rules.
- `typeOf` returns one of `'map' | 'list' | 'number' | 'string' | 'boolean' | 'null' | 'absent'`; `byteSize` returns serialized bytes (missing path or null value is `0`). Both are point-in-time reads — a convenience to avoid type-mismatch errors, not a substitute for the atomic write-time guards.

### Batch — apply

| Method | Signature | Parameters | Return |
|--------|-----------|------------|--------|
| apply  | `$db.map.apply({ key, ops })` | `ops` (array, required, non-empty) | `Promise<void>` |

`apply` runs many single-leaf ops on one key in a single atomic, all-or-nothing call — the whole batch succeeds together or none of it does. Each op is `{ op, path, value?, recursive?, expected? }`, where `op` is one of `updateItem | removeItem | increment | decrement | append | prepend | setIfAbsent | compareAndSet` and `path`/`value`/`expected` follow the same rules as the standalone methods above.

Inside `apply`, a `setIfAbsent` whose leaf is already set, or a `compareAndSet` whose leaf doesn't equal `expected`, rejects the whole batch and nothing is written. Use it to make a batch depend on a known state.

A failed condition rejects with an error whose `code` is `'CONDITION_FAILED'` (HTTP **412**), so you can tell a lost race (re-read and retry) from a bad request (a **400**, don't retry):

| Field | Value |
|-------|-------|
| `code` | `'CONDITION_FAILED'` |
| `message` | `Path '<path>' already holds a value` / `Path '<path>' does not hold the expected value` |
| `key` | the key `apply` was called on |
| `op` | index in `ops` of the op whose condition failed |
| `path` | that op's `path` |
| `current` | what is stored at that path now (left out when nothing is stored there) |

`op` names the first op that fails when the server re-reads the item after the refused write. If another caller changed the item again in between, the re-read can find every condition satisfied; the error then carries only `code`, `message` and `key`. Either way, the right response is to re-read and retry:

```js
for (let attempt = 0; attempt < 5; attempt++) {
  const order = await $db.map.get({ key: 'order' });
  try {
    await $db.map.apply({
      key: 'order',
      ops: [
        { op: 'compareAndSet', path: ['version'], expected: order.version, value: order.version + 1 },
        { op: 'updateItem', path: ['status'], value: 'shipped' },
      ],
    });
    break; // applied
  } catch (e) {
    if (e.code !== 'CONDITION_FAILED') throw e; // a real error: don't retry
  }
}
```

| Rule | Detail |
|------|--------|
| Max ops | At most **25** ops per call |
| No overlapping paths | Two ops in one batch cannot touch the same path, or a path and its parent/child (e.g. `['address']` and `['address', 'city']`). Different indexes of the same list are fine. |
| Size | A batch of many deep paths can be rejected as too large even under 25 ops — split it into smaller batches. |

If any rule is broken the whole batch is rejected and nothing is written.

**Example — whole-object:**

```javascript
await $db.map.set({ key: 'profile', value: { name: 'Ava', visits: 0, tags: ['new'] } });
const res = await $db.map.get({ key: 'profile' });
await $db.map.delete({ key: 'profile' });
```

**Example — nested-path operations:**

```javascript
await $db.map.set({ key: 'profile', value: { name: 'Ava', visits: 0, tags: ['new'] } });

// add / replace a nested field
await $db.map.updateItem({ key: 'profile', path: ['address', 'city'], value: 'Chennai' });

// atomic numeric update on a nested leaf; resolves with the new value
const visits = await $db.map.increment({ key: 'profile', path: ['visits'], value: 1 }); // 1

// append to a nested list leaf
await $db.map.append({ key: 'profile', path: ['tags'], value: ['verified'] });

// inspect before acting
const kind = await $db.map.typeOf({ key: 'profile', path: ['visits'] }); // 'number'

// remove a leaf, or a whole subtree with recursive
await $db.map.removeItem({ key: 'profile', path: ['tags', 0] });
await $db.map.removeItem({ key: 'profile', path: ['address'], recursive: true });
```

**Example — apply (atomic batch):**

```javascript
await $db.map.apply({
  key: 'profile',
  ops: [
    { op: 'increment', path: ['visits'], value: 1 },
    { op: 'updateItem', path: ['lastSeen'], value: '2026-10-06' },
    { op: 'append', path: ['tags'], value: ['returning'] },
  ],
}); // all three succeed together, or none do
```

---

## $db.boolean

| Method | Signature | Parameters | Return |
|--------|-----------|------------|--------|
| get    | `$db.boolean.get({ key })` | `key` (string, required) | `Promise<boolean>` |
| set    | `$db.boolean.set({ key, value })` | `key`, `value` (boolean, required) | `Promise<void>` |
| delete | `$db.boolean.delete({ key })` | `key` (string, required) | `Promise<void>` |
| setIfAbsent | `$db.boolean.setIfAbsent({ key, value })` | `key`, `value` (boolean, required) | `Promise<void>` |
| compareAndSet | `$db.boolean.compareAndSet({ key, expected, value })` | `key`, `expected` (string, number or boolean), `value` (boolean) | `Promise<void>` |

**Example:**

```javascript
await $db.boolean.set({ key: 'isActive', value: true });
const res = await $db.boolean.get({ key: 'isActive' });
await $db.boolean.set({ key: 'isActive', value: false });
await $db.boolean.delete({ key: 'isActive' });
```
