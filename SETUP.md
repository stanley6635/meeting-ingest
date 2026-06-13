# Setup Guide

## 1. Install the Skill

Clone into your skills directory:

**Claude Code:**
```bash
git clone https://github.com/YOUR_USERNAME/meeting-ingest.git ~/.claude/skills/meeting-ingest
```

**OpenCode / other platforms:** place the `skill.md` wherever your platform loads skills from.

## 2. Wiki Knowledge Base Setup

This skill assumes you have a structured wiki knowledge base. The expected structure:

```
your-kb/
  wiki/
    people/       # person pages with frontmatter
    products/     # product pages
    mechanisms/   # market/operational mechanisms
    projects/     # active workstreams
    judgments/    # structured analysis
  raw/
    meetings/     # where transcripts go (default: $MEETINGS_DIR)
  scripts/
    wiki_search.py   # BM25 search over wiki pages
    wiki_lint.py     # structure health checker
  index.md        # top-level navigation
  log.md          # append-only operation log
```

The two scripts (`wiki_search.py`, `wiki_lint.py`) are included in this repo under `scripts/`. Copy them to your knowledge base's scripts directory:

```bash
cp scripts/wiki_search.py scripts/wiki_lint.py /path/to/your-kb/scripts/
```

Both are Python 3 stdlib-only — no dependencies.

## 3. Configure Paths

Edit `skill.md`, find the **配置** section, and update the variables:

| Variable | What to set |
|----------|-------------|
| `$MEETINGS_DIR` | Where meeting transcript files live, e.g. `raw/meetings/` |
| `$WIKI_DIR` | Wiki pages root, e.g. `wiki/` |
| `$WIKI_SEARCH` | Path to search script, e.g. `scripts/wiki_search.py` |
| `$WIKI_LINT` | Path to lint script, e.g. `scripts/wiki_lint.py` |
| `$INDEX_FILE` | Wiki index, e.g. `index.md` |
| `$LOG_FILE` | Operation log, e.g. `log.md` |

Paths can be absolute or relative to your workspace root.

## 4. (Optional) Install file-ingest

The skill delegates file archiving to `file-ingest`. If you don't have it, the skill falls back to manual placement — just move your transcript files into `$MEETINGS_DIR` with the naming format `YYYY-MM-DD_description.ext`.

## 5. Test

Drop a meeting transcript .txt into `$MEETINGS_DIR` and say:

> 处理一下这个会议记录

The skill will run through error correction → wiki search → net new identification → wait for your confirmation before writing anything.

## Wiki Page Frontmatter Convention

For the skill's search and write-back to work correctly, wiki pages should use this frontmatter:

```yaml
---
title: Page Title
type: person | product | mechanism | project | event | judgment
created: YYYY-MM-DD
updated: YYYY-MM-DD
sources:
  - raw/path/to/source.ext
---
```

## Troubleshooting

- **Search returns nothing**: check that `$WIKI_SEARCH` points to a working copy of `wiki_search.py`, and that your wiki pages are in `$WIKI_DIR`
- **Lint fails**: make sure `$WIKI_LINT` is correctly configured and your wiki pages have proper `sources` fields
- **Correction table has too many "unknown" entries**: this is normal for a new wiki — the skill learns as your knowledge base grows
