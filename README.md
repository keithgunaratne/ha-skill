# ha-skill — Home Assistant Claude Code skill

A set of POSIX shell scripts that let [Claude Code](https://claude.com/claude-code) safely
inspect and manage a Home Assistant instance running in a Docker container: query the REST
API, validate config, tail logs, reload automations, restart safely, and snapshot config to
git.

---

## Installation

This is a Claude Code *skill*. Clone it into your Home Assistant config directory under
`.claude/skills/ha-skill/` (where Claude Code discovers skills):

```sh
git clone https://github.com/keithgunaratne/ha-skill.git \
  /path/to/your/homeassistant/.claude/skills/ha-skill
```

Then create your `.env` from the template (next section).

---

## Setup: long-lived access token

1. In the Home Assistant UI, go to **Profile → Long-Lived Access Tokens → Create Token**.
2. Copy the token — it is only shown once.
3. Copy the template and fill it in:

   ```sh
   cp env.example .env
   ```

   Set `HA_TOKEN` to your real token. **Never commit `.env`** — it is listed in `.gitignore`.

---

## Environment variables (`.env`)

| Variable        | Description                              | Example                         |
|-----------------|------------------------------------------|---------------------------------|
| `HA_URL`        | Base URL of the HA API                   | `http://127.0.0.1:8123`         |
| `HA_TOKEN`      | Long-lived access token                  | *(must be set manually)*        |
| `HA_CONFIG_DIR` | Absolute host path to the HA config dir  | `/path/to/your/homeassistant`   |
| `HA_CONTAINER`  | Docker container name                    | `homeassistant`                 |

---

## Available scripts

All scripts live in `bin/` and are run with their full path from the HA config root. They
load credentials from `.env` and fail loudly if the token is missing or still a placeholder.

### `ha-api METHOD PATH [JSON_BODY]`

Generic REST API wrapper. All other API scripts call this one.

```sh
.claude/skills/ha-skill/bin/ha-api GET /api/
.claude/skills/ha-skill/bin/ha-api POST /api/services/light/turn_on '{"entity_id":"light.living_room"}'
```

### `ha-states`

Fetch all entity states from the API.

```sh
.claude/skills/ha-skill/bin/ha-states | python3 -m json.tool | grep '"entity_id"'
```

### `ha-services`

List all available services.

```sh
.claude/skills/ha-skill/bin/ha-services | python3 -m json.tool
```

### `ha-check-config`

Run HA's built-in config validator inside the container. Always run this before a reload or
restart.

```sh
.claude/skills/ha-skill/bin/ha-check-config
```

### `ha-logs [N]`

Show the last N lines of the container's logs (default 200).

```sh
.claude/skills/ha-skill/bin/ha-logs
.claude/skills/ha-skill/bin/ha-logs 50
```

### `ha-reload-automations`

Reload automations without restarting the container. Prefer this over `ha-restart` when you
have only changed automations.

```sh
.claude/skills/ha-skill/bin/ha-reload-automations
```

### `ha-restart`

Full restart workflow: validate config → show `git diff` → ask for explicit confirmation →
restart container → tail logs. Never skips validation or confirmation.

```sh
.claude/skills/ha-skill/bin/ha-restart
```

### `ha-wait-ready [TIMEOUT_SECONDS]`

Poll the HA API (authenticated) until it responds successfully, then exit 0. Exits 1 if the
timeout (default 120s) is reached. Always use this instead of a bare
`until curl .../api/` loop — unauthenticated requests generate invalid-auth warnings in HA.

```sh
.claude/skills/ha-skill/bin/ha-wait-ready
.claude/skills/ha-skill/bin/ha-wait-ready 60
```

### `ha-nightly-snapshot`

Stage all non-ignored config changes, commit them (`chore: nightly snapshot <date>`), and push
to `origin` if a remote is configured. Intended to run nightly via cron so UI-made edits are
captured automatically. Resolves the repo root from `HA_CONFIG_DIR`, falling back to its own
location, so it has no hardcoded paths.

```sh
.claude/skills/ha-skill/bin/ha-nightly-snapshot
```

---

## Safe workflow

```
inspect   →   edit   →   validate   →   diff   →   reload / restart
```

1. **Inspect** — read the relevant YAML, query states/services with `ha-states` / `ha-services`.
2. **Edit** — make the change in the appropriate config file.
3. **Validate** — run `ha-check-config`. Fix any errors before continuing.
4. **Diff** — review `git diff` to confirm only intended changes are present.
5. **Reload/restart** — use `ha-reload-automations` for automation-only changes; use
   `ha-restart` only when a full restart is necessary.

---

## Security reminders

- `.env` holds your access token. It must never be committed — it is in `.gitignore`.
- The scripts never hardcode the token; they read it from `.env` at runtime.
- Never print, log, or pass the token to untrusted processes.
