# SleekSky plugins for Claude Code

```
/plugin marketplace add sleeksky-apps/plugins
/plugin install sleeksky@sleeksky
```

Then ask Claude to start, change or publish a SleekSky app ("make me a SleekSky app for booking rooms", "publish
version 1.2.0"). The `sleeksky` skill knows the manifest (`sleeksky.json`), the runtime rules, platform sign-in, and
the `sleeksky` CLI.

| Plugin | What it does |
|---|---|
| [`sleeksky`](plugins/sleeksky) | Start apps from the template, write and check manifests, add sign-in, schedules, secrets and migrations, publish releases. |

Docs: https://apps.sleeksky.com/docs
