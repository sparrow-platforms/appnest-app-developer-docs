# $db — Key/Value Database SDK

The `$db` module provides typed key/value storage operations. Use the namespaced APIs (`$db.string`, `$db.number`, `$db.list`, `$db.map`, `$db.boolean`) for full create/update support.

**Import:** `const { $db } = AppnestFunctions;` — see [Import and rules](README.md#import).

---

## Common Rules

| Parameter | Type   | Required | Constraints                          |
|----------|--------|----------|--------------------------------------|
| `key`    | string | yes      | Non-empty, max 1000 characters       |

**Return shape:** Read methods (`get`, `typeOf`, `byteSize`) return the value directly (string, number, array, object, or boolean). Create, update, and delete methods return nothing (`Promise<void>`).

**Size limit:** a stored value can be at most **380KB**. Writes over the limit throw an error.

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
| increment | `$db.number.increment({ key, value })` | `key` (string, required), `value` (number, optional) | `Promise<void>` |
| decrement | `$db.number.decrement({ key, value })` | `key` (string, required), `value` (number, optional) | `Promise<void>` |

**Example:**

```javascript
await $db.number.set({ key: 'counter', value: 0 });
const res = await $db.number.get({ key: 'counter' });
await $db.number.increment({ key: 'counter' });
await $db.number.decrement({ key: 'counter' });
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

**Example:**

```javascript
await $db.list.set({ key: 'logs', value: [] });
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
| increment  | `$db.map.increment({ key, path, value })` | `path` (array, required), `value` (number, required) | `Promise<void>` |
| decrement  | `$db.map.decrement({ key, path, value })` | `path` (array, required), `value` (number, required) | `Promise<void>` |
| append     | `$db.map.append({ key, path, value })` | `path` (array, required), `value` (array, required, non-empty) | `Promise<void>` |
| prepend    | `$db.map.prepend({ key, path, value })` | `path` (array, required), `value` (array, required, non-empty) | `Promise<void>` |
| typeOf     | `$db.map.typeOf({ key, path })` | `path` (array, required) | `Promise<string>` |
| byteSize   | `$db.map.byteSize({ key, path })` | `path` (array, required) | `Promise<number>` |

- `updateItem` requires a non-null `value` — use `removeItem` to delete.
- `removeItem` by default removes only a leaf (a scalar, or an already-absent key); a path landing on a nested object or list is rejected. Pass `recursive: true` to remove such a node together with its whole subtree.
- `increment`/`decrement` require the leaf to be a number; `append`/`prepend` require it to be a list and `value` to be a non-empty array.
- `typeOf` returns one of `'map' | 'list' | 'number' | 'string' | 'boolean' | 'null' | 'absent'`; `byteSize` returns serialized bytes (missing path or null value is `0`). Both are point-in-time reads — a convenience to avoid type-mismatch errors, not a substitute for the atomic write-time guards.

### Batch — apply

| Method | Signature | Parameters | Return |
|--------|-----------|------------|--------|
| apply  | `$db.map.apply({ key, ops })` | `ops` (array, required, non-empty) | `Promise<void>` |

`apply` runs many single-leaf ops on one key in a single atomic, all-or-nothing call — the whole batch succeeds together or none of it does. Each op is `{ op, path, value?, recursive? }`, where `op` is one of `updateItem | removeItem | increment | decrement | append | prepend` and `path`/`value` follow the same rules as the standalone methods above.

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

// atomic numeric update on a nested leaf
await $db.map.increment({ key: 'profile', path: ['visits'], value: 1 });

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

**Example:**

```javascript
await $db.boolean.set({ key: 'isActive', value: true });
const res = await $db.boolean.get({ key: 'isActive' });
await $db.boolean.set({ key: 'isActive', value: false });
await $db.boolean.delete({ key: 'isActive' });
```
