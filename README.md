# Ketrics IO Skills

A collection of [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skills for building applications on the [Ketrics IO](https://ketrics.io) platform.

## What are Claude Code Skills?

Skills are reusable prompt files that give Claude Code specialized knowledge about tools, frameworks, or workflows. When installed, they let Claude assist you with platform-specific tasks without needing to explain the context every time.

## Available Skills

| Skill | Description |
|-------|-------------|
| [ketrics-app](skills/ketrics-app/) | Build full-stack Ketrics tenant applications with backend handler functions and React frontends |

## Installation

### Option 1: Ketrics CLI (recommended)

```bash
# Install for your user - available in every project on this machine
ketrics skills

# Or install into a single repository
ketrics skills --project
```

`ketrics skills` reads `skills.json` from this repository, prompts for a skill,
and writes it to `~/.claude/skills/<name>` — or to `./.claude/skills/<name>`
with `--project`. Installing replaces any existing copy of that skill rather
than merging, so every machine converges on what this repository says.

A skill is normally a capability of the *developer* rather than of one checkout,
which is why the user-level install is the default. Reach for `--project` when a
skill genuinely belongs to a single repository.

Requires `@ketrics/ketrics-cli` 0.11.0 or later:

```bash
npm install -g @ketrics/ketrics-cli@latest
```

Install with `@latest`, never a bare `npm install -g @ketrics/ketrics-cli`. The
CLI declares `engines.node >= 24`, and on an older Node npm silently resolves
*down* to the newest version whose engines are satisfied instead of failing —
which is `0.1.0`, from December 2025. Check with `ketrics --version`.

### Option 2: Manual installation

Copy the skill folder into your skills directory — `~/.claude/skills/` for your
user, or `.claude/skills/` inside a project:

```bash
mkdir -p ~/.claude/skills/ketrics-app
cp -r skills/ketrics-app/* ~/.claude/skills/ketrics-app/
```

Claude Code detects the skill on the next session.

### Updating a skill

Change it here, push, and re-run `ketrics skills`. Do not edit an installed copy
directly: this repository is the source of truth, and a local edit is
overwritten by the next install with no warning.

## Skill Details

### ketrics-app

Scaffolds and builds Ketrics tenant applications. Covers:

- **Project scaffolding** — `ketrics.config.json`, backend/frontend directory structure
- **Backend handlers** — Database queries, DocumentDB, Volumes, Excel generation, messaging, background jobs, HTTP client, cross-app invocation
- **Frontend** — React + Vite + TypeScript with auth manager, service layer, mock handlers for local dev
- **Deployment** — GitHub Actions CI/CD pipeline, manual deploy via Ketrics CLI

Reference docs included:
- [SKILL.md](skills/ketrics-app/SKILL.md) — Main skill definition and quick reference
- [BACKEND_REFERENCE.md](skills/ketrics-app/BACKEND_REFERENCE.md) — Complete backend SDK API
- [FRONTEND_REFERENCE.md](skills/ketrics-app/FRONTEND_REFERENCE.md) — Frontend patterns and service layer
- [CONFIG_AND_DEPLOY.md](skills/ketrics-app/CONFIG_AND_DEPLOY.md) — Configuration and deployment guide

## Contributing

To add a new skill:

1. Create a directory under `skills/` with your skill name (e.g., `skills/my-skill/`)
2. Add a `SKILL.md` file with the frontmatter and skill instructions (see existing skills for the format)
3. Optionally add supplementary reference files
4. Update this README with the new skill in the table above
5. Submit a pull request

### Skill file format

Every skill requires a `SKILL.md` with YAML frontmatter:

```markdown
---
name: my-skill-name
description: One-line description of what the skill does and when to use it.
---

# Skill Title

Instructions and reference material for Claude Code...
```

## License

MIT
