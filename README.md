# Rock Star Skills

A collection of [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills for knowledge management, wiki building, and more.

## Skills

### LLM Wiki

A pattern for building persistent, interlinked knowledge bases using LLMs. Instead of re-deriving knowledge from raw documents on every query (like RAG), the LLM incrementally builds and maintains a structured wiki of markdown files. The wiki compounds over time as you add sources and ask questions.

Based on [Andrej Karpathy's LLM Wiki idea](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f). See also the [local copy with annotations](LLMWiki.md).

| Skill | Purpose |
|-------|---------|
| `/llmwiki:init` | Initialize wiki structure, set up qmd search, run first full ingest |
| `/llmwiki:ingest` | Process new/modified sources into the wiki (MD5 change detection) |
| `/llmwiki:search <query>` | Search wiki via qmd and synthesize answers with citations |
| `/llmwiki:optimize` | Compact, merge, reorganize, strengthen cross-references |
| `/llmwiki:health` | Audit for broken links, orphans, contradictions, source drift |

**How it works:**

1. You drop source files (articles, papers, notes) into `raw/`, `docs/`, or `notes/`.
2. Run `/llmwiki:ingest` -- the LLM reads each source, creates summary pages, entity pages, concept pages, and maintains cross-references using `[[wikilinks]]`.
3. Query the wiki with `/llmwiki:search` -- get synthesized answers with citations.
4. The wiki keeps getting richer with every source you add and every question you ask.

**Works great with [Obsidian](https://obsidian.md/)** -- the wiki is just a folder of interlinked markdown files. Open it in Obsidian and use graph view to see the shape of your knowledge base.

## Installation

### As a Claude Code plugin (recommended)

Run these commands in Claude Code:

```
# Add the marketplace (one-time)
/plugin marketplace add marvec/rock-star-skills

# Install the plugin
/plugin install rock-star-skills@rock-star-skills
```

### Manual settings.json

Alternatively, add to your `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "rock-star-skills": {
      "source": {
        "source": "github",
        "repo": "marvec/rock-star-skills"
      }
    }
  },
  "enabledPlugins": {
    "rock-star-skills@rock-star-skills": true
  }
}
```

Then restart Claude Code.

### Manual symlink installation

Clone the repo and symlink each skill into your Claude skills directory:

```bash
git clone https://github.com/marvec/rock-star-skills.git
cd rock-star-skills

# Symlink all skills
for skill in skills/*/; do
  ln -sf "$(pwd)/$skill" "$HOME/.claude/skills/$(basename $skill)"
done
```

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI or IDE extension
- [qmd](https://github.com/tobi/qmd) (optional but recommended) -- local markdown search engine with hybrid BM25/vector search. Install with `npm install -g @tobilu/qmd`. Without qmd, skills fall back to index-based search and grep.

## Wiki Structure

After running `/llmwiki:init`, the wiki directory looks like this:

```
wiki/
├── index.md          # Master index of all pages (updated on every ingest)
├── log.md            # Chronological log of all operations
├── .manifest.json    # MD5 hashes for source change detection
├── entities/         # Pages about specific things (people, companies, tools)
├── concepts/         # Conceptual/topic pages
├── sources/          # One summary page per ingested source document
├── comparisons/      # Comparison tables, side-by-side analyses
└── synthesis/        # Cross-cutting analyses, evolving theses
```

Source directories (never modified by the LLM):

```
raw/                  # Primary source documents
├── attachments/      # Images and binary assets
docs/                 # Research, ideas, brainstorms
notes/                # Personal notes, daily logs
```

## Page Conventions

Every wiki page has YAML frontmatter:

```yaml
---
title: Page Title
type: entity | concept | source | comparison | synthesis
tags: [tag1, tag2]
sources: [path/to/original.md]
created: 2026-04-06
updated: 2026-04-06
---
```

Cross-references use Obsidian-compatible `[[wikilinks]]`. All links are bidirectional.

## Change Detection

The ingest skill tracks every source file via MD5 checksums in `wiki/.manifest.json`. On each run it detects:

- **New files** -- not in manifest, queued for ingest
- **Modified files** -- MD5 changed, queued for re-ingest (updates existing wiki pages)
- **Deleted files** -- flagged for user attention
- **Unchanged files** -- skipped

The health skill also checks for source drift and reports stale wiki pages.

## Usage

### Setup

Add the following to your project's `CLAUDE.md` (or equivalent) so the LLM knows how to use the wiki:

```markdown
## LLM Wiki

This workspace includes an LLM-maintained wiki — a persistent, interlinked knowledge base
built from source documents. The LLM writes and maintains the wiki; the user curates sources
and directs analysis.

### Source Directories (Immutable — never modify)
- **`raw/`** — Primary source documents (articles, papers, clippings). Attachments in `raw/attachments/`.
- **`docs/`** — Research, ideas, brainstorms (also ingested into the wiki).
- **`notes/`** — Personal notes, daily logs.

### Wiki Output (`wiki/`)
All wiki pages live here. The LLM owns this directory entirely. Key files:
- `wiki/index.md` — Master index, updated on every ingest. Read this first when searching.
- `wiki/log.md` — Append-only chronological log of all operations.

### Wiki Page Conventions
- **Frontmatter:** Every page has YAML frontmatter with `title`, `type`
  (entity/concept/source/comparison/synthesis), `tags`, `sources`, `created`, `updated`.
- **Wikilinks:** Use `[[Page Title]]` for cross-references (Obsidian-compatible). Always bidirectional.
- **Filenames:** Lowercase kebab-case, `.md` extension. No spaces or special characters.

### Skills (invoke with `/llmwiki:<name>`)

| Skill | Purpose | When to Use |
|-------|---------|-------------|
| `/llmwiki:init` | Initialize wiki structure, set up qmd, run first full ingest | Once per project setup |
| `/llmwiki:ingest` | Process new/modified sources, create/update wiki pages | After adding new files to `raw/`, `docs/`, or `notes/` |
| `/llmwiki:search <query>` | Search wiki via qmd and synthesize answers with citations | When asking questions against the knowledge base |
| `/llmwiki:optimize` | Compact, merge, reorganize, strengthen cross-references | Periodically as wiki grows (every ~10-20 ingests) |
| `/llmwiki:health` | Audit for broken links, orphans, contradictions, stale content | Periodically to maintain wiki quality |

### Workflow
1. Drop source files into `raw/` (use Obsidian Web Clipper for articles).
2. Run `/llmwiki:ingest` to process new sources into the wiki.
3. Use `/llmwiki:search` to query the knowledge base.
4. Run `/llmwiki:health` periodically to check for issues.
5. Run `/llmwiki:optimize` when the wiki feels bloated or fragmented.

### Search (qmd)
The wiki is indexed by qmd (local markdown search engine) with collection name `wiki`.
Skills use qmd MCP tools automatically. To reindex manually: `qmd update`.
To build embeddings: `qmd embed`.
```

### Tips

- **[Obsidian Web Clipper](https://obsidian.md/clipper)** is a browser extension that converts web articles to markdown. Very useful for quickly getting sources into `raw/`.
- **Download images locally.** In Obsidian, set attachment folder path to `raw/attachments/` and bind a hotkey to download attachments. This lets the LLM view images directly.
- **Obsidian's graph view** is the best way to see the shape of your wiki -- what's connected, which pages are hubs, which are orphans.
- **Good answers can be filed back.** When `/llmwiki:search` produces a valuable analysis, it offers to save it as a wiki page so your explorations compound too.
- **The wiki is just markdown files.** It's a git repo, version-controlled, portable, and works with any tool that reads markdown.

## Adding Your Own Skills

To add a new skill to this plugin:

1. Create a directory under `skills/` with your skill name (kebab-case).
2. Add a `SKILL.md` file with YAML frontmatter (`name`, `description`, `allowed-tools`).
3. The skill will be automatically discovered when the plugin is installed.

## Credits

The LLM Wiki pattern is based on [Andrej Karpathy's idea](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f) of using LLMs to incrementally build and maintain persistent knowledge bases instead of re-deriving knowledge on every query.

## License

This project is licensed under the GNU General Public License v3.0. See [LICENSE](LICENSE) for details.
