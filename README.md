# Agent Skills

A library of agent skills for Claude Code, claude.ai and other agents that support the `SKILL.md` format.

## Skills

| Skill | Description |
|---|---|
| [dnd-kit-react](dnd-kit-react) | Drag-and-drop UIs in React with dnd kit: the new `@dnd-kit/react` API, plus maintaining or migrating legacy `@dnd-kit/core` / `@dnd-kit/sortable`. |

## Install

### With the skills CLI (Claude Code, Cursor, Codex, …)

```bash
npx skills add ChengTze-Wu/skills            # pick from the list
npx skills add ChengTze-Wu/skills --skill dnd-kit-react
```

### Manually (Claude Code)

Copy the skill folder into your personal or project skills directory:

```bash
git clone https://github.com/ChengTze-Wu/skills.git
cp -r skills/dnd-kit-react ~/.claude/skills/        # all projects
cp -r skills/dnd-kit-react .claude/skills/          # this project only
```

### claude.ai / Claude Desktop

Download the skill's zip from [Releases](../../releases) and upload it under **Settings → Capabilities → Skills**.

## Layout

```
<skill-name>/
├── SKILL.md            # required: frontmatter (name, description) + instructions
├── references/         # optional: docs loaded on demand
├── scripts/            # optional: helper scripts
└── evals/evals.json    # optional: test prompts and expected behavior
```

Some evals use non-English prompts on purpose, to check that the skill still triggers and that the answer comes back in the user's language.

## Adding a skill

1. Create `<skill-name>/SKILL.md` at the repo root. The folder name must match `name` in the frontmatter.
2. Add a row to the table above.

## License

[MIT](LICENSE)
