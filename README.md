# aura-skill

AI skill that teaches coding assistants how to use the [Aura framework](https://github.com/tianhaocui/aura).

## Installation

```bash
npx skills add tianhaocui/aura-skill
```

### Manual Installation

#### Claude Code

```bash
cp SKILL.md ~/.claude/skills/aura-framework/SKILL.md
```

#### Cursor

Copy the content of `SKILL.md` into your project's `.cursorrules` file.

#### GitHub Copilot

Copy the content into `.github/copilot-instructions.md`.

#### Any project using Aura

```bash
cp SKILL.md /path/to/your-project/CLAUDE.md
```

## What it does

When the AI detects you're working with Aura (mentions "aura", sees `io.aura` imports, or finds `aura-web` in pom.xml), it automatically knows:

- How to create routes (lambda, method ref, crud)
- How to write Service classes (plain Java, no annotations)
- How parameter binding works
- How to use the database module
- How to add middleware and error handling
- How to enable MCP for AI agent access
- How to test with TestClient
- Common mistakes to avoid
