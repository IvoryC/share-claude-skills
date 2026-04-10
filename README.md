# Lets Share some Claude Skills

This directory is a git-controled directory for skills developed for Claude.

## What's here

```
share-claude-skills
├── .claude
│   └── skills
│       └── git-commit-checklist
│           └── SKILL.md
├── skills
│   ├── build-q2-pluggins
│   │   ├── references
│   │   │   ├── action-registration.md
│   │   │   ├── external-tools.md
│   │   │   ├── package-structure.md
│   │   │   ├── testing.md
│   │   │   └── type-system.md
│   │   ├── README.md
│   │   └── SKILL.md
│   ├── snakemake
│   │   ├── references
│   │   │   ├── cluster-execution.md
│   │   │   ├── config-and-samples.md
│   │   │   ├── project-structure.md
│   │   │   ├── rule-patterns.md
│   │   │   └── snakemake-style-guide.rst.txt
│   │   ├── README.md
│   │   └── SKILL.md
│   └── when-am-i
│       ├── references
│       │   ├── calendar-context.md
│       │   └── milestones.md
│       ├── deadlines.md
│       └── SKILL.md
├── CLAUDE.md
├── .gitignore
└── README.md
```

## Use These Skills

To use these as your person Claude skills, link your ~/.claude/skills directory to this one.  Alternatively, link individual skill folders to use select skills from this set while keeping your own local set not in this repo.

**Option 1:** To use the whole folder wholesale:
```bash
ln -s $PWD/skills ~/.claude/skills
```
Replace $PWD with the path to this repo or run command from this folder.

**Option 2:** To use individual skills:
```bash
ln -s $PWD/skills/some-skill-1 ~/.claude/skills/some-skill-1
ln -s $PWD/skill/some-skill-2 ~/.claude/skills/some-skill-2
...
```
Option 1 is recommended for a repository that you maintain individually to for your personal collection.
Option 2 is recommended for a shared repository where other people may add skills that you only selectively want to give to Claude.

## Where Claude Looks for Skills

| Location   | Path                                              | Applies to                      | How many? |
|:-----------|:--------------------------------------------------|:--------------------------------|:----------|
| Enterprise | managed settings                                  | All users in your organization  | One       |
| Personal   | `~/.claude/skills/<skill-name>/SKILL.md`          | All your projects               | One       |
| Project    | `.claude/skills/<skill-name>/SKILL.md`            | This project only               | Many      |
| Plugin     | `<plugin>/skills/<skill-name>/SKILL.md`           | Where plugin is enabled         | Many      |

When skills share the same name across levels, higher-priority locations win: **enterprise > personal > project**. Plugin skills are namespaced as `plugin-name:skill-name` so they can never conflict with other levels.

**Personal** skills live in exactly one place: `~/.claude/skills/`. There is only one such folder per user.

**Project** skills are not limited to the repo root. Claude discovers `.claude/skills/` at the directory where you launched `claude`, and also in any subdirectory you're actively working in. In a monorepo, `packages/frontend/.claude/skills/` is loaded automatically when you're editing files under `packages/frontend/`. There can be many `.claude/skills/` locations within a single project.

**Plugin** skills are namespaced and isolated per plugin. Each installed plugin contributes its own `skills/` directory. You can have any number of plugins active at once, each adding their own skills without conflicting.

## Notes

### Potential gotchas

__provided by Claude.ai__

- The `--add-dir` flag grants file access rather than configuration discovery, but skills are an exception: `.claude/skills/` within an added directory is loaded automatically and picked up by live change detection, so you can edit those skills during a session without restarting.  The question is whether live change detection follows symlinks — it likely does on most systems, but worth testing.

- If you symlink the *entire* `~/.claude/skills` directory (option 1), make sure the repo root structure matches what Claude Code expects — i.e., each skill is a subdirectory with a `SKILL.md` inside.

- On macOS, symlinks are followed normally by most tools. On some Linux setups with `inotify`-based watchers, symlink traversal can occasionally behave unexpectedly, but it's rarely a problem in practice.


### Front Matter for skills

https://code.claude.com/docs/en/skills.md — "Frontmatter reference" section

This is still a changing ecosystem. Do not treat this as a comprehensive list.

| Field | Type | Description |
|---|---|---|
| `name` | string | **REQUIRED** Skill identifier (should match directory name) |
| `description` | string | **REQUIRED** Trigger conditions (model-invoked) or `/help` text (user-invoked) |
| `version` | string | __for humans__ Semantic version, e.g. `1.0.0` |
| `argument-hint` | string | Shown to user, e.g. `<required> [optional]` |
| `allowed-tools` | list | Pre-approved tools, reduces permission prompts |
| `model` | string | Override model: `haiku`, `sonnet`, `opus` |
| `disable-model-invocation` | bool | `true` = user-only (for skills with side effects) |
| `user-invocable` | bool | `false` = Claude-only (background knowledge) |
| `context` | string | `fork` = run in isolated subagent |
| `agent` | string | Which agent type when forked (e.g. `Explore`) |

Other front matter fields are ignored.
When no `allowed-tools` are given, the skills has the same tool permissions has the main conversation.
By default, the skill is invokable by the main conversation (disable-model-invocation: false) when Claude 
thinks the current task matches the `description` and invokable by the user as a /command (user-invocable: true).
