# sleeksky.json

The manifest belongs to the platform: you propose it, a reviewer approves it, and only the approved copy is used to
set up or deploy apps. Live reference: `<console>/docs/manifest`.

## Smallest valid manifest

```json
{
  "id": "acme-notes",
  "name": "Acme Notes",
  "resources": { "d1": { "binding": "DB" } },
  "products": [
    { "id": "acme-notes", "name": "Acme Notes",
      "variants": [{ "id": "acme-notes-solo", "name": "Solo", "price": 5 }] }
  ],
  "release": { "config": "wrangler.jsonc" }
}
```

## Fields

| Field | Meaning |
|---|---|
| `id` | Package id: lowercase letters, digits, dashes; 2–31 chars; starts with a letter. Permanent. |
| `name` | The app's name. |
| `home` | Path **Open app** lands on, e.g. `/app`. |
| `icon` | Path of the app's icon (PNG/JPEG/WebP/GIF/ICO/SVG, ≤200 KB). |
| `platform_signin` | `true`: people sign in with their platform account. |
| `signin_path` | With `platform_signin`: the path that starts sign-in (the console's **Your apps** opens it). |
| `resources.d1` | `{ "binding": "DB" }`. Required. |
| `resources.r2` / `kv` / `analytics_engine` | Lists of `{ "binding" }`, one per app. KV is a cache (not cloned). |
| `resources.durable_objects` | List of `{ "binding", "class" }` (a class your Worker exports). |
| `assets` | `{ "binding": "ASSETS", "not_found_handling": "single-page-application" \| "404-page" \| "none", "run_worker_first": [...] \| true }`; set `release.assets_dir`. |
| `platform_bindings` | List of `{ "type", "binding" }`, type `images`, `browser`, `send_email` or `ai`. |
| `secrets` | List of `{ "name", "kind" }`: `random` (per app, kept), `vapid` (Web Push pair: `<NAME>_PUBLIC_KEY`, `<NAME>_PRIVATE_KEY`), `supplied` (one value you set for every app; add `"override": true` to let account owners use their own). |
| `vars` | Plain strings, the same for every app. |
| `params` | Per-variant values: `{ "NAME": { "type": "int" \| "text" \| "select", "options": [...], "default": ... } }`. |
| `services` | Paid platform services: `openrouter`, `dataforseo`. Called at their normal URLs with no key; the platform meters them. |
| `egress` | `"any"` or a list of hosts (`api.example.com`, `*.example.com`) the app may fetch. |
| `edge_cache` | `true` to let the platform cache responses that send `Cache-Control: public, s-maxage=N`. |
| `payments` | `"none"` or `"connect"` (customers connect their own payment account). |
| `schedules` | List of `{ "every", "path" }`: `every` in `m`/`h`/`d`, a multiple of 15m; `path` under `/_platform/`. |
| `clone` | Present (`{}`) lets owners clone an app: `skip_tables`, `copy_secrets`, `r2`. |
| `install` | Who can add an app: `"public"`, `"developers"`, or `{ "domains": [...], "emails": [...] }`. |
| `products` | How it's sold; see below. |
| `release` | Build details for the CLI, not reviewed: `build` (shell command), `config` (Wrangler config), `assets_dir`. |

Binding names are `CAPITALS_AND_UNDERSCORES`, unique across the manifest. Reserved: `APP_ID`, `CONSOLE_URL`,
`PLATFORM_DOMAIN`, `APP_SUBDOMAIN`, `APP_PRODUCT`, `SKY`, `GATEWAY`. Removed and refused: `alarm`, `clone.hook`.

## Products

| Product field | Meaning |
|---|---|
| `id`, `name` (≤40), `description` (≤400) | |
| `icon` | `www`, `world`, `bag`, `store`, `rocket`, `sparkles`. |
| `color` | `ember`, `cobalt`, `lagoon`, `grape`, `teal`, `gray`. |
| `replaces` | ≤8 names, shown as "Alternative to". |
| `features` | ≤3 lines of ≤60 chars. |
| `links` | `website`, `demo`, `terms`, `privacy` (https). |
| `variants` | ≥1: `{ id (SKU, unique platform-wide, permanent), name, price (USD/month), params, description, icon, features }`. |

## The Wrangler config

No `routes`, `triggers`, `workers_dev`, or service bindings to other Workers: the platform decides how requests
reach the app. Keep the bindings you need for local development; on the platform they come from the manifest.
