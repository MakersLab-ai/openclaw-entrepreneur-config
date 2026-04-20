# Skill Candidates

Skills in diesem Ordner sind **nicht Teil des Starter-Sets** und werden vom
`openclaw`-Installer **nicht automatisch installiert**.

## Wann ist ein Skill hier gelandet?

Das Entrepreneur-Starter-Set setzt bewusst auf einen minimalen Fußabdruck: Memory
(OpenClaw native), Task-System (GroundControl), und ein paar Basics. Alles darüber
hinaus ist optional und integrations-spezifisch — also nicht für jeden Kunden
relevant.

Skills hier sind entweder:

- **Hardware-abhängig** (z. B. Limitless Pendant)
- **Service-spezifisch** (CRM-Tool, Task-Tool, Meeting-Service)
- **Meta-/Advanced-Tooling**, das erst im laufenden Betrieb Wert hat (Workflow-Builder,
  Smart-Delegation)

## Übersicht

| Skill | Kategorie | Wofür |
|-------|-----------|-------|
| `agentmail` | Email | AgentMail-hosted Inboxes (agentmail.to) — Alternative zu gog gmail für webhook-basierte oder Multi-Inbox-Setups |
| `asana` | Task-Management | Asana-Projekte und Tasks |
| `todoist` | Task-Management | Todoist |
| `followupboss` | CRM | Follow Up Boss |
| `limitless` | Meetings | Limitless Pendant Lifelogs |
| `fireflies` | Meetings | Fireflies.ai Transcripts |
| `fathom` | Meetings | Fathom Meeting Recordings |
| `parallel` | Research | Parallel.ai Web Search |
| `tgcli` | Kommunikation | Telegram persönlicher Account |
| `tgcli-topics` | Kommunikation | Telegram Forum-Topics |
| `quo` | Business Phone | Quo (ex-OpenPhone) Calls/SMS |
| `vapi-calls` | Voice | Vapi Outbound-Anrufe |
| `workflow-builder` | Meta | Autonome Workflows (Stewards) bauen |
| `smart-delegation` | Meta | Task-Routing zu Deep-Reasoning / Grok |

## Aktivieren

Die Skills sind **nicht** über `openclaw add-skill` installierbar (bewusst — das hält
den Starter-Pfad sauber). Zum Aktivieren manuell kopieren:

```bash
cp -r skill-candidates/<name> skills/<name>
```

Danach ganz normal im Workspace installieren, API-Keys setzen, fertig.

## Warum nicht einfach löschen?

Weil der Kontext (Triggers, API-Setup, Verwendungshinweise) wertvoll ist, sobald ein
Kunde eines dieser Tools einführt. Hier liegen sie griffbereit, ohne den Default-Pfad
zu belasten.
