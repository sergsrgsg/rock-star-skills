---
name: llmwiki:init
description: Initialize the LLM Wiki. Creates directory structure, index, log, sets up qmd collection, and runs initial ingest of all existing content from raw/, docs/, and notes/. Run once per project.
user-invokable: true
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, Agent, mcp__plugin_qmd_qmd__query, mcp__plugin_qmd_qmd__status, mcp__plugin_qmd_qmd__get, mcp__plugin_qmd_qmd__multi_get, TaskCreate, TaskUpdate, TaskList, TaskGet
---

# LLM Wiki: Initialize

Set up a new LLM Wiki in the current project and run the first full ingest.

## Pre-flight

1. Check if `wiki/index.md` already exists. If it does and has content beyond the template, **ask the user** whether to reinitialize (destructive) or skip.
2. Verify `raw/`, `docs/`, and `notes/` directories exist. Create any that are missing.
3. Verify `raw/attachments/` exists. Create if missing.

## Step 1: Create Wiki Structure

Ensure the following directories and files exist:

```
wiki/
├── index.md          # Master index of all wiki pages
├── log.md            # Chronological log of operations
├── entities/         # Pages about specific things (people, companies, products, tools)
├── concepts/         # Conceptual/topic pages
├── sources/          # Source summaries (one per ingested source)
├── comparisons/      # Comparison tables, side-by-side analyses
└── synthesis/        # Cross-cutting analyses, insights
```

If files already exist, leave them as-is.

## Step 2: Set Up qmd Collection

Run:
```bash
qmd collection add wiki ./wiki
```

This registers the `wiki/` directory as a qmd collection named "wiki". If the collection already exists, skip.

Then update the index:
```bash
qmd update
```

If qmd is not installed, warn the user: `npm install -g @tobilu/qmd` and continue without it.

## Step 3: Initial Ingest

Collect ALL existing files that should be ingested:

1. **`raw/`** — all markdown files and other documents (skip `raw/attachments/`)
2. **`docs/`** — all markdown files recursively
3. **`notes/`** — all markdown files recursively

For each file found, invoke the **llmwiki:ingest** skill's logic (or invoke the skill itself). Process files in batches using parallel subagents where possible:

- Group files by directory/topic
- For each batch, spawn a subagent to read the files and produce wiki pages
- Each subagent should:
  - Read the source file
  - Determine what wiki pages to create/update (entity pages, concept pages, source summary)
  - Write the source summary to `wiki/sources/`
  - Create or update entity/concept pages in `wiki/entities/` and `wiki/concepts/`
  - Add `[[wikilinks]]` between related pages
  - Return the list of pages created/updated

After all subagents complete:
- Rebuild `wiki/index.md` with all pages
- Append a log entry to `wiki/log.md`:
  ```
  ## [YYYY-MM-DD] init | Initialized wiki with N sources, M pages created
  ```

## Step 4: Build Embeddings (Optional)

If qmd is available, run:
```bash
qmd embed
```

This builds vector embeddings for semantic search. If it fails or takes too long, skip — BM25 search still works.

## Step 5: Report

Print a summary:
- Number of source files ingested
- Number of wiki pages created (by category)
- Any files that failed or were skipped
- qmd status (collection registered, embedding status)

## Wiki Page Conventions

All wiki pages MUST follow these conventions:

### Frontmatter
Every wiki page has YAML frontmatter:

```yaml
---
title: Page Title
type: entity | concept | source | comparison | synthesis
tags: [tag1, tag2]
sources: [relative/path/to/source.md]
created: YYYY-MM-DD
updated: YYYY-MM-DD
---
```

### Wikilinks
Use `[[Page Title]]` syntax for cross-references. Obsidian resolves these automatically. When creating a new page, add wikilinks to all related existing pages, and update those existing pages to link back.

### Source Summaries
Source summary filenames: kebab-case version of the source title, in `wiki/sources/`. Example: `wiki/sources/cross-platform-tech-stacks-2026.md`.

### Entity & Concept Pages
Use descriptive kebab-case filenames. Example: `wiki/entities/expo.md`, `wiki/concepts/local-first-architecture.md`.

### File Naming
- All wiki page filenames: lowercase kebab-case with `.md` extension
- No spaces, no special characters beyond hyphens
- Keep names concise but descriptive
