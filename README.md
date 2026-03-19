# Common Development Pitfalls — Claude Code Skill

A Claude Code skill that provides solutions to frequently encountered bugs in full-stack web development (Python/FastAPI + React/TypeScript).

## What's Included

All pitfalls are documented from real debugging sessions:

| Category | Topics |
|---|---|
| **Python** | Dependency issues (Python 3.13+), Pydantic `model_dump()` gotchas |
| **FastAPI** | Request format mismatches (FormData vs JSON) |
| **React** | Object reference comparison, Zustand persistence, useState stale data |
| **CSS** | Flexbox ellipsis, Ant Design table ellipsis |
| **Ant Design** | Upload `originFileObj`, Table editable+expandable conflict |
| **react-dnd** | Drag vibration fix |
| **Cross-Platform** | macOS NFD ↔ Linux NFC Unicode filename issues |
| **Database** | SQLite safety, cascade delete, SQLAlchemy migration |
| **Data Import** | File-internal duplicate detection |
| **Architecture** | One-way sync, localStorage vs server storage |
| **AI/OCR** | Fuzzy matching for noisy text output |
| **VPS/Server** | SSH safety rules |

## Installation

Copy the `SKILL.md` file to your Claude Code skills directory:

```bash
# Create skill directory
mkdir -p ~/.claude/skills/common-pitfalls

# Copy skill file
cp SKILL.md ~/.claude/skills/common-pitfalls/SKILL.md
```

Claude Code will automatically detect and use the skill when relevant issues are encountered.

## Usage

The skill is triggered automatically when you encounter related issues. You can also invoke it manually:

```
/common-pitfalls
```

## Quick Checklist

The skill includes a diagnostic checklist covering all documented pitfalls — useful as a pre-debug sanity check.

## Contributing

Found a new pitfall? PRs welcome! Follow the existing format:

```markdown
### Short Title

**Symptom:** What you see
**Cause:** Why it happens
**Fix:** Code example showing the solution
```

## License

MIT
