# Skill Conductor — Design Doc

**Date:** 2026-02-22
**Status:** Approved

## Problem

As the Claude Code skill ecosystem grows, users accumulate skills from multiple sources (superpowers, shaping-skills, review-loop, ui-val, etc.). These skills are flat and independent — each has its own trigger, no composition mechanism exists, and adding a new skill means manually reasoning about how it interacts with every other skill.

Specific pain points:
- **Trigger conflicts** — brainstorming and shaping both fire on "build a feature"
- **Sequencing is implicit** — nothing declares "shaping BEFORE writing-plans"
- **Composition is manual** — ui-val should plug into code-review, but they're peers
- **Adding a skill means editing the router** — every new skill means updating `using-superpowers`

## Solution

A **skill composition framework** consisting of:
1. A pipeline config (`pipelines.yaml`) that defines phases, skill mappings, and named workflows
2. A conductor skill that replaces `using-superpowers` as the runtime router
3. An aggregator repo that vendors external skills via git submodules

The framework is consumed by the author's own workflow as the first use case.

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Scope | Framework + personal workflow | Build the composition layer, use it ourselves |
| Platform | Claude Code first | Skills, hooks, Skill tool. Other platforms later |
| Packaging | Aggregator + dependencies | Our repo wires together external skills via git submodules |
| Skill types | Phases + modifiers | New lifecycle phases AND enhancers that attach to phases |
| Routing | Runtime router | Conductor skill reads pipelines.yaml at conversation time |
| Versioning | Pin vendor versions | Git submodules locked to specific commits/tags |
| Architecture | Approach A — Pipeline Config + Router Skill | Separates composition logic from skills themselves |

## Project Structure

```
skill-conductor/
├── pipelines.yaml          # Pipeline definitions
├── skills/
│   └── conductor/          # The master router skill
│       └── skill.md
├── adapters/               # Thin wrappers for skills needing interface bridging
├── vendor/                 # Git submodules pinned to specific commits
│   ├── superpowers/
│   ├── shaping-skills/
│   └── review-loop/
├── vendor.lock             # Human-readable: vendor name → pinned version + reason
├── install.sh              # Symlinks everything into ~/.claude/skills/
├── uninstall.sh            # Removes symlinks, restores previous state
└── docs/plans/
```

## Pipeline Config (pipelines.yaml)

### Phases

Ordered slots in a development lifecycle:

| Phase | Order | Description |
|-------|-------|-------------|
| explore | 10 | Understand the problem space |
| shape | 20 | Define requirements and solution approaches |
| plan | 30 | Create implementation plan |
| build | 40 | Implement the solution |
| verify | 50 | Verify correctness and quality |
| review | 60 | Independent review |
| finish | 70 | Integrate and ship |

### Skill Types

- **Phase skills** — Run as a step in the pipeline sequence
- **Modifier skills** — Trigger conditionally based on file changes or context (e.g., ui-val after .tsx edits)
- **Always-available skills** — Can be invoked at any point regardless of current phase (e.g., think-different, learning, debugging)

### Skill Mappings

| Skill | Source | Phase | Type |
|-------|--------|-------|------|
| brainstorming | vendor/superpowers | explore | phase |
| shaping | vendor/shaping-skills | shape | phase |
| breadboarding | vendor/shaping-skills | shape | phase |
| writing-plans | vendor/superpowers | plan | phase |
| test-driven-development | vendor/superpowers | build | phase |
| systematic-debugging | vendor/superpowers | build | phase |
| verification-before-completion | vendor/superpowers | verify | phase |
| ui-val | external | verify | modifier (*.tsx, *.css, *.html, *.vue, *.svelte) |
| review-loop | vendor/review-loop | review | phase |
| requesting-code-review | vendor/superpowers | review | phase |
| finishing-a-development-branch | vendor/superpowers | finish | phase |

### Always-Available Skills

- think-different
- learning
- dispatching-parallel-agents
- subagent-driven-development
- using-git-worktrees
- executing-plans
- receiving-code-review
- writing-skills

### Pipelines

**small-fix** — Bug fix or small tweak with clear approach
```
build → verify → finish
```
- build: systematic-debugging, test-driven-development
- verify: verification-before-completion
- finish: finishing-a-development-branch

**feature** — New feature with clear requirements
```
explore → plan → build → verify → review → finish
```
- explore: brainstorming
- plan: writing-plans
- build: test-driven-development
- verify: verification-before-completion
- review: requesting-code-review
- finish: finishing-a-development-branch

**complex** — Multi-approach problem, 0-to-1, or significant architectural work
```
explore → shape → plan → build → verify → review → finish
```
- explore: brainstorming
- shape: shaping, breadboarding
- plan: writing-plans
- build: test-driven-development
- verify: verification-before-completion
- review: review-loop
- finish: finishing-a-development-branch

## Conductor Skill

Replaces `using-superpowers` as the single entry-point skill loaded at conversation start.

### Responsibilities

1. **Classify the task** — Determine which pipeline applies. Present the choice to the user for confirmation.
2. **Track phase progression** — Know which phase we're in, which skills have run, what's next.
3. **Invoke skills in order** — At each phase transition, invoke the next skill(s) via the Skill tool.
4. **Handle modifiers** — After file changes, check if any modifier skills should trigger.
5. **Allow escalation** — Mid-feature, escalate from `feature` to `complex` pipeline without losing context.
6. **Keep always-available skills accessible** — Skills like think-different, learning, debugging can be invoked at any point.

### Non-responsibilities

- Does NOT replace or modify underlying skills — just sequences them
- Does NOT enforce rigid gates — user can skip phases or jump around
- Does NOT manage skill installation — that's install.sh's job

### Classification Prompt

When a task arrives, the conductor presents:

> This looks like a **feature** task. I'd follow this pipeline:
> `brainstorming → writing-plans → TDD → verify → code-review → finish`
>
> Should I proceed with this, or would you classify it differently?

User can override. Once confirmed, the conductor tracks state and drives transitions.

## Adapters

Some external skills don't fit cleanly into "invoke via Skill tool." Adapters bridge the gap.

Example: `review-loop` uses hooks and Codex. An adapter wraps the setup and translates it into a phase-compatible skill:

```
adapters/review-loop-adapter/
└── skill.md    # Wraps review-loop invocation for the conductor's review phase
```

Adapters are thin — they don't reimplement the skill, they bridge the interface.

## Install & Wiring

### install.sh

1. Clones/updates git submodules in `vendor/`
2. Symlinks all skills into `~/.claude/skills/` (conductor + vendor skills + adapters)
3. Registers the conductor as the entry-point skill

### Conflict Handling

If `~/.claude/skills/` already has a skill with the same name, the installer warns and asks whether to override or skip. Vendor-pinned versions take precedence when confirmed.

### uninstall.sh

Removes symlinks and restores previous state.

## Versioning

Git submodules pinned to specific commits/tags. `vendor.lock` records:
- Vendor name
- Pinned version/commit
- Reason for pinning
- Known incompatibilities
