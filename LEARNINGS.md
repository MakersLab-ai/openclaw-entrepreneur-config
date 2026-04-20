# Project Learnings

## Patterns That Work

- **[2026-04-20] Drei-Quellen-Vergleich bei Template-Rewrites:** OpenClaw native (kanonisch) + Clawdia live (praktisch evolviert) + aktuelles Repo-Template (TechNickAI opinionated) gegeneinander halten. Der beste Entwurf ist ein Merge, nicht einer davon. Gilt generell für Rewrites in Forks mit Upstream.
- **[2026-04-20] Plan-File vor großem Rewrite:** `tasks/NN - xxx.md` mit Key Decisions, Inputs, Structure, Open Points hat AGENTS.md vor Verirrungen gerettet. Große Rewrites brauchen einen geschriebenen Plan, nicht nur Kopf-Kontext.
- **[2026-04-20] Sektionsweises Vorgehen bei Templates:** Pro Sektion 2-3 Optionen präsentieren, User entscheiden, weiter. Viel effizienter als Gesamtentwürfe die dann komplett umgeworfen werden — aber nur bis die Design-Achsen klar sind; danach ist Batch-Rewrite okay.
- **[2026-04-20] OpenClaw native Prompt-Style für leere Sektionen:** Einzeiliger italic Fragenblock `_(What do they care about? ... Build this over time.)_` ist stärker als 5 leere `[...]`-Bullets. Funktioniert für User.md Sektionen, die organisch wachsen. Nicht für strukturierte Daten (Basic Info) oder opinionated Content (SOUL.md).
- **[2026-04-20] Git-Author-Fix via Rebase auf lokale Commits:** `git rebase -r HEAD~N --exec "git commit --amend --reset-author --no-edit"` korrigiert Author auf allen N letzten Commits. Funktioniert solange nichts gepusht ist.

## Mistakes to Avoid

- **[2026-04-20] Templates nicht mit Clawdia-Spezifika überfrachten:** Clawdias SOUL.md hat "Telegram-Benachrichtigung bei GC-Arbeit" — das gehört NICHT in ein Template. Template-Philosophie: Opinionated Defaults wo universell, leere Prompts wo user-specific. Unterscheiden zwischen "Persönlichkeit als Startpunkt" (SOUL.md vertragt Clawdia-Defaults) und "User-Fakten" (USER.md braucht leere Slots).
- **[2026-04-20] `git rm` staged SOFORT:** Eine `git rm file.md` staged die Löschung ohne separaten `git add`. Wenn dann `git commit` mit nur einzelnen Files gemeint ist, rutscht die Löschung mit rein. Lösung: `git reset --soft HEAD~1` + selektiv neu stagen.
- **[2026-04-20] Skills vs. Plugins nicht vermischen:** Skills = Python UV Single-File CLIs, On-Demand. Plugins = TypeScript Module, Gateway-geladen, Hooks + Tool-Injection. Gehören unter `plugins/`, nicht `skills/`. Convention-break kostet Klarheit.
- **[2026-04-20] Cortex-Memory-Folders als Default-Install sind Residue:** Der `openclaw` Skill erstellt `memory/{people,projects,topics,decisions}` — das war Cortex-Schema. Nach Szenario A (OpenClaw native) sind diese Ordner nicht nötig. Bei Installer-Update raus.

## Domain Knowledge

- **[2026-04-20] OpenClaw native Memory-System:** `openclaw memory` CLI hat `search/index/promote/rem-harness/rem-backfill` — macht Vector-Search (sqlite-vec, 1536 dims), FTS, Short-Term-Recall-Tracking, Auto-Promotion in MEMORY.md, REM/DREAMS-Consolidation. Tracking in `~/.openclaw/workspace/memory/.dreams/short-term-recall.json`. Cortex war nur zusätzliche Entity-Page-Pflege (people/projects/topics/decisions/) — mit OpenClaw native + GC redundant.
- **[2026-04-20] Install vs. Update sind zwei Mechanismen:** `skills/openclaw/SKILL.md` = interaktiver AI-geführter 12-Schritte-Install. `skills/openclaw/openclaw` (Bash) = Sync/Update CLI. Dieses Repo ist eine Config-Schicht über einem bereits laufenden OpenClaw-Gateway — kein OpenClaw-Installer selbst.
- **[2026-04-20] User-Owned vs Upstream-Owned:** SOUL.md, USER.md, TOOLS.md, IDENTITY.md werden beim Install kopiert und nie überschrieben. AGENTS.md, HEARTBEAT.md werden bei `openclaw sync` aktualisiert. In Workflows: `rules.md, agent_notes.md, preferences.md, processed.md, logs/` sind user-owned, alles andere upstream.
- **[2026-04-20] GC-Plugin Architektur:** Lebt in `~/repositories/groundcontrol/openclaw-plugin/` als TypeScript-Modul mit `openclaw.plugin.json` Manifest, `index.ts` Entry, `src/tools.ts` (20+ Tool-Definitionen), `scripts/gc-worker.sh` (Cron Worker). Wird vom Gateway beim Start geladen. ~980 Zeilen Code.
- **[2026-04-20] `boot-md` Hook:** Wenn in `~/.openclaw/openclaw.json` unter `hooks.internal.entries` aktiviert, lädt OpenClaw `BOOT.md` beim Agent-Startup in Context. Clawdia hat die `internal` Sektion NICHT — BOOT.md ist optional. Für Entrepreneur-Config nicht nötig, AGENTS.md "Every Session" reicht.
- **[2026-04-20] Fleet-URLs vs. Produkt-URLs:** CLAUDE.md-Regel "no hostnames im public repo" gilt für Fleet-Spezifika (z.B. `mac-mini-von-christoph.tail8db5a4.ts.net`). Produkt-URLs wie `groundcontrol.makerslab.ai` sind okay — das ist öffentlicher Dienst, nicht fleet-internal.

## Tool & Environment

- **[2026-04-20] Template-Placeholders werden vom Installer literal ersetzt (kein Runtime-Engine):** `{{USER_NAME}}`, `{{ASSISTANT_NAME}}`, `{{TIMEZONE}}`, plus nach dieser Session neu: `{{ASSISTANT_EMAIL}}`, `{{ASSISTANT_ROLE}}`. Raus: `{{PRIORITY_1}}`, `{{PRIORITY_2}}` (Priorities sind jetzt agent-maintained Liste). Wenn Template neue Platzhalter einführt, muss der Installer-Flow in `skills/openclaw/SKILL.md` Schritt 7 nachziehen.
- **[2026-04-20] `.gitignore` für `tasks/` Ordner ist Pflicht:** Public repo + lokale Plan-Files = PII-Leak-Risiko. Pattern `tasks/` in `.gitignore` gleich beim ersten `tasks/01 - xxx.md` setzen.
- **[2026-04-20] `gog gmail` ist natives OpenClaw-Plugin:** Nicht in `skills/`. Installed als Teil des Gateway-Setups. Agent kriegt vom Provider Email-Adresse → nutzt via gog gmail. Im Template Template-Command: `gog gmail search "..." --account {{ASSISTANT_EMAIL}}`.
- **[2026-04-20] `skill-candidates/` ist eine bewusst nicht-installierbare Archivschicht:** `openclaw add-skill` liest nur aus `skills/`. Skills in `skill-candidates/` müssen manuell per `cp` aktiviert werden. Hält den Default-Install-Pfad sauber ohne Skills zu verlieren.

## Consolidated Principles

<!-- Updated during consolidation. Under 50 entries — skipping for now. -->
