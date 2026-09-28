# $db — Key/Value Database SDK

The `$db` module provides typed key/value storage operations. Use the namespaced APIs (`$db.string`, `$db.number`, `$db.list`, `$db.map`, `$db.boolean`) for full create/update support.

---

## Import / Usage

```javascript
const { AppnestFunctions } = require('@sparrowengg/appnest-app-sdk-utils');
const { $db } = AppnestFunctions;
```

---

## Common Rules

| Parameter | Type   | Required | Constraints                          |
|----------|--------|----------|--------------------------------------|
| `key`    | string | yes      | Non-empty, max 1000 characters       |

**Return shape:** Get methods return the value directly (string, number, array, object, or boolean). Create, update, and delete methods return nothing (`Promise<void>`).

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
| set    | `$db.set({ key, value })` | Write string value at key (same op as `$db.string.set` / replace). | `Promise<void>` |
| delete | `$db.delete({ key })` | Delete by key | `Promise<void>` |

**Example:**

```javascript
await $db.set({ key: 'welcome', value: 'hello' });
const result = await $db.get({ key: 'welcome' }); // result is the string value
await $db.set({ key: 'welcome', value: 'hello world' });
await $db.delete({ key: 'welcome' });
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

| Method | Signature | Parameters | Return |
|--------|-----------|------------|--------|
| get    | `$db.map.get({ key })` | `key` (string, required) | `Promise<object>` |
| set    | `$db.map.set({ key, value })` | `key`, `value` (object, required, JSON-serializable) | `Promise<void>` |
| delete | `$db.map.delete({ key })` | `key` (string, required) | `Promise<void>` |

**Example:**

```javascript
await $db.map.set({ key: 'profile', value: { name: 'Ava', age: 28 } });
const res = await $db.map.get({ key: 'profile' });
await $db.map.set({ key: 'profile', value: { name: 'Ava', age: 29 } });
await $db.map.delete({ key: 'profile' });
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
