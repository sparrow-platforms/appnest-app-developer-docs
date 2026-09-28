# `window.AppnestFunctions` (developer reference)

**What it is:** A global object added when **`client.js`** runs in your marketplace app iframe. Use it from your frontend to call the **Sparrow Apps backend**, and (via **`$productParent`**) the **parent product** that embeds your app.

**Before you call it:** Load `client.js` first. Your app should run inside the product iframe. For **`$app.backend`**, the app URL should include **`appId`** and **`appVersionId`** query params.

---

## Exposed API

```js
window.AppnestFunctions = {
  $check,
  $app: { backend },
  $productParent: { getName, getRegion, invoke },
};
```

### `$check()`

Confirms the bridge loaded. Returns synchronously.

```js
const { data } = window.AppnestFunctions.$check();
// data.message — e.g. greeting from the client bridge
```

### `$app.backend({ functionName, functionPayload?, options? })`

Calls your server-side handler via the platform (`serverMethod` + payload). Returns a **`Promise`** that resolves to the API JSON body. Optional `options.timeout` (default ~120s).

```js
const result = await window.AppnestFunctions.$app.backend({
  functionName: 'myServerHandler',
  functionPayload: { id: 1 },
});
```

### `$productParent`

**`$productParent`** is used to call the parent product—the page that embeds your app in its iframe.

### `$productParent.getName`

| | |
|---|---|
| **Returns** | `Promise` — product parent identifier (e.g. `surveysparrow`) |

```js
const name = await window.AppnestFunctions.$productParent.getName();
```

### `$productParent.getRegion`

| | |
|---|---|
| **Returns** | `Promise` — region claim from the host |

```js
const region = await window.AppnestFunctions.$productParent.getRegion();
```

### `$productParent.invoke`

| | |
|---|---|
| **Signature** | `invoke({ actionName, actionPayload? })` |
| **Returns** | `Promise` — whatever the parent returns for that action |

Dispatches a product-specific action. Support depends on the **current product parent**. For **`surveysparrow`**, the parent typically supports actions such as:

| `actionName` | Description (typical) |
|--------------|------------------------|
| `getSurveyId` | Survey id inferred from parent URL; resolves to `{ surveyId }`. |
| `getLoggedInUserDetails` | Resolves to `{ loggedInUserDetails }` from `window.getLoggedInUserDetails()`. |
| `getLocation` | Resolves to `window.marketplaceLocalAppLocation`. |

Other product parents may reject unsupported actions.

```js
const { surveyId } = await window.AppnestFunctions.$productParent.invoke({
  actionName: 'getSurveyId',
});
```

---

## Object shape (JavaScript / JSDoc)

Paste near your app code if your editor uses JSDoc for hints. `window.AppnestFunctions` is still provided by **`client.js`** — this only documents the shape.

```js
/**
 * Shape of `window.AppnestFunctions` (injected by client.js).
 *
 * @typedef {object} AppnestFunctions
 * @property {() => { data: { message: string } }} $check
 * @property {{ backend: AppBackendInvoke }} $app
 * @property {{ getName: () => Promise<*>, getRegion: () => Promise<*>, invoke: AppProductParentInvoke }} $productParent
 */

/**
 * @callback AppBackendInvoke
 * @param {object} args
 * @param {string} args.functionName
 * @param {*} [args.functionPayload]
 * @param {{ timeout?: number }} [args.options]
 * @returns {Promise<*>}
 */

/**
 * @callback AppProductParentInvoke
 * @param {object} args
 * @param {string} args.actionName
 * @param {*} [args.actionPayload]
 * @returns {Promise<*>}
 */
```

On HTTP errors (e.g. 401 or status > 300) backend calls may show a toastr and reject; timeouts reject with an `Error`. Parent calls reject if the action fails or times out.
