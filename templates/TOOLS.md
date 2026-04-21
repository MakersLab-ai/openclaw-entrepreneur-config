# TOOLS.md - Local Notes

Skills define _how_ tools work. This file is for _your_ specifics — accounts, endpoints,
rules unique to your setup.

## Gmail (gog) — Agent Account

**Account:** {{ASSISTANT_EMAIL}}

| Action                | Allowed                                |
| --------------------- | -------------------------------------- |
| Read                  | ✅                                     |
| Draft                 | ✅                                     |
| Send to {{USER_NAME}} | ✅                                     |
| Send to anyone else   | ⚠️ Draft only — {{USER_NAME}} approves |

**Rules:**

- **To {{USER_NAME}} ({{USER_EMAIL}}):** Direct send okay (no approval needed).
- **To anyone else:** Always draft first, never send without approval. Use
  `gog gmail drafts create` for external mails.
- **No Markdown in drafts!** Gmail doesn't render Markdown. Use `--body-html` with HTML
  tags (`<b>`, `<br>`, `<p>`, `<ul>/<li>`) for formatting, or plain text with
  `--body-file`.

## GROUNDCONTROL

- **Purpose:** Shared task board for human-agent collaboration ({{USER_NAME}} +
  {{ASSISTANT_NAME}})
- **URL:** https://groundcontrol.makerslab.ai
- **Agent:** {{ASSISTANT_NAME}} (API key in OpenClaw plugin config)
- **OpenClaw Plugin:** linked from `plugins/groundcontrol/` in openclaw-config

### Rules

- **Every piece of work = a task.** Before {{ASSISTANT_NAME}} starts working — research,
  feature, analysis, anything — create the task in GROUNDCONTROL first, then work. Even
  for ad-hoc requests from chat. No task = work doesn't exist.
- **Results = GC Doc.** Research, analyses, concepts that come out as results — create a
  GC Doc and reference it from the task.
- **All markdown files live in GROUNDCONTROL Docs.** If {{ASSISTANT_NAME}} creates a
  markdown document (presentation, concept, analysis, plan), it MUST also be created as
  a GROUNDCONTROL Doc. No markdown just sitting in the workspace.
- **Task tracking is mandatory.** Every task follows: create → `in_progress` → progress
  as comments → `done`.

---

Add whatever helps you do your job. This is your cheat sheet.
