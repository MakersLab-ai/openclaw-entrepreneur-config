# Install Runbook — Fresh OpenClaw Instance

Runbook for the **Admin-Agent** to provision a new managed OpenClaw instance. This
is NOT a migration replay — it copies the current repo HEAD and sets
`current_migration` to the latest migration ID.

**Triggered by:** `fleet install <instance>` in the Admin-Skill.

**Preconditions:**

- SSH access to the target machine as the OpenClaw user.
- OpenClaw gateway is already installed on the machine (this repo is a
  configuration layer — it does NOT install the gateway itself).
- `claude` CLI is available on the target machine (`which claude` returns a path).

---

## Steps

### 1. Abort if already installed

```bash
ssh <instance> 'test -e ~/.openclaw/migration-state.json'
```

If the file exists, this is an update, not an install. Stop and tell the user to
run `fleet update <instance>` instead.

### 2. Clone the base repo to the instance

```bash
ssh <instance> 'git clone <BASE_REPO_URL> ~/.openclaw-config'
```

If the directory already exists (e.g., leftover from a failed previous install),
warn and confirm with Christoph before overwriting.

### 3. Collect instance-specific values

Prompt Christoph (or read from the fleet manifest in the Admin-Agent) for:

| Value                   | Required | Example               |
| ----------------------- | -------- | --------------------- |
| `USER_NAME`             | yes      | `Christoph`           |
| `ASSISTANT_NAME`        | yes      | `Clawdia`             |
| `ASSISTANT_EMAIL`       | yes      | `clawdia@example.com` |
| `ASSISTANT_ROLE`        | yes      | `ops assistant`       |
| `TIMEZONE`              | yes      | `Europe/Vienna`       |
| `TELEGRAM_BOT_TOKEN`    | yes      | (from vault)          |
| `TELEGRAM_USER_ID`      | yes      | numeric               |
| `TELEGRAM_TOPIC_*`      | optional | thread IDs            |

### 4. Copy + render templates

For each file in `templates/` (`AGENTS.md`, `SOUL.md`, `USER.md`, `HEARTBEAT.md`,
`TOOLS.md`, `IDENTITY.md`):

1. Read the source file from the cloned repo on the instance.
2. Substitute placeholders (`{{USER_NAME}}`, `{{ASSISTANT_NAME}}`, `{{TIMEZONE}}`,
   `{{ASSISTANT_EMAIL}}`, `{{ASSISTANT_ROLE}}`). See `defaults/README.md` for the
   full placeholder list.
3. Write to `~/openclaw/` on the instance — do NOT overwrite if the file already
   exists there.

> **Memory folders:** Do NOT create `memory/people/`, `memory/projects/`,
> `memory/topics/`, `memory/decisions/`. Those are legacy Cortex residue.
> OpenClaw's native memory system uses `MEMORY.md` + `memory/YYYY-MM-DD.md` files,
> indexed by `openclaw memory index` — no pre-seeded subfolders needed.

### 5. Render + place `openclaw.json`

1. Read `defaults/openclaw.json.template` from the cloned repo on the instance.
2. Substitute placeholders (see `defaults/README.md`).
3. Write to `~/.openclaw/openclaw.json` — do NOT overwrite if the file already
   exists; warn and ask.

### 6. Install the Groundcontrol plugin

Follow `plugins/groundcontrol/install.md` verbatim. On failure, escalate — do not
silently continue (tasks + initiatives don't work without the plugin).

### 7. Install the default cron

Follow the idempotent-apply procedure in `cron/default-cron.md`:

1. `mkdir -p ~/.openclaw/locks ~/.openclaw/logs` on the instance.
2. Resolve `CLAUDE_PATH` via `ssh <instance> 'which claude'`.
3. Read current crontab (`crontab -l`, handle empty).
4. Add each job from the manifest (using the rendered command with `CLAUDE_PATH`
   substituted).
5. Write the updated crontab back (`crontab -`).
6. Verify with `crontab -l | grep openclaw`.

### 8. Write `migration-state.json`

1. Find the newest migration folder in `core-migrations/` on the instance (or
   `null` if the folder only contains `README.md`).
2. Write `~/.openclaw/migration-state.json`:

   ```json
   {
     "repo_version": "<contents of VERSION>",
     "current_migration": "<newest-migration-id-or-null>",
     "history": [
       {
         "event": "install",
         "at": "<iso-timestamp-utc>",
         "baseline_migration": "<newest-migration-id-or-null>",
         "by": "admin-agent@<admin-host>"
       }
     ],
     "failed": []
   }
   ```

### 9. Restart the gateway

```bash
ssh <instance> 'openclaw gateway restart'
```

Wait for it to come back. Verify:

```bash
ssh <instance> 'openclaw gateway status'
ssh <instance> 'openclaw plugin list | grep groundcontrol'
```

### 10. Smoke test

Send a test message to the Telegram bot (or trigger health-check manually):

```bash
ssh <instance> 'flock -n ~/.openclaw/locks/health-check.lock claude -p "Run health check" --model simple --append-system-prompt-file ~/.openclaw-config/devops/health-check.md --dangerously-skip-permissions --max-budget-usd 1.00'
```

Expect a healthy report.

### 11. Report to Christoph

Summary message:

- Instance hostname + user
- Assistant name, email
- Current migration (baseline)
- Enabled plugins
- Enabled cron jobs
- Anything the smoke test flagged

---

## Failure Recovery

If any step fails:

1. Leave the instance in its current partial state (do NOT auto-rollback).
2. Escalate to Christoph with: which step failed, the error output, and the state
   of `migration-state.json` (if it was already written).
3. A failed install can be resumed manually by re-running the failed step once the
   underlying issue is fixed.

## Related

- `docs/update.md` — post-install updates.
- `docs/admin-agent.md` — command surface the Admin-Skill exposes.
- `core-migrations/README.md` — migration semantics.
