# Update Runbook — Apply Pending Migrations

Runbook for the **Admin-Agent** to bring a managed OpenClaw instance from its
current migration state up to the base repo's HEAD.

**Triggered by:** `fleet update <instance|all>` in the Admin-Skill.

**Preconditions:**

- SSH access to the target machine.
- Instance already installed (i.e., `~/.openclaw/migration-state.json` exists).
- Base repo checked out on the Admin-Agent's machine (not on the instance —
  migrations are applied over SSH).

---

## Steps

### 1. Pull the base repo on the Admin-Agent

```bash
git -C <base-repo> pull --ff-only
```

If there are local uncommitted changes, stop and escalate.

### 2. Read the instance's current state

```bash
ssh <instance> 'cat ~/.openclaw/migration-state.json'
```

Parse `current_migration` and check `failed[]`.

> **If `failed[]` is non-empty:** Stop. The Admin-Agent must not apply further
> migrations until the failed one is resolved. Escalate with the failed entry's
> details.

### 3. Compute the pending list

List folders in `core-migrations/` (sorted by numeric prefix). Keep only folders
whose ID is strictly greater than `current_migration`. If `current_migration` is
`null` (first update after install), apply all migrations.

### 4. Apply each pending migration

For each migration folder in order:

1. Read `core-migrations/<id>/migration.md`.
2. Follow the `## Admin-Agent Instructions` section literally. Each step has an
   idempotency check — respect it.
3. If the migration has a `files/` subfolder, use it as the source for any files
   that need to be added to the instance.
4. On success:
   - Update `current_migration` in the instance's `migration-state.json`.
   - Append an entry to `history[]`:
     ```json
     {
       "event": "apply",
       "at": "<iso-timestamp-utc>",
       "migration_id": "<id>",
       "by": "admin-agent@<admin-host>"
     }
     ```
5. On failure (instructions say "escalate", or a step errors out):
   - Do NOT update `current_migration`.
   - Append an entry to `failed[]`:
     ```json
     {
       "migration_id": "<id>",
       "at": "<iso-timestamp-utc>",
       "reason": "<short description>",
       "details": "<error output or blocking drift>"
     }
     ```
   - Stop. Do not attempt subsequent migrations. Report to Christoph.

### 5. Re-apply plugin pins

Read `plugins/*/install.md` and check whether the pinned ref has changed since the
previous `update`. For each plugin with a new pin:

1. Follow the plugin's `install.md` Install Steps (idempotent — git checkout to
   the new ref, `npm install && npm run build`, symlink stays the same).
2. Log the pin change in `history[]`:
   ```json
   {
     "event": "plugin-update",
     "at": "<iso-timestamp-utc>",
     "plugin": "<name>",
     "from": "<old-ref>",
     "to": "<new-ref>"
   }
   ```

### 6. Re-apply cron manifest

Follow the idempotent-apply procedure in `cron/default-cron.md`. The Admin-Agent
reads the instance's current crontab, diffs against the manifest, adds/replaces
entries that drifted, and leaves matched entries alone.

Never blow away the whole crontab — personal entries must survive.

### 7. Re-apply `openclaw.json` defaults (if changed)

If `defaults/openclaw.json.template` has changed since the previous `update`:

1. Re-render it with the instance's stored placeholder values (the Admin-Agent
   keeps these in the fleet manifest).
2. Diff against the current `~/.openclaw/openclaw.json` on the instance.
3. If the user has customized fields the Admin-Agent would change, escalate
   instead of overwriting.

Treat this like a migration: prefer explicit `core-migrations/*/migration.md`
entries for non-trivial changes.

### 8. Update `repo_version`

Write the current repo `VERSION` into `migration-state.json.repo_version`.

### 9. Restart the gateway (if needed)

Only restart if any plugin was updated or the gateway config changed:

```bash
ssh <instance> 'openclaw gateway restart'
```

Prefer the graceful-restart semantics of the native `openclaw gateway` CLI — do
not hard-kill the process.

### 10. Smoke test

Run a health check:

```bash
ssh <instance> 'flock -n ~/.openclaw/locks/health-check.lock claude -p "Run health check" --model simple --append-system-prompt-file ~/.openclaw-config/devops/health-check.md --dangerously-skip-permissions --max-budget-usd 1.00'
```

### 11. Report to Christoph

Summary:

- Instance
- Migrations applied (IDs + one-line descriptions)
- Plugin pins changed
- Cron drift fixed
- Smoke test result

---

## Failure Recovery

- **Migration failure:** state stays consistent (current_migration only advances on
  success). After Christoph resolves the issue and clears the `failed[]` entry,
  the Admin-Agent can resume from the next migration.
- **Rollback a specific migration:** `fleet rollback <instance> <migration-id>`
  reads the migration's `## Rollback` section and applies it. The Admin-Agent
  decrements `current_migration` to the previous applied migration.

## Related

- `docs/install.md` — first-time install.
- `docs/admin-agent.md` — command surface.
- `core-migrations/README.md` — migration semantics and authoring guide.
