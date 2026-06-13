# meeting-ingest

A Claude Code / OpenCode skill that processes voice-to-text meeting transcripts into structured wiki knowledge.

**What it does**: takes a messy meeting transcription → fixes voice-to-text errors → cross-references your knowledge base → identifies what's actually new → writes it into the right wiki pages.

**Key features**:
- Mandatory error-correction pass (names, products, terms mangled by ASR)
- Full wiki cross-reference to filter out already-known information
- Three-filter gating (known vs new, one-week test, execution vs durable knowledge)
- Structured write-back with time-layered updates and source tracking

## Quick Start

1. Clone this repo into your Claude Code skills directory:
   ```bash
   git clone https://github.com/YOUR_USERNAME/meeting-ingest.git ~/.claude/skills/meeting-ingest
   ```

2. Install the dependent `file-ingest` skill (or skip it — the skill works with manual file placement too).

3. Edit `skill.md` → **配置** section: set `$MEETINGS_DIR`, `$WIKI_DIR`, and other paths to match your knowledge base structure.

4. Make sure your wiki has the required scripts:
   ```bash
   cp scripts/wiki_search.py scripts/wiki_lint.py /path/to/your/kb/scripts/
   ```

## Requirements

- A structured wiki knowledge base with `people/`, `products/`, `mechanisms/`, `projects/` subdirectories
- Python 3 (stdlib only — no pip install needed)
- `file-ingest` skill (optional, for automatic file archiving)

## Setup

See [SETUP.md](SETUP.md) for detailed configuration instructions.

## License

MIT
