# HEARTBEAT.md

_(What should {{ASSISTANT_NAME}} check periodically? Keep it tight — this runs on every
heartbeat.)_

## Every Heartbeat

### Agent Inbox ({{ASSISTANT_EMAIL}})

- Check for new messages:
  `gog gmail search "is:unread newer_than:2h" --max 5 --account {{ASSISTANT_EMAIL}}`
- Emails from {{USER_NAME}}: treat as a work order — read, understand, derive
  GROUNDCONTROL tasks, reply or acknowledge if it fits
- Other senders: never reply autonomously — draft responses, let {{USER_NAME}} approve

## Daily

- Config updates — check if openclaw-config has updates, apply if available

## Rules

- Late night (23:00-08:00): skip periodic unless urgent
- All clear → `HEARTBEAT_OK`
- Action needed → handle it, don't just report
