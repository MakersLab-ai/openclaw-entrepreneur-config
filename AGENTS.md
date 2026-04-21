# Project Context for AI Assistants

## Project Overview

Shareable configuration for OpenClaw AI assistant — templates, memory architecture, and
integration skills.

## Tech Stack

- This repo is a configuration / documentation layer — no application code to compile or
  run at the repo level.
- Skills (in `skills/` or `skill-candidates/`) are standalone UV scripts (Python 3.13+,
  inline dependencies). Each skill is fully self-contained.
- No shared deps, no pyproject.toml, no test suite at the repo level.

## Project Structure

- `templates/` — AGENTS.md, SOUL.md, USER.md, HEARTBEAT.md, TOOLS.md, IDENTITY.md for
  OpenClaw instances
- `core-migrations/` — Versioned update-diffs for existing instances; applied by the
  Admin-Agent (see `docs/admin-agent.md`)
- `defaults/` — Default gateway config template (`openclaw.json.template`) with
  placeholders
- `cron/` — Canonical cron manifest (`default-cron.md`)
- `plugins/` — External plugin install pins (e.g., `plugins/groundcontrol/install.md`);
  TypeScript plugin code lives in separate repos
- `skills/` — Optional user-facing UV scripts (currently empty)
- `skill-candidates/` — Archive of non-default skills for manual activation
- `workflows/` — Autonomous agents (AGENT.md upstream-owned; rules.md, agent_notes.md,
  etc. user-owned)
- `memory/` — Example memory architecture structure
- `devops/` — Health checks, machine setup, security review, notification routing
- `docs/` — Admin-Agent runbooks (install, update, spec)

## Admin-Agent Architecture

This repo is a **base repository** for managed OpenClaw instances. It does not
self-install. An external **Admin-Agent** (Claude Code on the fleet owner's machine with
SSH access to each instance and a `fleet` skill loaded) handles provisioning and
updates. The `fleet` skill lives in `~/.claude/skills/openclaw-fleet/`, not in this repo
— see `docs/admin-agent.md` for its spec.

**Grundprinzip for template changes:** The base repo is always HEAD. When a template
changes, the new content goes directly into `templates/`, and a migration folder is
added to `core-migrations/` describing the diff for instances on the previous state.
Fresh installs do NOT replay migrations — they copy HEAD and set `current_migration` to
the latest ID. See `core-migrations/README.md`.

## Template Placeholders

Admin-Agent replaces these during `fleet install` (literal find-and-replace, no runtime
engine): `{{USER_NAME}}`, `{{ASSISTANT_NAME}}`, `{{TIMEZONE}}` (default
`Europe/Vienna`), `{{ASSISTANT_EMAIL}}`, `{{ASSISTANT_ROLE}}`. Gateway config adds
`{{TELEGRAM_BOT_TOKEN}}`, `{{TELEGRAM_USER_ID}}`, `{{TELEGRAM_TOPIC_*}}`. Full list in
`defaults/README.md`.

## User-Owned vs Base-Repo-Owned Files

Critical for update safety. Never touched by the Admin-Agent after install:

- Templates: `SOUL.md`, `USER.md`, `TOOLS.md`, `IDENTITY.md`
- Workflows: `rules.md`, `agent_notes.md`, `preferences.md`, `processed.md`, `logs/`
- `~/.openclaw/openclaw.json` — user overrides survive (changes reach the instance only
  via migrations, never blind overwrite)

Updated by the Admin-Agent (via `fleet update` + applicable migrations):

- Templates: `AGENTS.md`, `HEARTBEAT.md`
- Workflow algorithm files: `AGENT.md`, `classifier.md`, platform guides
- Plugin versions (per `plugins/*/install.md` pin)
- Crontab entries matching `cron/default-cron.md`

## Code Conventions

- Skills are standalone UV scripts with `# /// script` inline dependencies — no
  pyproject.toml, no shared code between skills. Each skill is fully self-contained.
- Each skill has `SKILL.md` (metadata) + executable script (same name as directory)
- Bump `VERSION` file and skill's `SKILL.md` version on changes
- Keep README.md in sync: update the skill/workflow tables when adding, removing, or
  versioning skills and workflows (counts and version badges are intentionally omitted
  from the README to avoid maintenance burden)
- Keep secrets out of the repo — API keys and `.env` files stay local
- **This is a public repo — zero PII, zero fleet specifics.** This repo is a shareable
  template that anyone can clone. Never include: real names, Telegram/chat IDs, phone
  numbers, bot usernames, API keys, IP addresses, hostnames, instance IDs, security
  group IDs, or any other personally identifiable information. Use generic placeholders
  (`<ADMIN_NAME>`, `<TELEGRAM_ID>`, `<BOT_TOKEN>`, `machine-1`) in examples. Fleet-
  specific details belong in `~/openclaw-fleet/` (private) or `CLAUDE.local.md`
  (gitignored). When in doubt, use a placeholder.
- Store persistent state in markdown, not JSON — agents read and update markdown
  naturally without parsing. JSON is fine for API responses and tool output (`--json`
  flags), but state files (health reports, security posture, drift baselines) should be
  markdown

## Deployment Model

This repo is the **base repository** for managed OpenClaw instances. The external
**Admin-Agent** (a separate Claude Code instance on the fleet owner's machine) clones
this repo, then provisions and updates each instance over SSH.

- `templates/` → copied to the instance workspace at install (placeholders substituted
  once). Subsequent changes flow through `core-migrations/`.
- `defaults/openclaw.json.template` → rendered + placed at install. Subsequent changes
  flow through migrations.
- `cron/default-cron.md` → idempotent-applied to the instance's user crontab at install
  and every update.
- `plugins/*/install.md` → Git-based plugin pins; Admin-Agent clones + builds on the
  instance, re-runs on pin change.
- `skills/` → copied to the instance workspace on install/update (safe to overwrite;
  currently empty).
- `workflows/` → AGENT.md / classifier.md / platform guides are base-repo-owned (updated
  via migrations). `rules.md`, `agent_notes.md`, `preferences.md`, `processed.md`,
  `logs/` are user-owned and never touched.
- `devops/` → health check + machine setup + security review specs, consumed by cron
  jobs on the instance.
- `core-migrations/` → Admin-Agent reads these on every `fleet update`, applies any
  whose ID is higher than the instance's `current_migration`.

The Admin-Agent itself is NOT in this repo — see `docs/admin-agent.md` for its spec.
Runbooks: `docs/install.md`, `docs/update.md`.

## Naming History

This project has been renamed twice: **Clawdbot** → **Moltbot** → **OpenClaw** (current,
after Anthropic trademark concerns). Legacy paths and config references using old names
still exist — both work, but prefer `openclaw` going forward.

## Git Workflow

Commit style: Conventional or emoji prefix, Co-Authored-By for AI commits.

Example: `Rewrite Parallel.ai skill from Bash to Python`
