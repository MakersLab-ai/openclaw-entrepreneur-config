# Admin-Agent Spec

Contract document for the Admin-Agent — a Claude Code instance running on
Christoph's personal machine that manages every OpenClaw instance in the fleet via
SSH. The Admin-Agent loads a `fleet` skill (built separately, lives in
`~/.claude/skills/openclaw-fleet/` or similar) and uses this repo as its source
of truth.

**This document is the spec for that skill.** The skill itself is not part of this
repo — this is the contract the skill must honor.

## Why the Admin-Agent is external

- **Separation of concerns.** End users never trigger install/update/fleet
  operations. The Admin-Agent runs outside the managed fleet.
- **Bootstrap simplicity.** The Admin-Agent manages instances; no one manages the
  Admin-Agent (other than `git pull` in this repo and restarting Claude Code).
- **Blast radius.** SSH credentials, migration authority, and fleet state never
  live on a managed instance.

## Command Surface

The `fleet` skill MUST expose these commands:

### `fleet list`

List all managed instances with: hostname, SSH target, current_migration,
repo_version, last-contact timestamp, failed-migrations count.

### `fleet install <instance>`

Fresh install per `docs/install.md`. Aborts if
`~/.openclaw/migration-state.json` already exists on the target.

### `fleet update <instance|all>`

Apply pending migrations per `docs/update.md`. When called with `all`, iterates
the fleet manifest and runs each instance sequentially (NOT in parallel —
sequential keeps the reporting sane and avoids authoritative-state races on the
Admin-Agent side).

### `fleet status <instance|all>`

Reports per instance:

- `current_migration`
- pending migrations (count + IDs)
- `failed[]` entries
- plugin pin status (matches HEAD vs drifted)
- cron manifest drift (matches vs drifted)

No side effects — read-only.

### `fleet create-migration <kebab-name>`

Scaffolds a new migration folder in the **base repo** (not on any instance):

1. Find the highest migration number in `core-migrations/`.
2. Create `core-migrations/<N+1 zero-padded>-<kebab-name>/migration.md` with the
   template from `core-migrations/README.md`.
3. Open the file in the Admin-Agent's editor (or print the path for Christoph to
   edit).
4. Remind Christoph: make the corresponding change in the HEAD files (`templates/`,
   `cron/`, `defaults/`, ...) in the SAME commit as the migration file.

### `fleet rollback <instance> <migration-id>`

1. Read `core-migrations/<migration-id>/migration.md`.
2. Execute the `## Rollback` section.
3. Decrement `current_migration` on the instance to the previous applied entry in
   `history[]`.
4. Append a rollback event to `history[]`.

### `fleet ssh <instance>` (convenience)

Opens an interactive SSH session to the target, using the credentials from the
fleet manifest. No special behavior beyond that.

## Fleet Manifest (Admin-Agent-local)

The Admin-Agent keeps a local manifest (NOT in this repo — it contains
hostnames, tokens, user info). Suggested location:
`~/.claude/skills/openclaw-fleet/fleet.yaml` or
`~/openclaw-fleet/` (markdown per instance, more readable).

For each instance, store at minimum:

- `name`
- `ssh_target` (user@host)
- `user_name`, `assistant_name`, `assistant_email`, `assistant_role`, `timezone`
- `telegram_user_id`, `telegram_bot_token` (reference to vault entry — do not
  inline the token)
- `telegram_topic_*` IDs
- `install_date`
- Notes

Never commit this file to the base repo.

## SSH Contract

The Admin-Agent assumes, for every managed instance:

- SSH access as the OpenClaw user (same user that runs the gateway).
- Base repo clone at `~/.openclaw-config` on the instance (created during
  install, maintained by `fleet update` via `git -C ~/.openclaw-config pull`).
- Workspace at `~/openclaw/` (templates live here after install).
- State at `~/.openclaw/` (migration-state.json, locks, logs, plugins).
- `claude` CLI on `$PATH` (verified during install).

## State Ownership

| Location                                                | Owner         | Never touched by |
| ------------------------------------------------------- | ------------- | ---------------- |
| `~/openclaw/SOUL.md`, `USER.md`, `TOOLS.md`, `IDENTITY.md` | user        | Admin-Agent (after install) |
| `~/openclaw/AGENTS.md`, `HEARTBEAT.md`                  | base repo (via migrations) | user         |
| `~/openclaw/workflows/*/rules.md`, `agent_notes.md`, `preferences.md`, `processed.md`, `logs/` | user | Admin-Agent      |
| `~/openclaw/workflows/*/AGENT.md`, `classifier.md`, etc. | base repo (via migrations) | user         |
| `~/.openclaw/openclaw.json`                             | base repo (defaults) + user overrides | Admin-Agent overwrites only via migration |
| `~/.openclaw/migration-state.json`                      | Admin-Agent   | user, base repo  |
| `~/.openclaw/plugins/*`                                 | Admin-Agent   | user             |
| Crontab entries matching the manifest                   | Admin-Agent   | user             |
| Other crontab entries                                   | user          | Admin-Agent      |
| `~/.openclaw/logs/`, `~/.openclaw/locks/`               | runtime       | everyone         |

## Escalation Rules

The Admin-Agent MUST escalate (stop + notify Christoph) in these cases:

1. A migration's instructions say "escalate" (e.g., user renamed a section the
   migration targeted).
2. A step in `install.md` / `update.md` fails with a non-idempotent error.
3. The instance's `migration-state.json` has non-empty `failed[]`.
4. A plugin install fails.
5. The gateway does not come back up after a restart.
6. SSH connection fails after 3 retries.
7. Disk / memory / other machine-level limits are breached (cross-reference
   `devops/health-check.md`).

"Escalate" means: report to Christoph via the configured admin channel, keep the
instance in its current state, do NOT attempt destructive recovery without a
human decision.

## Non-Goals

The Admin-Agent does NOT:

- Install the OpenClaw gateway itself — that's a one-time manual step per
  machine, out of scope.
- Manage user-owned files (memory, rules, notes, preferences).
- Auto-accept drift on `openclaw.json` — changes must go through migrations.
- Run the fleet in parallel by default — sequential for clarity and safety.
- Self-update this repo on instances without running the corresponding
  migrations.

## Related

- `docs/install.md` — install runbook.
- `docs/update.md` — update runbook.
- `core-migrations/README.md` — migration semantics and authoring guide.
- `cron/default-cron.md` — cron manifest.
- `plugins/groundcontrol/install.md` — GC install pin.
- `defaults/README.md` — placeholder list and default config documentation.
