# skills

A collection of [Claude Code](https://claude.com/claude-code) skills by [@AbdulMattee](https://github.com/AbdulMattee). Skills extend Claude Code with reusable, packaged instructions Claude loads on demand.

## Install

Browse and pick interactively:

```bash
npx skills add AbdulMattee/skills
```

Install a specific skill directly:

```bash
npx skills add AbdulMattee/skills --skill <skill-name>
```

List what's available without installing:

```bash
npx skills add AbdulMattee/skills --list
```

## Available skills

| Skill | Description |
|---|---|
| [`watch-pr`](skills/watch-pr/) | Automated PR review loop — polls a GitHub PR every minute, reads new review comments from bots and humans, fixes the code or replies, commits, and stops on approval/merge/close. |

See each skill's directory for full usage, configuration, and prerequisites.

## Layout

```
skills/
└── <skill-name>/
    ├── SKILL.md      # Instructions Claude loads when the skill is invoked
    └── README.md     # Human-facing docs (optional)
```

## License

MIT
