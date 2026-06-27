# Setup Guide

## 1. Install the Skill

Clone or copy this skill into the directory your agent loads skills from.

Claude Code example:

```bash
git clone https://github.com/stanley6635/meeting-ingest.git ~/.claude/skills/meeting-ingest
```

A separate code repository is not required at runtime. Keeping one canonical repo is useful for versioning and syncing the installed skill, but the agent only needs the installed skill files.

## 2. Knowledge Base Layout

The skill assumes a structured TARS-style knowledge base:

```text
your-kb/
  raw/
    meetings/     # ASR-generated meeting transcripts
  wiki/
    people/
    products/
    mechanisms/
    events/
    projects/
    judgments/
  index.md
  AGENTS.md
  CONVENTIONS.md
```

In TARS, `raw/` is normally immutable. The explicit exception is `raw/meetings/`: ASR-generated meeting minutes or transcripts may be corrected after Stanley confirms the correction table. This exception exists because uncorrected ASR errors can pollute downstream wiki pages and RO documents.

The exception does not apply to images, PDFs, web clippings, emails, product files, or other raw sources.

## 3. Configure Paths

Edit `skill.md`, find the configuration section, and set:

| Variable | What to set |
|----------|-------------|
| `$MEETINGS_DIR` | Where meeting transcript files live, e.g. `raw/meetings/` |
| `$WIKI_DIR` | Wiki pages root, e.g. `wiki/` |
| `$INDEX_FILE` | Wiki index, e.g. `index.md` |

Wiki search is handled through agentmemory `memory_smart_search`; final write-back is delegated to `pro-workflow:wiki-builder`.

## 4. Required Related Skills

- `file-ingest`: archives incoming transcript files into `$MEETINGS_DIR`.
- `pro-workflow:wiki-builder`: performs the final structured wiki write-back.
- agentmemory `memory_smart_search`: searches known people, products, mechanisms, projects, and prior notes.

## 5. Test

Place a meeting transcript `.txt` or `.md` file into `$MEETINGS_DIR`, then ask the agent:

> 处理一下这个会议记录

Expected behavior:

1. The agent identifies the file as a spoken meeting transcript.
2. It extracts high-risk fields such as speakers, people, products, organizations, hospitals, terms, dates, prices, quantities, and project names.
3. It searches the wiki and other allowed evidence sources.
4. It shows a correction table before modifying the transcript.
5. Only corrections confirmed as “改” are written back to the transcript file.
6. It then searches and reads relevant wiki pages before handing the corrected transcript to wiki-builder.

## Troubleshooting

- If too many items are marked unconfirmed, the available evidence is not strong enough; keep them unchanged or confirm them manually.
- If speaker mapping looks unstable, remember ASR speaker numbers are not reliable identities across the whole transcript.
- If the agent tries to correct image OCR under this skill, route the task away from meeting-ingest; this skill is only for spoken meeting transcripts.
- If the installed skill behaves differently from the repo copy, sync `skill.md`, `README.md`, and `SETUP.md` between the canonical repo and the installed skill directory.
