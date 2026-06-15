# ci-agent one-click apps

A catalog of ready-to-deploy app templates for
[ci-agent](https://github.com/m-hoseinzadeh/ci-agent). Add this repository as a
**catalog source** in ci-agent (Projects → Browse app catalog → Manage sources)
and its apps appear in the searchable catalog; pick one, fill a short form, and
ci-agent creates an inline-compose project from the template.

Currently **349 apps**, derived from the
[CapRover One-Click Apps](https://github.com/caprover/one-click-apps) project
(Apache-2.0 — see `LICENSE` and `NOTICE`).

## Layout

```
apps/<slug>.toml   # one template per app
```

ci-agent clones this repo on **Sync** and reads every `apps/*.toml`.

## Offline / air-gapped servers

A box with no git access can't Sync. For those, every change here is packaged
into a `catalog.zip` published as the rolling
[**`catalog-latest`** release](https://github.com/m-hoseinzadeh/ci-agent-one-click-apps/releases/tag/catalog-latest)
(see `.github/workflows/catalog-release.yml`). Stable URL:

```
https://github.com/m-hoseinzadeh/ci-agent-one-click-apps/releases/download/catalog-latest/catalog.zip
```

Download it (and `catalog.zip.sha256` to verify), carry it to the server, then
in ci-agent: **Manage sources → Add an uploaded-bundle source → Upload bundle**.
The zip holds `apps/` at its root — the same files Sync would clone.

## Template format

```toml
slug = "redis"                 # unique id (matches the file name)
name = "Redis"                 # display name
category = "Database"          # shown as a chip; optional
description = "In-memory key-value store, cache and message broker."
keywords = ["cache", "kv"]     # extra search terms; optional
http_port = 6379               # optional default host port (else ci-agent suggests one)

# IMPORTANT: every top-level key (including `compose`) must come BEFORE the
# [[variables]] blocks — TOML absorbs anything after [[variables]] into that
# table.
compose = '''
services:
  redis:
    image: redis:{{REDIS_VERSION}}
    restart: always
    command: ["sh", "-c", "redis-server --requirepass $$REDIS_PASSWORD"]
    environment:
      REDIS_PASSWORD: "{{REDIS_PASSWORD}}"
    ports: ["{{HTTP_PORT}}:6379"]
    volumes: ["redis-data:/data"]
volumes:
  redis-data:
'''

[[variables]]
key = "REDIS_PASSWORD"
label = "Redis password"
generate = true                # pre-fill the form field with a generated secret

[[variables]]
key = "REDIS_VERSION"
label = "Redis version tag"
default = "7.2.4"
```

Placeholders:

- `{{HTTP_PORT}}` — replaced with the host port chosen in the configure form.
- `{{ANYTHING_ELSE}}` — must be declared in a `[[variables]]` block (the
  configure form asks for it). `generate = true` pre-fills a random secret;
  `default = "…"` pre-fills a fixed value.
- Use `$$` for a literal `$` in compose values (docker compose interpolates a
  single `$`).

## Conversion caveats

These templates were mechanically converted from CapRover's format. CapRover
proxies apps internally; ci-agent publishes a host port on plain docker compose.
The conversion:

- strips CapRover's `srv-captain--` internal DNS prefix (so inter-service
  references resolve to plain compose service names),
- publishes the app's web port — or a recognized database/cache port — as
  `{{HTTP_PORT}}`,
- maps `$$cap_*` variables to `{{...}}` placeholders (`$$cap_gen_random_*`
  defaults become `generate = true`),
- escapes shell `$` to `$$`.

**Review and test an app before relying on it in production** — a few apps
expect CapRover-specific behavior (multiple proxied subdomains, specific port
exposure) that needs manual adjustment.

## Contributing

Add or edit `apps/<slug>.toml`, keeping the format above (top-level keys before
`[[variables]]`). One app per file.
