---
name: sleeksky
description: Build, check and publish apps for the SleekSky platform (Cloudflare Workers sold as single-tenant apps). Use when the user wants to start a SleekSky app, write or fix its sleeksky.json manifest, add platform sign-in, schedules, secrets or migrations, make an existing Worker run on SleekSky, or publish a release with the sleeksky CLI.
---

# Building SleekSky apps

SleekSky sells finished software (products) as **apps**: every customer gets their own copy of your Worker, with its
own D1 database, buckets and address (`<name>.sleeksky.app`). You, the developer, publish a **package**: your code plus
`sleeksky.json`, the manifest the platform reviews. The platform provisions, updates, bills and signs people in.

Read [reference/manifest.md](reference/manifest.md) before writing a manifest and
[reference/runtime.md](reference/runtime.md) before writing platform code. The live docs, which win if they differ,
are at `<console>/docs` (default console `https://apps.sleeksky.com`; `SLEEKSKY_CONSOLE` overrides it).

## Start a new app

```bash
npx @sleeksky/cli create <folder> [--name "My App"] [--id my-app]
cd <folder> && npm install && npm run dev
```

The template is a working app: `/_health`, an hourly schedule, platform sign-in, sessions, notes, and copy
detection. Change `id`, `name` and `products` in `sleeksky.json` first: `id` is permanent once published.

## Make an existing Worker a SleekSky app

1. Add `sleeksky.json` at the repo root (smallest valid one is in the manifest reference).
2. Remove `routes`, `workers_dev` and other service bindings from the Wrangler config. Turn crons into `schedules`
   with `/_sky/...` paths; turn hard-coded API keys into manifest `secrets` or platform `services`.
3. Keep it running on its own with the same code: `npm install @sleeksky/sky`, `export { Sky } from
   "@sleeksky/sky/standalone"`, wrap the default export in `withSky(app, { schedules })`, and in `wrangler.jsonc` bind
   `SKY` to the Worker's own `Sky` entrypoint, plus a `*/15 * * * *` cron for the schedules (reference/runtime.md).
   No `if (APP_ID && SKY)` branches: what really differs comes from init's `managed`.
4. Serve `GET /_health` (200 once the database answers) and each schedule path (`POST`, idempotent).
5. Move the schema into additive `migrations/*.sql`.
6. Replace self-hosted bootstrap (setup keys, "first user is admin") with platform sign-in when `APP_ID` is set.

## Rules that the review and the platform enforce

- Migrations **only add**: new tables, nullable or defaulted columns, indexes. No `DROP`, `RENAME`, `DELETE`,
  `REPLACE` or `PRAGMA`. Old code runs on the new schema during updates and after rollbacks.
- Talk to the platform only through `env.SKY.fetch("https://sky/...")`. No tokens or keys exist or are needed.
- Call `GET https://sky/api/app/init` and cache it for its `ttl`. Store `app.id`; if it changes, the data was
  copied from another app: clear the original's sessions, owners and outgoing jobs, then store the new id.
- Owners come from sign-in: a sign-in with `owner: true` makes that person an owner (one your own admins can't
  remove); a later sign-in with `owner: false` takes it away. Log `staff: true` sign-ins where owners can see them.
- When init says `"storage": "over"`, refuse writes that grow storage and point owners to the console.
- Never check for updates, show prices for the app itself, bill customers yourself, store customers' secrets in
  plain text, or fetch hosts outside the manifest's `egress`.
- `/_health` and `/_sky/*` are reserved. The platform calls nothing else on your app.

## Check before publishing

1. `npm run dev` works with no platform (`APP_ID` unset, `SKY` the stand-in), sign-in included.
2. The manifest is valid: the generator at `<console>/docs/manifest-generator` checks it with the platform's rules,
   and `npx @sleeksky/cli publish` refuses an invalid one before anything goes live.
3. Every migration is additive; the previous release still works on the new schema.
4. Every platform call goes over `env.SKY`; sign-in builds `` `${env.CONSOLE_URL}/auth` ``.

## Publish

```bash
npx @sleeksky/cli publish --notes "First change|Second change"
```

- `sleeksky.json`'s `id` is the app's id on SleekSky: the platform creates the app with it and adds the user as one
  of its people (admin or developer). Never change `id`. A publish for an app the platform hasn't created is refused.
- The first publish signs the user in: it prints a link and a code and opens their browser, where they sign in and
  approve; it waits until they do. Tell the user to approve it themselves. The sign-in lasts 30 days in
  `~/.sleeksky/credentials.json`: never read, print or copy that file. Don't run publish without the user asking.
- It publishes `"version"` from `sleeksky.json` (`YYYY.M.N` or `MAJOR.MINOR.PATCH`): raise it before each release,
  higher than the last one (the console's Releases tab, or ask). There's no `--version` flag.
- Flags: `--data-note "Adds 2 tables"` (describes migrations), `--security` (forced on manual updaters after 14
  days), `--allow-destructive "<why>"` only when the user confirms a non-additive migration is safe.
- From a developer, the first release and any release whose `sleeksky.json` changes (anything but `release`) is
  **held for review**; from an admin of the app it's approved as it's published. Code-only releases roll out to customers by their update channel. A failing `/_health` rolls back.
- Supplied secrets (manifest `secrets` of kind `supplied`), one value for every app:
  `printf %s "$VALUE" | npx @sleeksky/cli secret set --name NAME`, `npx @sleeksky/cli secret list`,
  `npx @sleeksky/cli secret clear --name NAME` (the app's admins only). The value comes on standard input, never as
  an argument.
