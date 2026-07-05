---
name: obsidian-second-brain
description: >
  Operate any Obsidian vault as a living, self-rewriting second brain (an evolution
  of Karpathy's LLM Wiki pattern: sources rewrite existing pages, contradictions
  reconcile automatically, scheduled agents maintain the vault while you sleep).
  Use this skill whenever
  the user asks Claude to read, write, update, search, or manage their Obsidian
  vault - including saving notes from conversation, creating daily entries, updating
  kanban boards, logging dev work, managing people notes, capturing decisions,
  tracking deals, or maintaining any vault structure. Also triggers when the user
  wants to bootstrap a new vault from scratch, run a vault health check, or drop
  a _CLAUDE.md into their vault so all Claude surfaces share the same operating rules.
  Includes a research toolkit (7 commands: /x-read, /x-pulse, /research, /research-deep,
  /notebooklm, /youtube, /podcast) for AI-powered research via Grok, Perplexity, NotebookLM,
  YouTube, and podcast feeds - findings save
  to the vault automatically following the AI-first vault rule. Use proactively whenever
  the conversation produces information worth preserving (decisions, people met, projects
  started, tasks completed, lessons learned, research findings).
---

# Obsidian Second Brain

> Claude operates your Obsidian vault as a self-rewriting knowledge base. An evolution of [Karpathy's LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f): sources rewrite existing pages instead of just appending, contradictions reconcile automatically, and scheduled agents maintain the vault while you sleep.
> Everything worth remembering gets saved. Every update propagates everywhere it belongs.

---

## Quick Start

### 0. Choose vault access method (in order of preference)

Try these methods in order. Use the first one available:

**Method 0 - SessionStart hook (if configured):**
If `hooks/load_vault_context.py` is wired as a SessionStart hook in `~/.claude/settings.json`, `_CLAUDE.md` is injected into context automatically at session start. Skip step 1 below.
To wire it: `bash scripts/setup.sh "/path/to/vault"` or run `/obsidian-setup`.

**Method A - Direct filesystem (default, always works):**
Use standard file tools (Read, Write, Edit, Glob) against the vault path. The vault is plain markdown, so every operation in this skill works this way with no setup. This is the normal path in Claude Code - the commands below use these tools directly.

**Method B - MCP server (optional, mainly for non-Claude-Code clients):**
This repo ships its own MCP server at `integrations/obsidian-mcp-server/` that exposes the vault as tools (`obsidian_search`, `obsidian_read_note`, `obsidian_save_note`, `obsidian_capture`, plus curator tools). It exists so other MCP clients - Hermes Agent, Claude Desktop, Cursor - can use the vault as a knowledge layer; in Claude Code itself, Method A is simpler and preferred. If those `obsidian_*` tools happen to be available in your client, you may use them instead of raw file tools. Setup lives in `integrations/obsidian-mcp-server/README.md` (it is `uv run --with mcp python .../server.py` with `OBSIDIAN_VAULT_PATH` set, not an `npx` package).

### 1. First time in a vault → read `_CLAUDE.md`

Before doing anything in a vault, check if `_CLAUDE.md` exists at the vault root and read it:

```
Read <vault>/_CLAUDE.md
```

If it exists: follow its rules exactly - they override the defaults in this skill. Where `_CLAUDE.md` is silent, fall back to the defaults below.
If it doesn't exist: use the defaults in this skill, then offer to create one.

If the SessionStart hook is active, `_CLAUDE.md` is already in context - skip this step.

### 2. First time with a new user → run discovery

```
Glob <vault>/**/*.md
```

Scan the structure to understand: folder names, template locations, naming conventions, frontmatter patterns. Then read 2-3 existing notes to calibrate writing style before creating anything new.

### 3. Bootstrap a new vault

If the user has no vault yet, run:
```bash
# One-line install + bootstrap (asks 3 questions: vault path, your name, preset)
curl -sL https://raw.githubusercontent.com/eugeniughelbur/obsidian-second-brain/main/scripts/quick-install.sh | bash

# Or manual:
python scripts/bootstrap_vault.py --path ~/path/to/vault --name "Your Name"

# With a preset:
python scripts/bootstrap_vault.py --path ~/my-vault --name "Your Name" --preset executive
python scripts/bootstrap_vault.py --path ~/my-vault --name "Your Name" --preset builder
python scripts/bootstrap_vault.py --path ~/my-vault --name "Your Name" --preset creator
python scripts/bootstrap_vault.py --path ~/my-vault --name "Your Name" --preset researcher

# With assistant mode (maintaining vault for someone else):
python scripts/bootstrap_vault.py --path ~/my-vault --name "Your Name" --mode assistant --subject "Boss Name"
```

Then just point the skill at the new vault path (Method A above). If you use the optional bundled MCP server, set `OBSIDIAN_VAULT_PATH` to the new path and restart the client.

**Presets** customize the vault for different use cases:
- **`executive`** - Decisions, people, meetings, strategic planning. Kanban: OKRs, Quarterly, Weekly.
- **`builder`** - Projects, dev logs, architecture decisions, debugging. Kanban: Backlog, Sprint, Done.
- **`creator`** - Content calendar, ideas pipeline, audience notes, publishing. Kanban: Ideas, Drafts, Published.
- **`researcher`** - Sources, literature notes, hypotheses, methodology. Kanban: Reading, Processing, Synthesized.

Default (no preset) gives a general-purpose vault. All presets use wiki-style by default.

**Assistant mode** creates a `_CLAUDE.md` configured for operating a vault on behalf of someone else. See `references/claude-md-assistant-template.md`.

See `references/vault-schema.md` for full structural details.

---

## Core Operating Principles

### AI-first vault rule (applies to every note)
The vault is designed for **future-Claude** to read and reason over, not for human review. Every note Claude writes - across all 44 commands - must follow `references/ai-first-rules.md`:

1. **Self-contained context** - each note explains itself; don't rely on backlinks alone
2. **"For future Claude" preamble** - 2-3 sentence summary so Claude can decide relevance in 10 seconds
3. **Rich, consistent frontmatter** - `type`, `date`, `tags`, `ai-first: true`, plus type-specific fields (see `ai-first-rules.md` for schemas per note type)
4. **Recency markers per claim** - "Mem0 raised $24M (as of 2026-04, mem0.ai)" so future-Claude knows what to verify
5. **Sources preserved verbatim** - every external claim has its source URL inline
6. **Cross-links mandatory** - every person/project/idea/decision uses `[[wikilinks]]`
7. **Confidence levels** - `stated | high | medium | speculation` where applicable

This rule lives in `_CLAUDE.md` Section 0 of every vault using this skill, and in `references/ai-first-rules.md` (the canonical specification with frontmatter schemas + preamble templates per note type).

### Never create in isolation
Every write operation must ask: *where else does this belong?*

| You create/update... | Also update... |
|---|---|
| A new project note | Kanban board (add to Backlog), today's daily note (link it) |
| A task completed | Kanban board (move to Done), project note (log it), daily note |
| A person note | Daily note (mention interaction), People index if it exists |
| A dev log | Daily note (link it), project note (Recent Activity) |
| A deal update | Side Biz / Deals kanban, Dashboard totals |
| A decision made | Project note (Key Decisions), daily note |
| A mention/shoutout | Mentions Log, person's note, daily note |
| A hook, contrarian angle, or content idea | `social-media/ideas.md` (if folder exists) |
| A specific reusable number or stat | `social-media/data-points.md` (if folder exists) |
| An external post that performed well + why | `social-media/swipe-file.md` (if folder exists) |
| Research findings worth keeping | `social-media/research/YYYY-MM-DD — topic.md` (if folder exists) |
| Any vault write | operation log (`Logs/YYYY-MM-DD.md` if `Logs/` exists, else `log.md`), `index.md` (update if new note created) |

Always propagate. Never create a single orphaned note.

### Bi-temporal facts - never overwrite, always append
When a fact changes (role, company, status, location, tool), NEVER delete the old value. Add a new entry to the `timeline:` frontmatter array with both event time AND transaction time:

```yaml
timeline:
  - fact: "CTO at Single Grain"
    from: 2024-01-01            # event time: when it was true
    until: 2026-04-07
    learned: 2026-02-23         # transaction time: when the vault learned it
    source: "[[2026-02-23]]"    # where from
  - fact: "Architect at Single Grain"
    from: 2026-04-07
    until: present
    learned: 2026-04-07
    source: "[[2026-04-07]]"
```

Top-level fields (`role:`, `status:`, `company:`) always reflect the CURRENT state. The `timeline:` preserves the full history with provenance.

This enables:
- Historical queries ("who was my manager in February?")
- Reflective thinking ("you believed X on Tuesday, then ingested Y on Wednesday and shifted to Z")
- Smart reconciliation (different facts at different times = not a contradiction)
- Full audit trail (when did the vault learn each fact, from what source?)

### CRITICAL_FACTS.md - always loaded
A tiny file (~120 tokens) loaded alongside `SOUL.md` at L0 in every session. Contains facts needed in every conversation:
- Timezone
- Current manager
- Current location
- Current company and role
- Any other fact that's true RIGHT NOW and relevant to every interaction

Update this file whenever a critical fact changes. Keep it under 150 tokens.

### Raw is immutable
In wiki-style vaults, the `raw/` folder contains original sources (articles, transcripts, PDFs). Claude reads these but NEVER modifies them. They are the source of truth. If a wiki page gets corrupted, re-derive it from the raw source. When ingesting, always save the original to `raw/` and the derived pages to `wiki/`.

### Maintain `index.md` and `log.md`
Two structural files that keep the vault navigable and auditable:

- **`index.md`** - A catalog of all vault pages organized by category. Claude reads this FIRST when navigating the vault instead of searching - faster and cheaper on tokens. Update it whenever a new note is created or deleted. Format: `- [[Note Name]] — brief description` grouped under folder headings.

- **`log.md`** - An append-only chronological log of every vault operation. Every save, ingest, health check, and structural change gets a timestamped entry. Never delete or rewrite entries - only append. Format: `## [YYYY-MM-DD] action | Description`

### Per-day operation logs (modernized vaults)
Vaults initialized with `/obsidian-init` (v0.9+) use a split log structure instead of a monolithic `log.md`:

- **`Logs/YYYY-MM-DD.md`** - one file per day, append-only. Format: `**HH:MM** - action | description`
- **`log.md` at vault root** - pointer file only. Never write entries here; it explains the per-day structure and ships the entry template.

To migrate an existing monolithic `log.md`: run `python scripts/migrate_log.py --vault <path>`.
To refresh the stats block in `index.md` after bulk writes: run `python scripts/vault_stats.py --vault <path>`.

When writing operation log entries, check whether the vault uses the old (`log.md`) or new (`Logs/YYYY-MM-DD.md`) structure and write to the correct location.

### The vault is a living system
The vault is not a filing cabinet. It is a living knowledge base that rewrites itself with every input. When new information enters:
- Existing pages get REWRITTEN with new context, not just appended to
- Contradictions between old and new claims get resolved or explicitly documented
- New patterns across multiple sources trigger automatic synthesis pages
- Stale claims get replaced with current information, with history preserved

The vault after an ingest should be DIFFERENT - not just bigger. If pages that existed before aren't smarter, more connected, and more current, the ingest wasn't deep enough.

### Two-Output Rule
Every interaction that produces insight must generate two outputs:
1. **The answer** - what the user sees in the conversation
2. **A vault update** - the insight filed back into the relevant note(s)

This applies to all thinking tools and any query where Claude synthesizes information from the vault.

### Synthesis Hook
When Claude notices a pattern during any operation (ingest, query, challenge, emerge), it should automatically create a synthesis page in `wiki/concepts/`. Patterns include:
- The same concept appearing in 3+ unrelated sources
- A claim being reinforced by multiple independent sources
- A trend emerging across time-sequenced notes
- Two entities sharing unexpected connections

Synthesis pages are the vault thinking for itself - connecting dots the user hasn't connected yet.

### Reconciliation
The vault should never contain two pages that disagree without knowing they disagree. When contradictions are found (during ingest, health checks, or queries), either:
- Resolve them: rewrite the outdated page, preserve history
- Document them: create an explicit conflict page marked as an open question

Use `/obsidian-reconcile` for vault-wide truth maintenance.

### Proactive save reminders
Unsaved conversations are lost knowledge. Claude should proactively remind the user to save:
- After 10+ exchanges: suggest "Want me to run /obsidian-save before we continue?"
- When the user signals wrap-up (e.g., "ok", "thanks", "done", "bye", "that's it"): suggest "Before you go - want me to /obsidian-save this conversation?"
- When a logical work block completes (feature shipped, decision made, problem solved): suggest saving
- Never skip the reminder. This is especially critical on Claude Desktop where there's no background agent.

### Search before creating
Before creating any new note, search for an existing one:
```
search(query="keyword from title")
```
Duplicate notes are vault rot. Merge or update instead of creating new.

### Never claim absence from memory, never fabricate
Two failure modes corrupt the vault silently:
- **False absence (most common):** never say "no note exists" or create a note on the assumption none exists without searching exhaustively first - by every plausible name, alias, and folder, listing and grepping, not from memory. When in doubt, over-include and label the uncertainty.
- **Fabrication:** never invent facts, entities, rates, dates, or relationships that were not actually stated. Mark unknowns as `TBD`; an empty section is correct when nothing was said. External claims carry a source URL + recency marker; inferences carry a confidence level.

See the anti-fabrication and search-completeness hard rules in `references/ai-first-rules.md`.

### Match the vault's voice
Read existing notes in the same folder before writing new ones.
Match: frontmatter schema, heading style, list formatting, tone, emoji usage (or lack of it).
Never introduce new conventions - extend what's already there.

### Frontmatter is mandatory
Every note gets frontmatter. At minimum:
```yaml
---
date: 2026-03-24
tags:
  - <note-type>
---
```
See `references/vault-schema.md` for full frontmatter specs by note type.

---

## Write Rules

See `references/write-rules.md` for the complete guide. Summary:

- **Links**: Use `[[Note Name]]` for internal links. Always link to people, projects, and jobs mentioned in a note.
- **Dates**: ISO format (`YYYY-MM-DD`) in frontmatter. Human format (`March 24`) in body text.
- **Naming**: `YYYY-MM-DD — Title.md` for dated notes. `Title.md` for evergreen notes. No special characters except `—` (em dash).
- **Status values**: `active` / `planning` / `completed` / `archived` / `on-hold` for projects. `in-progress` / `done` / `waiting` for tasks.
- **Kanban**: Items follow the format `- [ ] 🔴 **Title** · @{YYYY-MM-DD}\n\tDescription [[Link]]`

---

## The `_CLAUDE.md` File

This is the most important concept in this skill.

`_CLAUDE.md` lives at the vault root and persists Claude's operating rules across every session and every surface (Claude Desktop, Claude Code, VS Code, terminal). Without it, Claude has to re-learn your vault conventions every conversation.

**Precedence rule:** `_CLAUDE.md` wins on all vault-specific rules (folder names, naming conventions, frontmatter fields, auto-save behavior, private folders). The defaults in this skill file apply only where `_CLAUDE.md` is silent. Never let skill defaults override an explicit `_CLAUDE.md` rule.

**What it contains:**
- Your vault's folder map and what each folder is for
- Frontmatter schemas for your specific note types
- Naming conventions you use
- What to auto-save vs. what to ask first
- People and projects that need special handling
- Links to key files (boards, dashboard, templates)

To generate a `_CLAUDE.md` for an existing vault, run vault discovery then use the template in `references/claude-md-template.md`.

To install it: write the file to the vault root. Every Claude session that starts in that vault should read it first.

---

## Common Operations

### Save info from conversation
When a conversation produces something vault-worthy:
1. Identify the note type (decision → project note, person met → People/, task → board + Tasks/, etc.)
2. Check if a relevant note already exists
3. Write or update - always frontmatter-first
4. Propagate to boards, daily note, linked notes

### Create today's daily note
```
date = today in YYYY-MM-DD format
path = Daily/{date}.md
```
Read `Templates/Daily Note.md`, fill in the date fields, create the file.
Then scan recent conversation for anything worth logging in today's sections.

### Log a dev session
Read `Templates/Dev Log.md`. Fill: date, project name, what was worked on, problems solved, decisions made, next steps.
Save to `Dev Logs/YYYY-MM-DD — Project Name.md`.
Link from project note's Recent Activity section and today's daily note.

### Update a kanban board
Boards use the `kanban-plugin: board` frontmatter.
Columns are `## Column Name` headers.
Items are `- [ ] **Title** · @{due-date}\n\tDescription [[Links]]`
Completed items move to the `## ✅ Done` column with a strikethrough: `- [x] ~~**Title**~~ ✅ Date`

### Run vault health check
```bash
python scripts/vault_health.py --path ~/path/to/vault
```
Reports: duplicate notes, orphaned files (no incoming links), stale tasks (overdue), empty folders, broken links, notes missing frontmatter.

Proactively suggest running this when the user says the vault feels messy, notes are hard to find, they mention duplicates, or they haven't mentioned a health check in a long time. Offer: *"Want me to run a vault health check?"*

---

## Commands

These slash commands can be used in any Claude surface. Each one is smart - it reads context, searches before writing, and propagates everywhere changes belong.

**Name matching:** If a name argument has a typo or is approximate, search the vault for the closest match, show what was found, and confirm with the user before proceeding. Never silently create a note with a misspelled name.

---

### `/obsidian-save`

**The master save command.** Reads the entire conversation and extracts everything worth preserving.

Full procedure: `commands/obsidian-save.md`

---

### `/obsidian-daily`

**Creates or updates today's daily note.**

Full procedure: `commands/obsidian-daily.md`

---

### `/obsidian-calendar <mode>`

**One calendar command with four modes.** Claude Code only (needs the Google Calendar MCP). The first word selects the mode; a bare range word defaults to `agenda`; no argument at all defaults to `agenda today`.

Full procedure: `commands/obsidian-calendar.md`

---

### `/obsidian-recurring`

**Tracks a recurring obligation (payment, filing, ops) with a cadence and a computed next-due date.**

Full procedure: `commands/obsidian-recurring.md`

---

### `/obsidian-log`

**Logs a work or dev session to the vault.**

Full procedure: `commands/obsidian-log.md`

---

### `/obsidian-task [description]`

**Adds a task to the vault and the right kanban board.**

Full procedure: `commands/obsidian-task.md`

---

### `/obsidian-person [name]`

**Creates or updates a person note.**

Full procedure: `commands/obsidian-person.md`

---

### `/obsidian-capture [optional: idea text]`

**Quick idea capture with zero friction.**

Full procedure: `commands/obsidian-capture.md`

---

### `/obsidian-catchup [today|week|all]`

**Process what the Telegram journal bot captured on the go.** The laptop-side companion to the `integrations/telegram-journal/` bot, which captures voice/text/image/PDF/link from your phone and appends each to a `catchup.md` queue in the vault.

Full procedure: `commands/obsidian-catchup.md`

---

### `/obsidian-find [query]`

**Smart vault search.**

Full procedure: `commands/obsidian-find.md`

---

### `/obsidian-recap [today|week|month]`

**Summarizes a time period from the vault.**

Full procedure: `commands/obsidian-recap.md`

---

### `/obsidian-review`

**Generates a structured weekly or monthly review note.**

Full procedure: `commands/obsidian-review.md`

---

### `/obsidian-board [optional: board name]`

**Shows or updates a kanban board.**

Full procedure: `commands/obsidian-board.md`

---

### `/obsidian-board-hygiene [optional: board name]`

**Bulk-triages a kanban board whose columns have gone stale.** Where `/obsidian-board` only flags overdue items, this clears them.

Full procedure: `commands/obsidian-board-hygiene.md`

---

### `/obsidian-project [name]`

**Creates or updates a project note.**

Full procedure: `commands/obsidian-project.md`

---

### `/obsidian-projects [optional: project name]`

**Live status overview across all tracked projects.**

Full procedure: `commands/obsidian-projects.md`

---

### `/obsidian-health`

**Runs a vault health check and summarizes findings.**

Full procedure: `commands/obsidian-health.md`

---

### `/obsidian-retrieval-eval [optional: N to generate | report]`

**Measures how well vault search actually finds the right note - so improving retrieval is a number, not a hunch.**

Full procedure: `commands/obsidian-retrieval-eval.md`

---

### `/obsidian-reconcile`

**Finds and resolves contradictions across the vault.**

Full procedure: `commands/obsidian-reconcile.md`

---

### `/obsidian-synthesize`

**Automatic synthesis - the vault thinks for itself.**

Full procedure: `commands/obsidian-synthesize.md`

---

### `/obsidian-export`

**Export a clean snapshot any agent or tool can consume.**

Full procedure: `commands/obsidian-export.md`

---

### `/obsidian-init`

**Bootstraps `_CLAUDE.md` for the vault - the operating manual.**

Full procedure: `commands/obsidian-init.md`

---

### `/obsidian-architect`

**Scans a codebase and writes a maintained set of architecture notes into the vault - overview, per-module notes, key decisions. Re-runnable.**

Full procedure: `commands/obsidian-architect.md`

---

### `/obsidian-visualize [scope]`

**Generates a JSON Canvas map of the vault's knowledge graph, openable in Obsidian's native canvas viewer.**

Full procedure: `commands/obsidian-visualize.md`

---

### `/create-command [seed]`

**Scaffolds a new obsidian-second-brain command through a short interview - no markdown or frontmatter editing.**

Full procedure: `commands/create-command.md`

---

### `/obsidian-ingest`

**Ingests a source into the vault - one source touches many pages.**

Full procedure: `commands/obsidian-ingest.md`

---

## Thinking Tools

These commands use the vault as a thinking partner - not just storage. They surface insights, challenge assumptions, and generate connections that the user cannot see on their own.

---

### `/obsidian-challenge`

**Red-teams your current idea against your own vault history.**

Full procedure: `commands/obsidian-challenge.md`

---

### `/obsidian-emerge`

**Surfaces unnamed patterns from recent notes - recurring themes and conclusions you haven't explicitly stated.**

Full procedure: `commands/obsidian-emerge.md`

---

### `/obsidian-connect [topic A] [topic B]`

**Bridges two unrelated domains using the vault's link graph to spark new ideas.**

Full procedure: `commands/obsidian-connect.md`

---

### `/obsidian-graduate`

**Promotes an idea fragment into a full project spec with tasks, board entries, and structure.**

Full procedure: `commands/obsidian-graduate.md`

---

### `/obsidian-panel`

**Convenes a panel of distinct perspectives on a decision - one independent verdict per lens, then a synthesis.**

Full procedure: `commands/obsidian-panel.md`

---

### `/vault-deep-synthesis [topic]`

**Cross-references everything the vault knows about one topic: agreements, contradictions, stale claims, coverage gaps.**

Full procedure: `commands/vault-deep-synthesis.md`

---

### `/obsidian-distill [note or source]`

**Condenses one source into key claims, each tagged with provenance back to the exact block it came from.** A distillation, not a summary: condensed enough to read fast, anchored enough to audit.

Full procedure: `commands/obsidian-distill.md`

---

### `/idea-discovery`

**Ranks 3-5 next-direction candidates from ungraduated ideas, open project questions, and orphan research.**

Full procedure: `commands/idea-discovery.md`

---

### `/obsidian-learn [recent|all|topic]`

**Reviews the lessons scattered across the vault and turns them into a living rulebook.**

Full procedure: `commands/obsidian-learn.md`

---

## Context Engine

### `/obsidian-world`

**Loads your identity, values, priorities, and current state in one shot - with progressive context levels.**

Full procedure: `commands/obsidian-world.md`

---

### `/obsidian-decide [topic] [--formal]`

**Records decisions at two depths.** Default mode captures the decisions made in a conversation as dated one-liners appended to the relevant project notes' `## Key Decisions` sections (and the daily note) - for the steady stream of choices made while working. `--formal` (or leading with `adr`) instead writes one full Architecture Decision Record - Decision / Context / Options Considered / Rationale / Consequences / Related - to the decisions folder (resolved per `references/folder-map.md`: wiki-style `wiki/decisions/`, Obsidian-style `Knowledge/`), links it from the project's Key Decisions and `index.md`, and logs it. Use the formal mode for a structural or directional decision worth a real writeup; `python scripts/mine_commit_decisions.py` surfaces decision-shaped commits as ADR candidates.

Full procedure: `commands/obsidian-decide.md`

---

## Research Commands

Seven commands that pull external knowledge into the vault - X posts, X discourse, web research with citations, vault-grounded synthesis, YouTube videos, and podcast episodes (plus `/obsidian-ingest` above for arbitrary URLs, PDFs, audio, and screenshots). All output AI-first notes per the vault's Section 0 rule (preamble, rich frontmatter, recency markers, mandatory wikilinks, sources verbatim).

**Setup:** API keys live at `~/.config/obsidian-second-brain/.env`. Run `install.sh` and answer "y" to the research toolkit prompt, or copy `.env.example` manually. xAI Grok and Perplexity keys are required; YouTube key is optional (transcripts work without it).

**Stack:** Python 3.10+ with `uv`. Install deps via `uv sync` from the repo root.

---

### `/x-read [url]`

**Deep-read an X post** via Grok + Live Search. Verbatim post + thread + TL;DR + key claims + reply sentiment + voices to watch.

Full procedure: `commands/x-read.md`

---

### `/x-pulse [topic]`

**Scan X for what's trending** in a topic. Themes (with rep posts + voices), gaps, hooks working, voice/tone, post ideas.

Full procedure: `commands/x-pulse.md`

---

### `/research [topic]`

**Web research with citations** via Perplexity Sonar Pro. Deep dossier: summary, key facts (with recency markers), timeline, key players, contrarian views, further reading, open questions.

Full procedure: `commands/research.md`

---

### `/research-deep [topic]`

**Vault-first deep research with cross-vault propagation.** The chain-everything command.

Full procedure: `commands/research-deep.md`

---

### `/notebooklm [topic]`

**Vault-first source-grounded research.** The parallel to `/research-deep` - but grounded in your own sources instead of the open web.

Full procedure: `commands/notebooklm.md`

---

### `/youtube [url] [--visual]`

**Extract and summarize a YouTube video.** Transcript (free, no API key) + metadata + top comments (Data API v3, optional) → summarized via Grok. Add `--visual` to also *watch* the video: scene-change frame extraction Claude reads with its own vision.

Full procedure: `commands/youtube.md`

---

### `/podcast [url]`

**Extract and summarize a podcast episode.** Apple Podcasts URL or RSS feed → transcript (RSS `<podcast:transcript>` tag, Whisper API if `OPENAI_API_KEY` set, or show-notes fallback) → summarized via Grok.

Full procedure: `commands/podcast.md`

---

### Cost tracking

`/x-read`, `/x-pulse`, `/youtube`, and `/podcast` (Grok summarize step) log usage to `~/.research-toolkit/usage.log`. View monthly totals via:
```bash
uv run python -c "from scripts.research.lib.usage import month_total; t,c = month_total(); print(f'\${t:.2f} across {c} calls')"
```

No usage tracking on Perplexity calls (intentional - user opted out).

No hard caps. No blocking. No per-call confirmation prompts. Trust the user to monitor.

---

## Scheduled Agents

Four autonomous agents designed to run on a schedule with no user intervention. Each runs a focused vault operation at a set time, then stops. They are conservative by default - they never delete or archive anything autonomously, and they never ask the user questions mid-run.

Set these up once using the `/schedule` skill in Claude Code.

---

### `obsidian-morning` - Daily at 8:00 AM

**Creates today's daily note and surfaces what needs attention.**

Prompt to schedule:
```
Read _CLAUDE.md. Create today's daily note in Daily/ using the Daily Note template.
Pull in any tasks from kanban boards that are due today or overdue.
List any projects with status active that have no recent activity in the last 7 days.
Do not ask questions — infer everything from the vault. Save and stop.
```

Setup:
```
/schedule obsidian-morning — daily 8:00 AM
```

---

### `obsidian-nightly` - Daily at 10:00 PM

**Sleeptime consolidation - the vault gets smarter overnight.**

This agent does more than close the day. It actively consolidates and improves the vault while you sleep.

Prompt to schedule:
```
Read _CLAUDE.md. This is a sleeptime consolidation pass — the vault should be smarter when the user wakes up.

Phase 1 — Close the day:
- Read today's daily note. Append a ## End of Day section with a 3-5 bullet summary.
- Move any completed kanban tasks to Done.

Phase 2 — Reconcile:
- Scan wiki/entities/ for outdated roles, companies, or descriptions that conflict with newer daily notes.
- Scan wiki/concepts/ for claims contradicted by recently ingested sources.
- Auto-resolve clear winners. Flag ambiguous ones in wiki/decisions/.

Phase 3 — Synthesize:
- Scan sources ingested today and yesterday. Find concepts that appear in 2+ unrelated sources.
- If patterns found: create wiki/concepts/Synthesis — Title.md with evidence and interpretation.

Phase 4 — Heal:
- Find notes created today with no incoming links. Add links from relevant existing pages.
- Check if any entity pages reference old timeline entries without an "until" date that should be closed.
- Rebuild index.md to reflect today's changes.

Phase 5 — Log:
- Append to log.md: ## [YYYY-MM-DD] nightly | End of day + X reconciled, Y synthesized, Z orphans linked

Do not ask questions. Do not fix anything destructive — only add, update, link. Save and stop.
```

Setup:
```
/schedule obsidian-nightly — daily 10:00 PM
```

---

### `obsidian-weekly` - Every Friday at 6:00 PM

**Generates a weekly review note from the vault.**

Prompt to schedule:
```
Read _CLAUDE.md. Run /obsidian-recap week to gather this week's activity.
Generate a weekly review note using the Review template (or standard structure if none exists).
Save to Reviews/YYYY-MM-DD — Weekly Review.md.
Link it from this week's last daily note.
Do not ask questions. Save and stop.
```

Setup:
```
/schedule obsidian-weekly — every Friday 6:00 PM
```

---

### `obsidian-health-check` - Every Sunday at 9:00 PM

**Runs the vault health check and logs a report.**

Prompt to schedule:
```
Read _CLAUDE.md. Run: python scripts/vault_health.py --path ~/path/to/vault --json
Parse the output. Write a health report to Knowledge/Vault Health YYYY-MM-DD.md
summarizing findings by severity (critical, warning, info).
Do not fix anything autonomously — only report.
Do not ask questions. Save and stop.
```

Setup:
```
/schedule obsidian-health-check — every Sunday 9:00 PM
```

---

### Setting up scheduled agents

All four can be configured at once:

```
/schedule
```

Then tell Claude which agents you want and at what times. Claude Code's scheduling system will handle the rest - agents run autonomously in the background on the defined cron schedule.

To list or remove scheduled agents:
```
/schedule list
/schedule remove obsidian-morning
```

### Running commands headless (`claude -p`) - important gotcha

Custom slash commands do NOT expand in non-interactive mode. A cron job or launchd
job that runs `claude -p "/obsidian-daily"` will send the literal text `/obsidian-daily`
as a prompt - Claude never loads the command file, so nothing happens.

The reliable pattern for any headless run (cron, launchd, a wrapper script) is to point
Claude at the command file and tell it to carry out the instructions:

```bash
# Wrong - the slash command is not expanded in -p mode:
claude -p "/obsidian-daily"

# Right - read the command file and execute its steps:
cd "$VAULT" && claude --dangerously-skip-permissions \
  -p "Read ~/.claude/commands/obsidian-daily.md and carry out its instructions exactly."
```

Because `~/.claude/commands/obsidian-daily.md` is symlinked from this repo (see Testing
locally in the README), a scheduled run always uses the current command logic. Export an
explicit `PATH` in launchd jobs - launchd strips the environment, so `claude` and `python3`
may not be found otherwise.

---

## Background Agent (PostCompact Hook)

A background agent that fires automatically whenever Claude compacts the conversation context. It reads the session summary and propagates everything worth preserving to the vault - no user action required.

**What it does:** After each compaction, a headless `claude -p` subprocess wakes up, reads `_CLAUDE.md`, scans the summary for vault-worthy items (people, projects, decisions, tasks, dev work, ideas), and writes updates everywhere they belong - people notes, project notes, dev logs, kanban boards, and today's daily note.

**How it works:**
1. `PostCompact` hook fires in Claude Code after context compaction
2. Hook script reads the JSON summary from stdin
3. Spawns a headless `claude --dangerously-skip-permissions -p` subprocess in the vault directory
4. Agent runs silently, propagates updates, and exits - user sees nothing

**Setup:**

1. Make the hook script executable (one-time):
   ```bash
   chmod +x ~/.claude/skills/obsidian-second-brain/hooks/obsidian-bg-agent.sh
   ```

2. Set `OBSIDIAN_VAULT_PATH` in `~/.claude/settings.json`:
   ```json
   {
     "env": {
       "OBSIDIAN_VAULT_PATH": "/path/to/your/vault"
     }
   }
   ```

3. Add the `PostCompact` hook to `~/.claude/settings.json`:
   ```json
   {
     "hooks": {
       "PostCompact": [
         {
           "matcher": "",
           "hooks": [
             {
               "type": "command",
               "command": "/Users/you/.claude/skills/obsidian-second-brain/hooks/obsidian-bg-agent.sh",
               "timeout": 10,
               "async": true
             }
           ]
         }
       ]
     }
   }
   ```

**Debugging:** The agent logs to `/tmp/obsidian-bg-agent.log`. Check there if updates aren't appearing.

**Safety:** The agent never deletes, archives, or merges anything. It only adds or updates. If the summary has nothing vault-worthy, it exits without touching the vault.

---

## Write-Time AI-First Validator (PostToolUse Hook)

A non-blocking validator that fires after every `Write` or `Edit` on a markdown file inside the configured vault. It warns when the file fails the AI-first rule (missing required frontmatter, missing `## For future Claude` preamble, broken YAML) and surfaces the warning back to Claude on stderr so the agent can repair the note in the same turn.

**What it checks:**
1. The file has frontmatter delimiters (`--- ... ---`)
2. No tabs in frontmatter (YAML requires spaces)
3. Required AI-first fields present: `date:`, `type:`, `tags:`, `ai-first: true`
4. The body contains a `## For future Claude` preamble (rule #2 of [`references/ai-first-rules.md`](references/ai-first-rules.md))

**What it skips:**
- Files outside `OBSIDIAN_VAULT_PATH`
- Files under `raw/`, `templates/`, `_export/`, `.obsidian/`, `.git/`, `.trash/`

**Setup:**

1. Make the script executable (one-time):
   ```bash
   chmod +x ~/.claude/skills/obsidian-second-brain/hooks/validate-ai-first.sh
   ```

2. `OBSIDIAN_VAULT_PATH` must already be set in `~/.claude/settings.json` (the background agent setup above covers this).

3. Add the `PostToolUse` hook to `~/.claude/settings.json`:
   ```json
   {
     "hooks": {
       "PostToolUse": [
         {
           "matcher": "Write|Edit",
           "hooks": [
             {
               "type": "command",
               "command": "bash ~/.claude/skills/obsidian-second-brain/hooks/validate-ai-first.sh"
             }
           ]
         }
       ]
     }
   }
   ```

**Behavior:** Non-blocking. If a write fails the AI-first rule, Claude sees the warning text on stderr (with one line per missing requirement) and can re-write the file in the same conversation turn to fix it. The original write is NOT reverted.

**Other platforms (Codex CLI / Gemini CLI / OpenCode):** The hook script ships in `dist/<platform>/hooks/` for all platform builds, but each platform's hook system differs. Wiring it up beyond Claude Code is left to the platform's own configuration. See [`hooks/validate-ai-first.hook.yaml`](hooks/validate-ai-first.hook.yaml) for the platform-neutral spec.

---

## Per-Project Vaults (multi-repo workflows)

The default install path (`scripts/setup.sh`) writes `OBSIDIAN_VAULT_PATH` into `~/.claude/settings.json` (global). That assumes one machine = one vault. If you work across multiple repos and want each to have its own dedicated vault - or want to keep work and personal vaults separate without manually swapping config - use Claude Code's per-project settings to override the global setting on a per-directory basis.

**How it works:** every hook in this skill (`hooks/load_vault_context.py`, `hooks/validate-ai-first.sh`, `hooks/obsidian-bg-agent.sh`) reads `OBSIDIAN_VAULT_PATH` from process env at fire-time. Claude Code merges `.claude/settings.json` from the current project directory on top of `~/.claude/settings.json`, so a project-scoped env block overrides the global one for any session launched from that directory.

**Setup per repo:**

1. Bootstrap each vault once: `bash scripts/setup.sh` for the first vault (writes the global default), then `python scripts/bootstrap_vault.py --path ~/vaults/repo-b --name "Your Name"` for any additional vaults (skips the global config step).

2. In each repo where you want a non-default vault, create `.claude/settings.json`:

   ```json
   {
     "env": {
       "OBSIDIAN_VAULT_PATH": "/Users/you/vaults/repo-a-vault"
     }
   }
   ```

3. Restart Claude Code (or open a new session in that directory). All slash commands, hooks, and scripts will now operate on the project-specific vault.

**What this does NOT give you:** isolation within a single vault. The skill has no `--scope` concept - `/obsidian-find`, `/obsidian-recap`, and `/obsidian-emerge` scan the entire configured vault. If you want multiple projects sharing one vault, you can organize them by top-level folders for visual grouping, but commands will still see across folders. A real `--scope` refactor is tracked in discussion threads - open a discussion if this is your use case.

**Slash commands and hooks are still globally installed.** Only the `OBSIDIAN_VAULT_PATH` env var is per-project. You do not need to re-symlink commands or re-register hooks per repo.

---

## Reference Files

- `references/vault-schema.md` - Complete folder structure + frontmatter specs for all note types
- `references/write-rules.md` - Detailed writing, linking, and formatting rules
- `references/claude-md-template.md` - Template for generating a vault's `_CLAUDE.md`

## Scripts

- `scripts/setup.sh` - One-command installer (wires hook + env var + MCP)
- `scripts/bootstrap_vault.py` - Bootstrap a complete vault from scratch
- `scripts/vault_health.py` - Audit a vault for structural issues
