---
description: Ingest a source into the vault - the vault rewrites itself around new knowledge. Every ingest updates entities, rewrites stale claims, synthesizes new concepts, and resolves contradictions.
category: research
triggers_en: ["ingest this source", "add this article", "import this", "absorb this"]
---

Use the obsidian-second-brain skill. Execute `/obsidian-ingest $ARGUMENTS`:

The argument is a URL, file path, or pasted text. If no argument, ask what to ingest.

1. Read `_CLAUDE.md` first if it exists in the vault root

2. Classify the source type before reading the full content:
   - **Article/blog post** - extract key claims, people, tools, concepts
   - **PDF/document** - extract structure, findings, recommendations
   - **Transcript (meeting/podcast)** - extract speakers, decisions, action items, quotes
   - **YouTube video** - pull metadata, description, and transcript (see step 3 for method)
   - **Audio file** (.m4a, .mp3, .wav, .ogg, .webm) - transcribe, identify speakers, extract decisions/tasks/promises
   - **Image/screenshot** (.png, .jpg, .jpeg, .webp) - read/OCR the image, extract text and context
   - **Raw text** - classify by content (opinion, technical, narrative) and extract accordingly

3. Read or fetch the full source content:

   **For YouTube URLs** - try methods in this order (use the first one that works):

   **Method A - `yt-dlp` (best, works in Claude Code / terminal):**
   ```bash
   which yt-dlp || brew install yt-dlp
   yt-dlp --skip-download --print title --print description --print duration_string --print view_count --print like_count --print upload_date --print channel "URL"
   yt-dlp --write-auto-sub --sub-lang en --skip-download -o "/tmp/%(id)s" "URL"
   ```

   **Method B - YouTube MCP tools (works in Claude Desktop if configured):**
   Check if YouTube MCP tools are available. If so, use them.

   **Method C - oEmbed fallback (works everywhere, limited data):**
   Fetch `https://www.youtube.com/oembed?url=URL&format=json` - gives title and channel only. Ask user to paste description for full ingest.

   **For audio files** (.m4a, .mp3, .wav, .ogg, .webm):
   ```bash
   # Transcribe with Whisper (install if missing)
   which whisper || pip install openai-whisper
   whisper "path/to/audio.m4a" --model base --output_format txt --output_dir /tmp
   ```
   If `whisper` can't be installed, ask the user to paste the transcript.
   After transcription: identify speakers if possible, extract decisions, action items, promises, and who said what.
   Save the transcript to `raw/transcripts/`.

   **For images/screenshots** (.png, .jpg, .jpeg, .webp):
   Claude can read images directly. Analyze the image for:
   - Text content (OCR) - extract all readable text
   - UI screenshots - describe what's shown, extract data from tables/forms/dashboards
   - Whiteboard/diagram photos - describe the structure and extract concepts
   - Chat screenshots - extract messages, people, decisions
   Save the image description to `raw/articles/` as a markdown summary with context.

   **For articles** - use WebFetch to pull the page content
   **For PDFs** - read the file directly
   **For pasted text** - use as-is

4. Extract and organize:
   - **Entities**: people mentioned, companies, tools, projects
   - **Concepts**: key ideas, frameworks, methodologies
   - **Claims**: specific assertions with supporting evidence
   - **Action items**: anything actionable for the user
   - **Quotes**: notable quotes worth preserving

5. Save the raw source to `raw/` (immutable - never modify after saving):
   - Create `raw/articles/YYYY-MM-DD — Source Title.md` (or transcripts/, pdfs/, videos/)
   - Frontmatter: `date`, `tags: [source, <type>]`, `source_url`, `source_type`, `content_hash`

6. **REWRITE the vault** - this is the critical step. Don't just create new pages. Rewrite existing ones.

   Read `index.md` first to understand what already exists in the vault. Then spawn parallel subagents. Include in each agent's prompt: the vault root path, the folder map from `_CLAUDE.md`, the extracted items assigned to it, and the candidate note paths already identified from `index.md`. Agents must not re-read `_CLAUDE.md`, `SKILL.md`, or `index.md`.

   **Write protocol - agents never edit wiki pages directly.** Parallel agents editing the same page silently lose each other's work (blind read-modify-write). Instead, each agent appends its proposed changes to a delta file it alone owns - `raw/deltas/YYYY-MM-DD — <source-slug> — <agent-role>.md` - one entry per target note, in this format. (If these agents run sequentially rather than in parallel - e.g. on a platform without subagents - skip the delta files and edit pages directly: the write protocol exists only to keep parallel writers safe.)

   ```
   ## target: <existing note path, or NEW: <proposed path>>
   action: rewrite-section | append-section | create | flag-contradiction
   section: <heading name - only for the two section actions>
   ---
   <the full proposed markdown for that section or page>
   ```

   - **Entities agent**: for each person/company/tool mentioned:
     - Search `wiki/entities/` for existing page
     - If found: propose a rewrite as a delta entry - merge new info with old, update role/context/interactions, add new links. Don't just append - integrate.
     - If not found: propose a new entity page (action: create) with full context

   - **Concepts agent**: for each idea/framework/methodology:
     - Search `wiki/concepts/` for existing or related pages
     - If found: propose a rewrite as a delta entry - update the concept with new evidence, new examples, new connections. If the new source adds depth, rewrite the whole section.
     - If not found: propose a new concept page (action: create)
     - If the ingest reveals a PATTERN across multiple existing concepts: propose a new synthesis page that connects them (e.g., "Three sources now mention X - this is a trend, not a one-off")

   - **Projects agent**: for each project referenced:
     - Search `wiki/projects/` for matching project
     - If found: propose delta entries updating it with new findings, Recent Activity, and Key Decisions if the source contains relevant decisions

   - **Contradictions agent**: for each claim in the new source:
     - Search the vault for CONFLICTING claims in existing pages
     - If contradiction found: add a `flag-contradiction` delta entry for that page with the conflicting claim, the new evidence, and which claim is more recent/authoritative
     - If the new source SUPERSEDES old info: propose the rewrite as a delta entry, noting what changed and why for the page's history section

7. **Merge pass - the only step that edits wiki pages, always serial.** After all agents return, read the delta files and group entries by target note. For each target note: read it once, apply every delta entry for it in ONE rewrite, reconciling overlapping proposals side by side (contradiction flags get resolved here, where both versions are visible). Immediately after writing each note, mark its delta entries applied (add `applied: true` under the action line) so an interrupted merge never applies an entry twice. Delete each delta file once all its entries are applied.

   **Batch ingest:** when running several `/obsidian-ingest` calls in parallel (one per source), run each ingest through step 6 only (delta files, no merge), then run ONE merge pass over all delta files together, then steps 8-10 once for the whole batch. Never run two merge passes at once.

8. Update structural files:
   - REBUILD `index.md` - don't just append. Regenerate the sections that changed so descriptions stay current with the rewritten pages.
   - Append to the operation log: if `Logs/` exists write `**HH:MM** - ingest | Source Title (type) - X created, Y rewritten, Z contradictions resolved` to `Logs/YYYY-MM-DD.md`; otherwise append `## [YYYY-MM-DD] ingest | Source Title (type) — X created, Y rewritten, Z contradictions resolved` to `log.md`

9. Update today's daily note with:
   - What was ingested
   - What pages were REWRITTEN (not just created - this is the important part)
   - Any contradictions found and how they were resolved
   - Any new synthesis pages created from emerging patterns

10. Report back:
   - Source title and type
   - **New pages created** (list)
   - **Existing pages rewritten** (list with what changed)
   - **Contradictions resolved** (list with old claim vs new claim)
   - **Synthesis pages created** (patterns that emerged from this + existing knowledge)

The vault should be DIFFERENT after every ingest - not just bigger. Pages that existed before should be smarter, more connected, and more current. If an ingest only creates new pages and doesn't rewrite anything, it wasn't deep enough.

---

**AI-first rule:** Every note created or updated by this command MUST follow `references/ai-first-rules.md` - `## For future Claude` preamble, rich frontmatter (`type`, `date`, `tags`, `ai-first: true`, plus type-specific fields), recency markers per external claim, mandatory `[[wikilinks]]` for every person/project/concept referenced, sources preserved verbatim with URLs inline, and confidence levels where applicable. The vault is for future-Claude retrieval - not human reading.

**Anti-fabrication:** Search exhaustively before claiming any note, person, or file is absent - false absence is the most common failure mode - and never invent facts, entities, or dates (mark unknowns as `TBD`). See the anti-fabrication and search-completeness hard rules in `references/ai-first-rules.md`.
