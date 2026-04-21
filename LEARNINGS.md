# Project Learnings

## Patterns That Work

- **[2026-04-20] Base-repo-is-HEAD statt klassischer Replay-Migrations:** Core-Files im
  Repo reflektieren immer den aktuellen Zustand. Migrations beschreiben nur den Diff
  vom vorherigen auf den aktuellen Stand, nicht "von Null auf HEAD". Neuinstallation
  kopiert HEAD und setzt `current_migration` auf latest — kein Replay. Updates applyen
  nur Migrations mit ID > `current_migration`. Viel einfacher, keine N-schrittigen
  Replays für frische Instances. Siehe `core-migrations/README.md`.
- **[2026-04-20] Migration-Instructions als Runbook für Claude, nicht als Code:**
  `migration.md` ist Klartext-Instruction für den Admin-Agent, nicht SQL/DDL.
  Idempotenz-Check + Escalation-on-drift statt Magic-Auto-Merge. Claude kann "lies
  File X, prüfe ob Section Y existiert, füg Bullet Z ein" sauber ausführen — dafür ist
  das Modell gut, und der Runbook-Stil macht das Verhalten reviewable + stoppbar.
- **[2026-04-20] Plan-File editable während Planning:** User-Korrektur mid-Plan
  ("Migrations sind Update-Diffs, nicht Replay") konnte im Plan-File geändert werden
  bevor Code entstand. Billiger als im Code zu korrigieren. Plan-Mode mit editierbarem
  Plan-File ist der richtige Ort für architektonische Kurskorrekturen, nicht der Editor.
- **[2026-04-20] SKILL.md-Audience-Trennung vs. Script-Deployment:** Der routing-layer
  einer Skill (SKILL.md mit Triggern) definiert *für wen* sie sichtbar ist. Der
  execution-layer (Script) kann separat deployed werden. Admin-Skill gehört komplett
  auf den Admin-Rechner (SKILL.md + Script). Infra-Scripts können auf Target leben
  ohne SKILL.md — Cron-Jobs rufen sie auf, User-KI weiß nicht mal dass sie existieren.
- **[2026-04-20] Drei-Quellen-Vergleich bei Template-Rewrites:** OpenClaw native
  (kanonisch) + Clawdia live (praktisch evolviert) + aktuelles Repo-Template (TechNickAI
  opinionated) gegeneinander halten. Der beste Entwurf ist ein Merge, nicht einer davon.
  Gilt generell für Rewrites in Forks mit Upstream.
- **[2026-04-20] Plan-File vor großem Rewrite:** `tasks/NN - xxx.md` mit Key Decisions,
  Inputs, Structure, Open Points hat AGENTS.md vor Verirrungen gerettet. Große Rewrites
  brauchen einen geschriebenen Plan, nicht nur Kopf-Kontext.
- **[2026-04-20] Sektionsweises Vorgehen bei Templates:** Pro Sektion 2-3 Optionen
  präsentieren, User entscheiden, weiter. Viel effizienter als Gesamtentwürfe die dann
  komplett umgeworfen werden — aber nur bis die Design-Achsen klar sind; danach ist
  Batch-Rewrite okay.
- **[2026-04-20] OpenClaw native Prompt-Style für leere Sektionen:** Einzeiliger italic
  Fragenblock `_(What do they care about? ... Build this over time.)_` ist stärker als 5
  leere `[...]`-Bullets. Funktioniert für User.md Sektionen, die organisch wachsen.
  Nicht für strukturierte Daten (Basic Info) oder opinionated Content (SOUL.md).
- **[2026-04-20] Git-Author-Fix via Rebase auf lokale Commits:**
  `git rebase -r HEAD~N --exec "git commit --amend --reset-author --no-edit"` korrigiert
  Author auf allen N letzten Commits. Funktioniert solange nichts gepusht ist.

## Mistakes to Avoid

- **[2026-04-20] Admin-Skills gehören NICHT auf User-Instanzen:** Der frühere
  `skills/openclaw/` (Installer/Updater) und `skills/gateway-restart/` (Fleet-Op) waren
  auf jeder User-Instanz. Das vermischt Separation-of-Concerns — User darf nie Fleet
  operationen triggern. Admin-Skills leben in `~/.claude/skills/openclaw-fleet/` auf
  Christophs Rechner, Instanzen bekommen davon nichts.
- **[2026-04-20] Broken Tests sind dead weight, nicht "nochmal fixen":** `tests/`-Ordner
  war seit Starter-Set-Split broken (Tests zeigen auf `skills/*`, Skills sind in
  `skill-candidates/`). Statt flicken: komplett löschen wenn das Code-Surface weg ist.
  Dieses Repo ist Config/Docs, keine Tests nötig am Repo-Level. Skill-Candidates
  testen sich selbst (UV-scripts mit inline deps) wenn nötig.
- **[2026-04-20] Cron-Refs auf `$HOME/.openclaw-config/...` bedeuten Target braucht
  Clone:** Cron-Jobs referenzieren Files im Base-Repo-Clone direkt (z.B.
  `devops/health-check.md`). Deploy-Flow muss den Clone auf Target pflegen — nicht nur
  auf dem Admin-Agent pullen. Sonst sieht die Cron-Maschine upstream-Updates nie.
- **[2026-04-20] Templates nicht mit Clawdia-Spezifika überfrachten:** Clawdias SOUL.md
  hat "Telegram-Benachrichtigung bei GC-Arbeit" — das gehört NICHT in ein Template.
  Template-Philosophie: Opinionated Defaults wo universell, leere Prompts wo
  user-specific. Unterscheiden zwischen "Persönlichkeit als Startpunkt" (SOUL.md
  vertragt Clawdia-Defaults) und "User-Fakten" (USER.md braucht leere Slots).
- **[2026-04-20] `git rm` staged SOFORT:** Eine `git rm file.md` staged die Löschung
  ohne separaten `git add`. Wenn dann `git commit` mit nur einzelnen Files gemeint ist,
  rutscht die Löschung mit rein. Lösung: `git reset --soft HEAD~1` + selektiv neu
  stagen.
- **[2026-04-20] Skills vs. Plugins nicht vermischen:** Skills = Python UV Single-File
  CLIs, On-Demand. Plugins = TypeScript Module, Gateway-geladen, Hooks + Tool-Injection.
  Gehören unter `plugins/`, nicht `skills/`. Convention-break kostet Klarheit.
- **[2026-04-20] Cortex-Memory-Folders als Default-Install sind Residue:** Der
  `openclaw` Skill erstellt `memory/{people,projects,topics,decisions}` — das war
  Cortex-Schema. Nach Szenario A (OpenClaw native) sind diese Ordner nicht nötig. Bei
  Installer-Update raus.

## Domain Knowledge

- **[2026-04-20] Deploy-Transport-Optionen (Entscheidung offen):** Public-Repo-git-clone
  braucht keinen User auf Target (HTTPS, anonym), aber `git` + Internet. Rsync vom
  Admin-Agent via existierendem SSH-User: kein git, kein Internet auf Target, funktioniert
  auch bei Private-Repo ohne Credential-Verteilung. Tarball-via-scp: dazwischen, commit-
  genau via `git archive`. Trade-off-Tabelle in Chat-Historie der 2026-04-20 Session.
  Entscheidung noch offen, `docs/install.md` aktuell auf git clone.
- **[2026-04-20] Drei Locations pro OpenClaw-Target:** `~/.openclaw-config/` = Clone der
  Base-Repo (Source of Truth auf Maschine). `~/openclaw/` = Workspace (Templates
  gerendert, Workflow-Kopien). `~/.openclaw/` = Gateway-State (openclaw.json,
  migration-state.json, plugins/, locks/, logs/). Doppelte Präsenz (Clone + Workspace)
  nötig, weil Cron-Jobs direkt auf Clone zeigen und User-Customization in Workspace
  erlaubt sein soll.
- **[2026-04-20] OpenClaw native Memory-System:** `openclaw memory` CLI hat
  `search/index/promote/rem-harness/rem-backfill` — macht Vector-Search (sqlite-vec,
  1536 dims), FTS, Short-Term-Recall-Tracking, Auto-Promotion in MEMORY.md,
  REM/DREAMS-Consolidation. Tracking in
  `~/.openclaw/workspace/memory/.dreams/short-term-recall.json`. Cortex war nur
  zusätzliche Entity-Page-Pflege (people/projects/topics/decisions/) — mit OpenClaw
  native + GC redundant.
- **[2026-04-20] ~~Install vs. Update sind zwei Mechanismen~~** ~~`skills/openclaw/SKILL.md`
  = interaktiver AI-geführter 12-Schritte-Install. `skills/openclaw/openclaw` (Bash) =
  Sync/Update CLI. Dieses Repo ist eine Config-Schicht über einem bereits laufenden
  OpenClaw-Gateway — kein OpenClaw-Installer selbst.~~ **(Superseded 2026-04-20 durch
  Base-Repo + Admin-Agent Refactor:** Install-Flow in `docs/install.md`, Update-Flow in
  `docs/update.md`, beide aufgerufen vom externen Admin-Agent via SSH. Skills/openclaw
  entfernt.)
- **[2026-04-20] User-Owned vs Upstream-Owned:** SOUL.md, USER.md, TOOLS.md, IDENTITY.md
  werden beim Install kopiert und nie überschrieben. AGENTS.md, HEARTBEAT.md werden bei
  Updates aktualisiert (via Admin-Agent + core-migrations). In Workflows:
  `rules.md, agent_notes.md, preferences.md, processed.md, logs/` sind user-owned, alles
  andere upstream.
- **[2026-04-20] GC-Plugin Architektur:** Lebt in
  `~/repositories/groundcontrol/openclaw-plugin/` als TypeScript-Modul mit
  `openclaw.plugin.json` Manifest, `index.ts` Entry, `src/tools.ts` (20+
  Tool-Definitionen), `scripts/gc-worker.sh` (Cron Worker). Wird vom Gateway beim Start
  geladen. ~980 Zeilen Code.
- **[2026-04-20] `boot-md` Hook:** Wenn in `~/.openclaw/openclaw.json` unter
  `hooks.internal.entries` aktiviert, lädt OpenClaw `BOOT.md` beim Agent-Startup in
  Context. Clawdia hat die `internal` Sektion NICHT — BOOT.md ist optional. Für
  Entrepreneur-Config nicht nötig, AGENTS.md "Every Session" reicht.
- **[2026-04-20] Fleet-URLs vs. Produkt-URLs:** CLAUDE.md-Regel "no hostnames im public
  repo" gilt für Fleet-Spezifika (z.B. `mac-mini-von-christoph.tail8db5a4.ts.net`).
  Produkt-URLs wie `groundcontrol.makerslab.ai` sind okay — das ist öffentlicher Dienst,
  nicht fleet-internal.

## Tool & Environment

- **[2026-04-20] Template-Placeholders werden vom Admin-Agent literal ersetzt (kein
  Runtime-Engine):** `{{USER_NAME}}`, `{{ASSISTANT_NAME}}`, `{{TIMEZONE}}`,
  `{{ASSISTANT_EMAIL}}`, `{{ASSISTANT_ROLE}}`. Plus Gateway-Config-Platzhalter
  (`{{TELEGRAM_*}}`) in `defaults/openclaw.json.template`. Alle Placeholders in
  `defaults/README.md` dokumentiert. Wenn ein Template neue Platzhalter einführt, muss
  `defaults/README.md` + `docs/install.md` Step 3 nachgezogen werden.
- **[2026-04-20] `.gitignore` für `tasks/` Ordner ist Pflicht:** Public repo + lokale
  Plan-Files = PII-Leak-Risiko. Pattern `tasks/` in `.gitignore` gleich beim ersten
  `tasks/01 - xxx.md` setzen.
- **[2026-04-20] `gog gmail` ist natives OpenClaw-Plugin:** Nicht in `skills/`.
  Installed als Teil des Gateway-Setups. Agent kriegt vom Provider Email-Adresse → nutzt
  via gog gmail. Im Template Template-Command:
  `gog gmail search "..." --account {{ASSISTANT_EMAIL}}`.
- **[2026-04-20] `skill-candidates/` ist eine bewusst nicht-installierbare
  Archivschicht:** Admin-Agent installiert nur, was in `skills/` liegt (aktuell leer).
  Skills in `skill-candidates/` müssen manuell per `cp` nach `skills/` befördert werden.
  Hält den Default-Install-Pfad sauber ohne Skills zu verlieren.

## Consolidated Principles

<!-- Updated during consolidation. Under 50 entries — skipping for now. -->
