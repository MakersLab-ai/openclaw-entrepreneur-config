# Default Cron Configuration

Canonical schedule for every managed OpenClaw instance. The Admin-Agent reads this
file during `fleet install` and `fleet update` and writes matching entries into the
instance's user crontab.

**Source of truth.** If an instance's crontab drifts from this manifest,
`fleet update` re-aligns it (idempotent — existing entries that match are left
alone).

## Conventions

- All jobs run as the OpenClaw user on the instance.
- Logs go to `~/.openclaw/logs/cron-<job>.log` unless noted otherwise.
- Locking: every job uses `flock -n <lockfile>` to prevent overlap.
- `CLAUDE_PATH` is resolved per-instance during install (`which claude`) and
  substituted into the crontab entries below.
- Topic delivery (where applicable) is handled by the job's workflow / agent logic,
  not by the cron wrapper — see `devops/notification-routing.md` for the two-lane
  model.

## Jobs

| Job                  | Schedule            | Model  | Purpose                               | Notify     |
| -------------------- | ------------------- | ------ | ------------------------------------- | ---------- |
| health-check         | `*/30 * * * *`      | simple | Gateway + service health              | on failure |
| email-steward        | `*/30 7-22 * * *`   | haiku  | Inbox triage                          | flagged    |
| task-steward         | `0 8-22 * * *`      | haiku  | Task classification + execution       | silent     |
| calendar-steward     | `0 8 * * *`         | haiku  | Daily briefing                        | daily      |
| contact-steward      | `0 7 * * *`         | haiku  | Unknown-contact detection             | silent     |
| cron-healthcheck     | `5 * * * *`         | haiku  | Detect broken cron jobs               | on failure |
| learning-loop        | `0 23 * * *`        | opus   | Capture corrections → patterns        | silent     |
| llm-usage-report     | `0 20 * * *`        | haiku  | Daily LLM spend digest                | daily      |
| security-sentinel    | `0 4 * * 1`         | opus   | Weekly threat intelligence research   | on finding |
| bridge-health        | `*/15 * * * *`      | simple | Telegram/WhatsApp/Slack bridge health | on failure |
| forward-motion       | `0 9 * * *`         | haiku  | Morning priorities surface            | daily      |

## Crontab Entries

The Admin-Agent assembles the crontab from these templates. Each entry is
self-contained and locks itself.

### health-check

```cron
*/30 * * * * test -f "$HOME/.openclaw-config/devops/health-check.md" && flock -n "$HOME/.openclaw/locks/health-check.lock" CLAUDE_PATH -p "Run health check" --model simple --append-system-prompt-file "$HOME/.openclaw-config/devops/health-check.md" --dangerously-skip-permissions --max-budget-usd 5.00 >> "$HOME/.openclaw/logs/cron-health-check.log" 2>&1
```

### Workflow jobs (template)

For each workflow listed in the table, the crontab entry follows this shape — the
Admin-Agent substitutes `<SCHEDULE>`, `<WORKFLOW>`, `<MODEL>`:

```cron
<SCHEDULE> test -d "$HOME/openclaw/workflows/<WORKFLOW>" && flock -n "$HOME/.openclaw/locks/<WORKFLOW>.lock" CLAUDE_PATH -p "Run <WORKFLOW>" --model <MODEL> --append-system-prompt-file "$HOME/openclaw/workflows/<WORKFLOW>/AGENT.md" --dangerously-skip-permissions --max-budget-usd 10.00 >> "$HOME/.openclaw/logs/cron-<WORKFLOW>.log" 2>&1
```

### Groundcontrol worker

See `plugins/groundcontrol/install.md` — the plugin ships its own worker script.
Canonical schedule:

```cron
*/5 * * * * "$HOME/repositories/groundcontrol/openclaw-plugin/scripts/gc-worker.sh" >> "$HOME/.openclaw/logs/gc-worker.log" 2>&1
```

## Prerequisites

The Admin-Agent must ensure the following exist on the instance before writing the
crontab:

```bash
mkdir -p "$HOME/.openclaw/locks" "$HOME/.openclaw/logs"
```

## Idempotent Apply

The Admin-Agent applies this config via the following procedure (documented fully
in `docs/update.md`):

1. Read the current user crontab.
2. For each entry in the manifest, check whether an equivalent line exists. Match
   by a stable marker (the lockfile path is unique per job).
3. Add missing entries, replace drifted entries, leave matched entries alone.
4. Write the updated crontab back.

Never blow away the whole crontab — the user may have personal entries that should
survive.
