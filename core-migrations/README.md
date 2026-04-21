# Core Migrations

Versioned update-diffs that bring existing OpenClaw instances from an older state up
to the current repo HEAD. Applied by the **Admin-Agent** via SSH, not by the
instances themselves.

## Grundprinzip: Base-Repo ist HEAD, Migrations sind Diffs

The files in `templates/`, `cron/`, `defaults/`, `plugins/`, `workflows/` always
reflect the **current state**. When Christoph makes a change, the change goes
directly into those files — and a migration file is added here that describes the
**diff** for instances that were running the previous version.

**Consequences:**

- **Fresh install does NOT replay migrations.** The Admin-Agent copies the current
  HEAD content and sets `current_migration` to the latest migration ID in this
  folder. See `docs/install.md`.
- **Update applies only newer migrations.** The Admin-Agent reads the instance's
  `current_migration`, finds migrations in this folder with a higher ID, and runs
  each one. See `docs/update.md`.
- **No baseline migration.** There is no `0001-initial-install` — install is a
  separate code path in the Admin-Skill, not a migration.

## Folder Layout

```
core-migrations/
├── README.md                       ← this file
├── 0001-<kebab-name>/
│   ├── migration.md                ← authoritative runbook for the Admin-Agent
│   └── files/                      ← (optional) new files to add during the migration
│       └── some-new-file.md
└── 0002-<kebab-name>/
    └── migration.md
```

Numbering is zero-padded, monotonically increasing, four digits. No gaps, no
renumbering after the fact.

## Migration Format

`migration.md` is a **runbook for the Admin-Agent**, not a human changelog. It tells
Claude exactly what to do on each existing instance. Plain prose, deterministic
steps, explicit idempotency checks.

```markdown
---
id: 0002-add-telegram-reminder
created: 2026-05-01
affects:
  - templates/HEARTBEAT.md
---

## What changed

HEARTBEAT.md gains a new "Telegram Reminder" bullet in the "Every X hours" section.

## Why

[One or two sentences — for Christoph when he looks at this in 6 months.]

## Admin-Agent Instructions

For each managed instance:

1. SSH, read `~/.openclaw/workspace/HEARTBEAT.md`.
2. Check whether the section `## Every X hours` contains a bullet mentioning
   "Telegram".
   - If yes: skip (idempotent). Mark migration applied.
   - If no: insert the following bullet immediately after the section heading:
     ```
     - Check Telegram unread count via `tgcli unread`
     ```
3. Edge case: if the section `## Every X hours` is missing (user renamed or deleted
   it), do NOT apply — escalate to Christoph.

## Rollback

Remove the added bullet from `~/.openclaw/workspace/HEARTBEAT.md`.
```

## Key Rules for Writing Migrations

1. **Describe the diff, not the destination.** "Add bullet X" is correct. "HEARTBEAT.md
   should contain bullet X" is wrong — that's just the current repo state.
2. **Idempotency first.** Every step checks if the change is already present before
   applying. Migrations must be safe to re-run.
3. **Escalate on drift.** If the file has been user-modified in a way that blocks the
   migration (section deleted, renamed, etc.), stop and escalate — do not guess.
4. **Provide a rollback section.** Even if just "remove the added line".
5. **One conceptual change per migration.** Don't bundle unrelated template edits
   into the same migration folder — each migration should be revertible on its own.

## State Tracking on the Instance

Each managed instance has `~/.openclaw/migration-state.json`:

```json
{
  "repo_version": "0.25.0",
  "current_migration": "0002-add-telegram-reminder",
  "history": [
    {
      "event": "install",
      "at": "2026-04-20T10:00:00Z",
      "baseline_migration": "0001-<name>",
      "by": "admin-agent@<admin-host>"
    },
    {
      "event": "apply",
      "at": "2026-05-01T14:20:00Z",
      "migration_id": "0002-add-telegram-reminder",
      "by": "admin-agent@<admin-host>"
    }
  ],
  "failed": []
}
```

- `current_migration` — the latest migration successfully applied (or the baseline
  set at install time).
- `history` — append-only audit log.
- `failed` — any migration that failed. The Admin-Agent blocks further updates on
  that instance until the entry is resolved.

## Relationship to Other Repo Parts

- **`templates/`, `cron/`, `defaults/`, `plugins/groundcontrol/install.md`** — always
  current. Edited directly when a change is made.
- **`workflows/*/AGENT.md`** — upstream-owned. When these change, add a migration
  here so existing instances get updated.
- **`workflows/*/rules.md`, `agent_notes.md`, `preferences.md`, `processed.md`,
  `logs/`** — user-owned. Migrations never touch these.
- **`docs/install.md`** — defines fresh-install flow (not a migration).
- **`docs/update.md`** — defines the update flow (migration replay).
- **`docs/admin-agent.md`** — Admin-Skill command surface, including
  `fleet create-migration <name>` which scaffolds a new folder here.
