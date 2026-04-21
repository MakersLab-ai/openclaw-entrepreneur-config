# Defaults

Default configuration files that the Admin-Agent copies to a new OpenClaw instance
during `fleet install`. Placeholders are substituted from values collected during
install (or, for secrets, from the Admin-Agent's vault).

## Files

- **`openclaw.json.template`** — Gateway config (channels, memory search, plugins,
  safety limits). Rendered to `~/.openclaw/openclaw.json` on the instance.

## Placeholders

The Admin-Agent replaces these at install time with a literal find-and-replace. No
runtime substitution engine — if the placeholder string is present, the value is
inserted verbatim.

### Template identity placeholders (also used in `templates/`)

| Placeholder           | Source                                     | Example                |
| --------------------- | ------------------------------------------ | ---------------------- |
| `{{USER_NAME}}`       | Installer prompt                           | `Christoph`            |
| `{{USER_FULL_NAME}}`  | Installer prompt                           | `Christoph Schleifer`  |
| `{{ASSISTANT_NAME}}`  | Installer prompt                           | `Clawdia`              |
| `{{ASSISTANT_EMAIL}}` | Installer prompt                           | `clawdia@example.com`  |
| `{{ASSISTANT_ROLE}}`  | Installer prompt                           | `operations assistant` |
| `{{TIMEZONE}}`        | Installer prompt (default `Europe/Vienna`) | `Europe/Vienna`        |

### `openclaw.json.template`-specific placeholders

| Placeholder                     | Source                      | Notes                |
| ------------------------------- | --------------------------- | -------------------- |
| `{{TELEGRAM_BOT_TOKEN}}`        | Admin-Agent vault           | Per-instance bot     |
| `{{TELEGRAM_USER_ID}}`          | Installer prompt            | Allowlisted end-user |
| `{{TELEGRAM_TOPIC_HOME}}`       | Installer prompt (optional) | Thread ID            |
| `{{TELEGRAM_TOPIC_AUTOMATION}}` | Installer prompt (optional) | Thread ID            |
| `{{TELEGRAM_TOPIC_SYSTEM}}`     | Installer prompt (optional) | Thread ID            |

## Why JSON here, not Markdown

Gateway config is machine-consumed, not agent-read. JSON is the right format. See
`CLAUDE.md` / `AGENTS.md` — the "markdown for state files, JSON for machine config" rule
still holds.

## Related

- `docs/install.md` — consumes these defaults during `fleet install`.
- `docs/update.md` — re-applies changed defaults via a migration (not blind overwrite).
