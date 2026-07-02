# nest-skills

A [Claude Code](https://claude.com/claude-code) **plugin marketplace** hosting
shared skills for working with [NestJS](https://nestjs.com) projects, so they
can be installed and used across any NestJS codebase — not just this repo.

## Install

From any project, add this marketplace once:

```
/plugin marketplace add neatau/nest-skills
```

Then install the bundled skills:

```
/plugin install nest-skills@nest-skills
```

CLI equivalents:

```bash
claude plugin marketplace add neatau/nest-skills --scope user
claude plugin install nest-skills@nest-skills
```

Once installed, the skills are available in every session and are namespaced as
`/nest-skills:<skill-name>`.

## Layout

```
nest-skills/
├── .claude-plugin/
│   └── marketplace.json          # Catalog: lists the plugin(s) below
├── plugins/
│   └── nest-skills/
│       ├── .claude-plugin/
│       │   └── plugin.json        # Plugin manifest (bump version to ship updates)
│       └── skills/
│           └── service/
│               ├── SKILL.md       # One folder per skill
│               └── reference.md   # Optional on-demand docs/examples
└── README.md
```

> Skills live **inside the plugin**, not in this repo's own `.claude/skills/`.
> A repo-level `.claude/skills/` would only be active while editing *this* repo;
> putting them in the plugin is what makes them portable to other NestJS codebases.

## Adding a skill

1. Create `plugins/nest-skills/skills/<skill-name>/SKILL.md`.
2. Write the frontmatter — `name` and `description` are what matter most. The
   `description` is the only text loaded up front and is how Claude decides to
   invoke the skill, so front-load the "use when …" triggers (e.g. "Use when
   generating a NestJS module, controller, or provider").
3. Bump `version` in `plugins/nest-skills/.claude-plugin/plugin.json` so
   installed users receive the update.
4. Commit and push. Users get the update on their next `/plugin marketplace update`.

See `plugins/nest-skills/skills/service/SKILL.md` for a worked example (with an
accompanying `reference.md` for on-demand detail).

## Skill authoring reference

Common `SKILL.md` frontmatter fields:

| Field | Purpose |
|---|---|
| `name` | Display name (defaults to the folder name). |
| `description` | **Most important.** What it does + when to use it. |
| `allowed-tools` | Tools pre-approved without prompts while the skill runs. |
| `disable-model-invocation` | `true` = user must invoke it explicitly. |
| `argument-hint` | Autocomplete help for user-invoked skills. |

Bundled files (docs, scripts) in a skill folder are read on demand — reference
them by relative path from `SKILL.md`.
