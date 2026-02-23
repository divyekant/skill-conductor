# Skill Conductor

A skill composition framework for [Claude Code](https://claude.com/claude-code). Compose skills from multiple sources into configurable pipelines with a runtime router.

## The Problem

As the Claude Code skill ecosystem grows, you accumulate skills from different sources — [superpowers](https://github.com/anthropics/superpowers), [shaping-skills](https://github.com/rjs/shaping-skills), [review-loop](https://github.com/hamelsmu/claude-review-loop), and more. Each works great on its own, but:

- **Trigger conflicts** — brainstorming and shaping both fire on "build a feature"
- **No sequencing** — nothing declares "shaping BEFORE writing-plans"
- **No composition** — ui-val should plug into verification, but skills are peers
- **Manual wiring** — every new skill means figuring out how it fits

## The Solution

A **conductor skill** that replaces `using-superpowers` as the entry point. It reads a `pipelines.yaml` config and:

1. **Classifies** your task (small-fix, feature, complex)
2. **Selects** the right pipeline
3. **Sequences** skills through phases in order
4. **Handles** escalation, modifiers, and user overrides

## Pipelines

Three built-in pipelines (customizable in `pipelines.yaml`):

```
small-fix:   build → verify → finish
feature:     explore → plan → build → verify → review → finish
complex:     explore → shape → plan → build → verify → review → finish
```

Each phase maps to specific skills:

| Phase | Skills |
|-------|--------|
| explore | brainstorming |
| shape | shaping, breadboarding |
| plan | writing-plans |
| build | test-driven-development, systematic-debugging |
| verify | verification-before-completion, ui-val (modifier) |
| review | requesting-code-review or review-loop |
| finish | finishing-a-development-branch |

## Prerequisites

- [Claude Code](https://claude.com/claude-code) CLI
- [Superpowers](https://github.com/anthropics/superpowers) plugin installed
- Git (for submodule management)

## Install

```bash
git clone --recursive https://github.com/divyekant/skill-conductor.git ~/.local/share/skill-conductor
cd ~/.local/share/skill-conductor
./install.sh
```

This symlinks the conductor and vendor skills into `~/.claude/skills/`. Superpowers skills are referenced by name from the plugin system — they don't need symlinking.

### What gets installed

| Skill | Source |
|-------|--------|
| conductor | This repo |
| shaping | [rjs/shaping-skills](https://github.com/rjs/shaping-skills) |
| breadboarding | [rjs/shaping-skills](https://github.com/rjs/shaping-skills) |
| breadboard-reflection | [rjs/shaping-skills](https://github.com/rjs/shaping-skills) |

Existing skills in `~/.claude/skills/` are left untouched. If a name conflicts, the installer asks before overriding.

## Uninstall

```bash
cd ~/.local/share/skill-conductor
./uninstall.sh
```

Removes only the symlinks this project created. Restores backups if any were made during install.

## Usage

Start a new Claude Code session. The conductor loads automatically and waits for your task.

```
You: Add a search bar to the dashboard

Claude: This looks like a **feature** task. I'd follow this pipeline:
       explore → plan → build → verify → review → finish

       Should I proceed, or would you classify it differently?

You: Proceed

Claude: [invokes brainstorming skill...]
```

### User overrides

You're always in control:

- **Skip phases:** "Skip brainstorming, I know what I want"
- **Jump ahead:** "Let's go straight to planning"
- **Change pipeline:** "Actually, let's shape this" (escalates feature → complex)
- **Invoke directly:** Call any skill by name at any time

### Escalation

If during a feature pipeline Claude (or you) discover multiple viable approaches, the conductor suggests escalating to the complex pipeline — adding a shaping phase without losing prior work.

## Customization

### Add a new skill

1. Add an entry to `pipelines.yaml` under `skills:`
2. Symlink the skill into `~/.claude/skills/`
3. Reference it in the appropriate pipeline

No adapters needed — Claude reads any skill directly.

### Create a new pipeline

Add a new entry under `pipelines:` in `pipelines.yaml`:

```yaml
pipelines:
  quick-prototype:
    description: Fast prototype without formal review
    phases: [explore, build, verify]
    skills:
      explore:
        - brainstorming
      build:
        - test-driven-development
      verify:
        - verification-before-completion
```

### Modifier skills

Skills that trigger conditionally based on file changes:

```yaml
skills:
  ui-val:
    source: external
    phase: verify
    type: modifier
    triggers:
      files-changed:
        - "*.tsx"
        - "*.css"
```

## Architecture

```
skill-conductor/
├── pipelines.yaml          # Pipeline definitions (phases, skills, workflows)
├── skills/
│   └── conductor/          # The runtime router skill
│       └── skill.md
├── vendor/                 # Git submodules (pinned versions)
│   ├── shaping-skills/
│   └── review-loop/
├── vendor.lock             # Human-readable version pins
├── install.sh              # Symlinks skills into ~/.claude/skills/
└── uninstall.sh            # Removes symlinks
```

### Key concepts

- **Phases** — Ordered slots in the dev lifecycle (explore → shape → plan → build → verify → review → finish)
- **Phase skills** — Run as a step in the pipeline sequence
- **Modifier skills** — Trigger conditionally (e.g., ui-val after .tsx edits)
- **Always-available skills** — Can be invoked at any point (think-different, learning, etc.)
- **Pipelines** — Named sequences that pick which phases and skills to use

### Vendor management

External skills are pinned via git submodules. `vendor.lock` records the pinned version and context for each dependency. Update a vendor:

```bash
cd vendor/shaping-skills
git pull origin main
cd ../..
git add vendor/shaping-skills
git commit -m "chore: update shaping-skills"
```

## Credits

Built on top of:

- [superpowers](https://github.com/anthropics/superpowers) by Anthropic — the skill framework and core dev skills
- [shaping-skills](https://github.com/rjs/shaping-skills) by [@rjs](https://github.com/rjs) — Shape Up methodology for Claude Code
- [claude-review-loop](https://github.com/hamelsmu/claude-review-loop) by [@hamelsmu](https://github.com/hamelsmu) — automated Codex review loop

## License

MIT
