# Authoring one-click app templates

Each app is one `apps/<slug>.toml` file. The catalog parser substitutes the
operator's answers into the `compose` block and creates an inline-compose
project from the result. These rules keep every template valid and deployable
on ci-agent.

## TOML shape

```toml
slug = "myapp"            # must equal the filename (no rename)
name = "My App"
category = "Productivity"
description = "One sentence."
keywords = ["tag1", "tag2"]
http_port = 8080          # OPTIONAL — see "Ports" below
compose = '''
services:
  ...
'''

[[variables]]
key = "..."
label = "..."
# default / description / generate are optional
```

**Ordering:** every top-level scalar key (`slug`, `name`, `category`,
`description`, `keywords`, `http_port`, `compose`) MUST appear *before* the
first `[[variables]]` table. Anything after `[[variables]]` is parsed as a
member of that table.

## Images

- **Every service must have an `image:`.** The catalog never builds — there is
  no `build:` context. A service without an image cannot deploy.
- Use official/upstream images. Pin a sensible tag. When the version should be
  operator-tunable, use a `{{SOMETHING_VERSION}}` variable with a real default
  (e.g. `image: postgres:{{POSTGRES_VERSION}}` + a variable defaulting to `16`).
- In multi-service stacks, worker/cron/sidecar services are usually the **same
  image as the main app** with a different `command:`/`entrypoint:`. Set them.

## Injected automatically — do NOT put these in templates

The runner generates an `override.yml` at deploy time. Do not add them:

- `networks:` — every service is attached to the project network *and* the
  shared `ci-agent-shared` network (with stable aliases) automatically.
- `container_name:` — set to `<slug>-<service>` at deploy time (the slug is only
  known then; hardcoding it would collide when an app is deployed twice).
- `restart:` for normal long-running services — injected as `unless-stopped`
  when a service does not declare one.

**Exception:** one-shot containers (init / migration / backup jobs that run once
and exit) MUST declare `restart: "no"` explicitly, or the injected default would
force them into a restart loop.

## Ports

- **Main web service:** publish `ports: - '{{HTTP_PORT}}:<containerPort>'` and
  set top-level `http_port = <containerPort>` so the configure form pre-fills it.
- **Internal sidecars** (databases, redis, queues, workers in a multi-service
  app): publish **no** ports — they reach each other by service name on the
  project network.
- **Standalone database apps** (the DB is the primary service, e.g. `postgres`,
  `mysql`, `mariadb`, `redis`, `mongodb`): keep
  `ports: - '{{HTTP_PORT}}:<dbPort>'` but do **not** set top-level `http_port`.
  The port field then defaults blank → the DB is internal-only by default, and
  the operator can type a port to expose it. A blank port renders the template
  with the `{{HTTP_PORT}}` mapping (and an emptied `ports:` key) removed.

## Variables and secrets

- Every `{{VAR}}` used in `compose` must be declared in a `[[variables]]` table.
- `generate = true` → the form pre-fills a fresh random secret, and if the
  operator clears the field a new one is generated on submit. Use it ONLY for
  **internal secrets the app mints for itself**: database passwords,
  `SECRET_KEY_BASE`, `APP_KEY`, encryption keys, admin passwords.
- **External credentials the operator must obtain elsewhere** — third-party API
  keys, OAuth client IDs/secrets, SMTP passwords, push tokens, S3/AWS keys,
  `PLEX_TOKEN`, etc. — must NOT use `generate`. Leave `default` empty (the
  operator fills it, or leaves it blank when the integration is unused). Say so
  in `description`.
- Give non-secret config a real `default` (versions, usernames, db names).

## No CapRover artifacts

These templates were converted from CapRover and must be cleaned:

- Remove `{{ROOT_DOMAIN}}`, `$$cap_appname`, any `$$cap_*`, and
  `srv-captain--*`. ci-agent assigns domains *after* the project is created, so
  do not bake a public URL into the compose. If the app genuinely needs a base
  URL at boot, declare an `APP_URL`/`PUBLIC_URL` variable (empty or
  `http://localhost:<port>` default) for the operator to set later.
- Remove `caprover/nginx-reverse-proxy` sidecar services. They existed only to
  bridge CapRover's single-port model; on ci-agent the real service publishes
  its own port via `{{HTTP_PORT}}`.

## Before committing

The template must satisfy: TOML parses; the `compose` YAML parses (with
placeholders treated as plain strings); every service has an image; every
placeholder is declared; no `networks:`/`container_name:` keys; one-shot
containers carry `restart: "no"`.
