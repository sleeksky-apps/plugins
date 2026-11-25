# Runtime and the platform API

Live reference: `<console>/docs/runtime`, `<console>/docs/sky-api`, `<console>/docs/sign-in`.

## What the app receives on the platform

| Binding | Meaning |
|---|---|
| `APP_ID` | This app's id. With `SKY`, means "running on the platform". |
| `CONSOLE_URL` | The console, where people sign in (`/auth`). |
| `SKY` | Service binding to the platform. No key. |
| `PLATFORM_DOMAIN`, `APP_SUBDOMAIN` | The app's address is `<subdomain>.<platform domain>`. |
| `APP_PRODUCT` | The SKU it was set up from. |
| params, resources, secrets, vars | As the approved manifest declares. |

Requests arrive through the platform; the app routes its own hostnames (`<sub>.<domain>`, `<sub>--<label>.<domain>`
and verified custom domains). Visitor location is in the `x-sleeksite-geo` header (URI-encoded JSON); `request.cf`
doesn't reach apps. CPU and subrequests per request depend on the customer's plan.

## Platform API (all over `env.SKY.fetch("https://sky/<path>")`)

| Endpoint | Use |
|---|---|
| `GET /api/app/init` | `{ app: {id, name, subdomain, product, account}, services, allowance, storage: "ok"\|"warn"\|"over", ttl }`. Cache for `ttl` seconds. |
| `POST /auth/token` | `{code, redirect}` → `{email, name, owner, staff}`. Within 60 s, once. |
| `POST /api/app/mail/signin-code` | Send a sign-in code email (rate-limited), for apps with their own sign-in. |
| `POST /api/app/signins/remove` | `{email}`: you removed this person; drops the app from their **Your apps**. |
| `POST /api/app/analytics/sql` | `{sql}`: one SELECT over your Analytics Engine bindings. |
| `POST /api/app/cache/invalidate` | Clear the app's edge cache (with `edge_cache`). |
| `/api/app/domains` | `GET`/`POST`; `POST …/<host>/check`; `DELETE …/<host>`: custom domains. |

## Sign in with the platform

```js
// Start (signin_path): send them to the console.
const state = crypto.randomUUID();                      // also set as a short-lived HttpOnly cookie
const to = new URL("/auth", env.CONSOLE_URL);
to.searchParams.set("app", env.APP_ID);
to.searchParams.set("redirect", `${url.origin}/auth/callback`); // https, on one of the app's own addresses
to.searchParams.set("state", state);

// Callback: check state against the cookie, then exchange the code over SKY.
const r = await env.SKY.fetch("https://sky/auth/token", {
  method: "POST", headers: { "content-type": "application/json" },
  body: JSON.stringify({ code, redirect: `${url.origin}/auth/callback` }),
});
const { email, name, owner, staff } = await r.json();    // then start the app's own session
```

Optional `prompt=login` (always show the form) and `login_hint` (prefill email) on `/auth`.

## Endpoints the app serves

| Endpoint | What |
|---|---|
| `GET /_health` | `200 {"ok":true}` once the database answers. Checked after every deploy; failure rolls back. |
| each `schedules[].path` | `POST` on schedule, aligned to the clock. Idempotent. Only the platform can reach `/_sky/*`, so no token check. |

For work more often than every 15 minutes, use a Durable Object alarm and re-arm it from a schedule.
