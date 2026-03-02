# ketrics-io-skills

Public repository of Claude Code skills for building Ketrics IO platform applications.

## Repository structure

```
skills/
  <skill-name>/
    SKILL.md              # Required — skill definition with YAML frontmatter
    *.md                  # Optional — supplementary reference docs
```

## Conventions

- Each skill lives in its own directory under `skills/`
- `SKILL.md` is the entry point — it must have `name` and `description` in YAML frontmatter
- Supplementary docs (e.g., `BACKEND_REFERENCE.md`) are linked from `SKILL.md`
- Keep skill content focused and actionable — Claude Code will read these during sessions
- Use TypeScript code examples throughout
- All code examples should follow Ketrics SDK patterns (no raw imports for backend — use the `ketrics` global)

## Adding a new skill

1. Create `skills/<skill-name>/SKILL.md` with frontmatter
2. Add reference docs as needed
3. Update `README.md` with the new skill in the Available Skills table
