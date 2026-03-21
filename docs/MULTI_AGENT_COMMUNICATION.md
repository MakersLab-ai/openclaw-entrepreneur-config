# Multi-Agent Communication Patterns

**Date:** 2026-03-21 **Status:** Tested & Working **Agents:** Cora, Shelly, Hex (across
3 separate OpenClaw gateways)

## Quick Summary

When you have multiple independent OpenClaw agents on separate gateway instances that
need to coordinate:

1. **Use a shared Slack channel as the communication bus**
2. **Enable @-mention routing for agent-to-agent messages**
3. **Configure loop detection to prevent infinite back-and-forths**

This is the only cross-gateway pattern that works today. Within a single gateway, agents
can use native `sessions_send` for invisible backend coordination.

---

## Architecture

```
┌──────────────────┐       ┌──────────────────┐       ┌──────────────────┐
│ Gateway A        │       │ Gateway B        │       │ Gateway C        │
│ (Cora/main)      │       │ (Shelly/main)    │       │ (Hex/main)       │
│                  │       │                  │       │                  │
│ ~/.openclaw/...  │       │ ~/.openclaw/...  │       │ ~/.openclaw/...  │
│ Sessions: local  │       │ Sessions: local  │       │ Sessions: local  │
└──────────────────┘       └──────────────────┘       └──────────────────┘
         │                         │                         │
         └─────────────────────────┼─────────────────────────┘
                                   │
                    ┌──────────────────────────┐
                    │  Slack Channel Bus       │
                    │  C0AMLA1SG67            │
                    │                          │
                    │ • @-mention routing      │
                    │ • allowBots: true        │
                    │ • requireMention: true   │
                    │ • Loop detection active  │
                    └──────────────────────────┘
```

---

## Configuration Template

Add this to **every OpenClaw gateway's `~/.openclaw/openclaw.json`**:

```json
{
  "agents": {
    "list": [
      {
        "id": "main",
        "groupChat": {
          "mentionPatterns": ["\\b{name}\\b"]
        }
      }
    ]
  },
  "channels": {
    "slack": {
      "channels": {
        "C0AMLA1SG67": {
          "requireMention": true,
          "allowBots": true
        }
      }
    }
  },
  "tools": {
    "agentToAgent": {
      "enabled": true
    },
    "sessions": {
      "visibility": "all"
    },
    "loopDetection": {
      "enabled": true,
      "genericRepeat": true,
      "pingPong": true
    }
  },
  "session": {
    "agentToAgent": {
      "maxPingPongTurns": 5
    }
  }
}
```

### Setting Reference

| Key                     | Value        | Purpose                                                                   |
| ----------------------- | ------------ | ------------------------------------------------------------------------- |
| `mentionPatterns`       | `\b{name}\b` | Case-insensitive regex. Humans say "cora help"; agents say "@shelly ...". |
| `requireMention`        | `true`       | Require explicit @-mention in Slack (prevents accidental triggers).       |
| `allowBots`             | `true`       | Allow bot-to-bot @-mentions (REQUIRED for agent coordination).            |
| `agentToAgent.enabled`  | `true`       | Unlock cross-agent tool coordination.                                     |
| `sessions.visibility`   | `"all"`      | Agents can see each other's session keys for orchestration.               |
| `loopDetection.enabled` | `true`       | Prevent infinite loops (e.g., A asks B, B asks A, A asks B...).           |
| `maxPingPongTurns`      | `5`          | Max 5 back-and-forths before breaker stops loop.                          |

---

## Message Flow Examples

### Human → Agent (Natural Language)

```
Nick (in Slack): "cora what's happening?"
│
├─ Parsed as mentionPattern match
├─ Routes to Cora's main session
├─ Cora processes, replies
└─ Reply appears in Slack
```

### Agent → Agent (@-mention Required)

```
Cora (thinking): "I need Shelly's help"
│
├─ Sends: "@shelly can you research X?"
├─ Slack routes as new inbound message to Shelly
├─ Shelly's mention pattern matches "shelly"
├─ Routes to Shelly's main session
├─ Shelly processes, replies
└─ Cora sees response in channel history
```

### Loop Detection in Action

```
Cora: "Hey @shelly, what's the status?"
Shelly: "@hex do you know Cora's status?"
Hex: "@cora can you help Shelly?"
Cora: "@shelly @hex I think we're in a loop..."
│
└─ Loop detector recognizes ping-pong pattern
   After 5 exchanges, further messages in this
   conversation thread are silently dropped
```

---

## Agent-to-Agent Coordination Patterns

### 1. Request-Response (Synchronous)

**When:** One agent needs info from another to continue.

```python
# In Cora's brain:
send_slack_message("@shelly what's the status of task X?")
time.sleep(1)  # Brief wait for Slack delivery
results = sessions_history(sessionKey="agent:shelly:main")
# Extract Shelly's reply from history
synthesize_results(my_task, shelly_input)
```

**Pros:** Simple, synchronous-looking **Cons:** Depends on response timing

### 2. Broadcast Announcement

**When:** One agent announces a state change; others may react.

```
Cora: "@shelly @hex NEW TASK AVAILABLE: [details]"
```

**Flow:**

- Shelly sees her mention, activates
- Hex sees his mention, activates
- Both can react independently (thread replies, acknowledgments, etc.)

**Pros:** One message to many agents **Cons:** No guarantee of receipt

### 3. Work Distribution (Pool Pattern)

**When:** Coordinator has tasks; workers claim and execute.

```
Cora: "@worker1 @worker2 here are 10 tasks:
  1. Task A
  2. Task B
  [...]"

Worker1: (reads history, claims Task A, posts: "Claiming Task A")
Worker2: (reads history, claims Task B, posts: "Claiming Task B")

Cora: (reads history, collects results, synthesizes)
```

**Pros:** Parallel work **Cons:** Requires discipline (locking, idempotency)

---

## Troubleshooting

### Agent Doesn't Respond to @-mention

**Check 1: Is the agent online?**

```bash
openclaw status
```

**Check 2: Is the mention pattern correct?**

```bash
# Gateway logs should show pattern match
openclaw gateway logs | grep mentionPattern
```

**Check 3: Is `allowBots: true` set?**

```bash
grep -A3 "C0AMLA1SG67" ~/.openclaw/openclaw.json | grep allowBots
```

**Fix:** Restart gateway if config changed:

```bash
openclaw gateway restart
```

### Messages Get Dropped / Loop Breaks Too Early

**Check loop detection config:**

```bash
grep -B2 -A2 "maxPingPongTurns" ~/.openclaw/openclaw.json
```

**Increase if needed (but don't go crazy):**

```json
{
  "session": {
    "agentToAgent": {
      "maxPingPongTurns": 10
    }
  }
}
```

**Restart gateway:**

```bash
openclaw gateway restart
```

### Agents See Each Other's Messages But Don't Respond

**Issue:** `sessions.visibility` not set or `agentToAgent.enabled` false.

**Fix:** Ensure this is in config:

```json
{
  "tools": {
    "agentToAgent": { "enabled": true },
    "sessions": { "visibility": "all" }
  }
}
```

---

## Channel Setup (Slack)

1. **Create Slack channel** (or use existing)
   - Name: `#multi-agent` (or whatever you prefer)
   - Note the channel ID (e.g., `C0AMLA1SG67`)

2. **Add bot to channel**
   - In Slack: `/invite @openclaw-bot`

3. **Update each gateway's config**
   - Edit `~/.openclaw/openclaw.json`
   - Add channel ID to `channels.slack.channels.{channelId}`
   - Set `allowBots: true, requireMention: true`

4. **Restart all gateways**

   ```bash
   openclaw gateway restart
   ```

5. **Test from Slack**
   ```
   @cora hello
   @shelly what's up
   ```

---

## Migration to Single-Gateway Setup (Future)

If you consolidate all agents to one OpenClaw instance:

1. **All agents in same `openclaw.json`:**

   ```json
   {
     "agents": {
       "list": [{ "id": "cora" }, { "id": "shelly" }, { "id": "hex" }]
     }
   }
   ```

2. **Use native `sessions_send` for silent coordination:**

   ```python
   sessions_send(
     sessionKey="agent:shelly:main",
     message="@shelly help with X?"
   )
   ```

3. **Slack channel becomes optional** (still works for human interaction, but
   agent-to-agent is backend-only)

---

## Best Practices

### DO ✅

- ✅ **Use @-mentions in Slack** — Be explicit about who you're addressing
- ✅ **Enable loop detection** — Prevent runaway conversations
- ✅ **Set `requireMention: true`** — Avoid accidental triggers
- ✅ **Use `mentionPatterns` for natural language** — Humans don't type `@cora`, just
  say "cora"
- ✅ **Test with small requests first** — Verify routing works before complex
  coordination
- ✅ **Archive channel history** — Slack stores full message context (useful for
  debugging)

### DON'T ❌

- ❌ **Don't set `allowBots: false`** — Agent-to-agent won't work
- ❌ **Don't set `requireMention: false` + `allowBots: true`** — Creates death spiral
  (agents trigger on every message)
- ❌ **Don't skip loop detection** — You'll get infinite loops
- ❌ **Don't expect `sessions_send` to work across gateways** — It only works within one
  gateway
- ❌ **Don't overload a single Slack channel** — Many agents × frequent messages = noise

---

## Related Documentation

- **OpenClaw Session Management:**
  [docs/concepts/session.md](https://github.com/anthropics/openclaw/docs/concepts/session.md)
- **Agent Discovery & Visibility:**
  [docs/tools/subagents.md](https://github.com/anthropics/openclaw/docs/tools/subagents.md)
- **Loop Detection & Coordination:**
  [docs/tools/slash-commands.md](https://github.com/anthropics/openclaw/docs/tools/slash-commands.md)
- **Fleet Boot Patterns:**
  [../research/fleet-boot-patterns.md](https://github.com/nickjs/openclaw-config/research/fleet-boot-patterns.md)

---

## Summary

- **Multi-gateway agents** use Slack @-mention routing as the communication bus
- **Configuration is simple:** Enable `allowBots`, set `requireMention`, configure
  `mentionPatterns`
- **Loop detection prevents runaway conversations** (up to `maxPingPongTurns`)
- **This is production-tested** with Cora, Shelly, and Hex
- **Future:** Consolidating to one gateway enables native `sessions_send` (silent
  backend coordination)

---

_Maintained by Cora — last updated 2026-03-21_
