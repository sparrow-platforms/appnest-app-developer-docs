# $fetch — Outbound HTTP Request SDK

The `$fetch` module lets you make outbound HTTP requests (GET, POST, PUT, DELETE, PATCH) with URL, headers, body, and optional **options** (`isOauth`, `maxAttempts`, `additionalOptions`).

---

## Import / Usage

```javascript
const { AppnestFunctions } = require('@sparrowengg/appnest-app-sdk-utils');
const { $fetch } = AppnestFunctions;
```

---

## API

### request

| Method | Signature | Description |
|--------|-----------|-------------|
| request | `$fetch.request({ url, method, headers, body, options })` | Execute an HTTP request. |

**Parameters:**

| Parameter | Type   | Required | Default | Constraints |
|-----------|--------|----------|---------|-------------|
| url       | string | yes      | —       | Must start with `https://` or `http://`, max 2048 chars |
| method    | string | yes      | —       | One of: `GET`, `POST`, `PUT`, `DELETE`, `PATCH` |
| headers   | object | no       | `{}`    | String key/value pairs; max 1000 keys; each value string, max 1000 chars |
| body      | object | no       | `{}`    | String key/value pairs; max 1000 keys; each value string, max 1000 chars |
| options   | object | no       | `{}`    | See **options** below |

**options** (merged into the platform request payload):

| Field              | Type    | Default | Description |
|--------------------|---------|---------|-------------|
| `isOauth`          | boolean | `false` | Whether to use OAuth for the outbound request. |
| `maxAttempts`      | number  | `0`     | Maximum retry attempts (platform-defined behavior). |
| `additionalOptions`| object  | `{}`    | Passthrough bag for additional/future options. Its keys are spread as **top-level fields** into the outbound options payload and forwarded as-is to the platform — allowing new options to be sent without an SDK change. Cannot override the SDK's own fields (`isOAuth`, `maxAttempts`, `headers`, `userOauthId`). |

The SDK builds the outbound `options` object as: any keys from `additionalOptions` (spread first), then `isOauth`, `maxAttempts`, your **headers**, and `userOauthId` (empty string unless set by the platform).

**Return:** `Promise<{ headers: Object, body: any, status: number }>` — response headers, response body, and HTTP status code.

---

## Installation params (iparams) for API calls

**Rule — required for all API calls:** Any value that comes from app configuration (installation params / iparams) and is used in a `$fetch` request **must** be expressed using the replace syntax `<%=iparams.<param_key>%>` in **url**, **headers**, or **body**. Do not hardcode these values or read them at runtime; the AppNest framework replaces the placeholders at request time.

When making an API call through `$fetch`, if the developer needs to use an iparam-specific key/value from their manifest, they must define it under **installation_parameters** in `manifest.json`, inside `product_config.<product>` (see `appnest-governance/app-configuration/Manifest-Rules.md`).

### Defining iparams in the manifest

Under `product_config.<product>` (e.g. `product_config.surveysparrow`):

```json
"installation_parameters": {
  "surveysparrow_api_key": {
    "display_name": "SurveySparrow API Key",
    "description": "Please enter your SurveySparrow API key. You can find it in Settings → Apps & Integrations → Custom Apps.",
    "required": true,
    "secure": true,
    "type": "api_key"
  },
  "surveysparrow_api_url": {
    "display_name": "SurveySparrow API URL",
    "description": "Please enter your SurveySparrow API URL.",
    "required": true,
    "secure": true,
    "type": "url"
  }
}
```

### Using iparams in the request

Reference iparams in the request **url**, **headers**, or **body** using the replace syntax:

- **In the URL:**  
  `<%=iparams.surveysparrow_api_url%>/items`

- **In headers or body:**  
  `<%=iparams.surveysparrow_api_key%>`

The same replacement syntax (`<%=iparams.<param_key>%>`) can be used wherever dynamic values from iparams are required in the request configuration. `<%=installation_parameters.<param_key>%>` is an equivalent alias.

### For AI / code generation

When generating code that uses `$fetch.request`:

- **options** is optional: `{ isOauth?: boolean, maxAttempts?: number, additionalOptions?: object }` (defaults: `false`, `0`, `{}`). Include `isOauth`/`maxAttempts` when OAuth or retries are required per product docs. Use `additionalOptions` only to forward newer platform-supported option fields that don't yet have a dedicated parameter.
- **Always** use `<%=iparams.<param_key>%>` for any value that comes from installation params (API keys, base URLs, tenant IDs, etc.) in `url`, `headers`, or `body`. Never substitute a variable or literal for an iparam value in the request config.
- **Never** hardcode API keys, base URLs, or other config that is (or could be) defined in the app manifest; use the replace syntax so the framework can inject the correct value per installation.
- If the manifest defines an iparam (e.g. `api_key`, `api_url`), every `$fetch` usage that needs that value **must** reference it as `<%=iparams.<param_key>%>` in the request object.

### Using access_token in headers

To send the platform’s **access_token** in the request (e.g. for OAuth or API auth), use the replace syntax in **headers**. The AppNest framework will replace `<%=user_oauth.access_token%>` with the actual token at runtime.

**Example:**

```javascript
headers: {
  Authorization: "Bearer <%=user_oauth.access_token%>",
  "Content-Type": "application/json"
}
```

Use this in your `$fetch.request` call; the framework replaces `<%=user_oauth.access_token%>` before the request is sent.

---

## Examples

**GET:**

```javascript
const result = await $fetch.request({
  url: 'https://api.example.com/v1/items',
  method: 'GET',
  headers: { Authorization: 'Bearer <token>' },
  body: {}
});
```

**POST with body:**

```javascript
const result = await $fetch.request({
  url: 'https://api.example.com/v1/items',
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: { name: 'Item A', category: 'tools' }
});
```

**With `options` (`isOauth`, `maxAttempts`):**

```javascript
const result = await $fetch.request({
  url: 'https://api.example.com/v1/me',
  method: 'GET',
  headers: { Authorization: 'Bearer <%=user_oauth.access_token%>' },
  body: {},
  options: { isOauth: true, maxAttempts: 3 }
});
```

**PUT / DELETE / PATCH:** Use the same shape; set `method` to `PUT`, `DELETE`, or `PATCH` and provide `url`, `headers`, and `body` (and `options` if needed).

**Using iparams in the request:**

```javascript
// url, headers, or body can use <%=iparams.<key>%> (replaced at runtime)
const result = await $fetch.request({
  url: '<%=iparams.surveysparrow_api_url%>/v1/items',
  method: 'GET',
  headers: { Authorization: 'Bearer <%=iparams.surveysparrow_api_key%>' },
  body: {}
});
```

**Using access_token in headers (framework replacement):**

```javascript
const result = await $fetch.request({
  url: 'https://api.example.com/v1/me',
  method: 'GET',
  headers: {
    Authorization: "Bearer <%=user_oauth.access_token%>",
    "Content-Type": "application/json"
  },
  body: {}
});
```
