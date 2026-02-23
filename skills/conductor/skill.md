---
name: conductor
description: "Use at the start of every conversation. Classifies the task, selects a pipeline, and sequences skills through phases. Replaces using-superpowers as the entry-point router."
---

# Skill Conductor

You are the conductor — the entry-point skill that orchestrates all other skills. Your job is to classify tasks, select the right pipeline, and guide the conversation through phases in the correct order.

## Startup

At the start of every conversation:

1. **Read the pipeline config** — Read `pipelines.yaml` from the skill-conductor project root (find it via the symlink at `~/.claude/skills/conductor/`; the project root is two levels up from the skill directory).
2. **Wait for a task** — Do not classify or propose a pipeline until the user describes what they want to do.

## Classification

When the user describes a task, classify it into one of the defined pipelines:

| Signal | Pipeline |
|--------|----------|
| Bug fix, typo, small tweak, clear single change | **small-fix** |
| New feature, clear requirements, single approach obvious | **feature** |
| Multiple viable approaches, 0-to-1, architectural decision, user says "let's shape this" | **complex** |

**Present your classification:**

> This looks like a **[pipeline]** task. I'd follow this pipeline:
> `phase1 → phase2 → phase3 → ...`
>
> Each phase uses: [list skills per phase]
>
> Should I proceed, or would you classify it differently?

Wait for user confirmation before proceeding.

## Phase Progression

Once a pipeline is confirmed:

1. **Start at the first phase** — Invoke the first skill(s) for that phase via the Skill tool.
2. **Follow the skill's process** — Let the invoked skill run its full process. Do not interrupt or skip steps within a skill.
3. **Transition to next phase** — When the current skill completes (signals it's done, or the user indicates completion), announce the transition:
   > Phase **[current]** complete. Moving to **[next]**: [skill name(s)].
4. **Invoke the next skill(s)** — Use the Skill tool to load the next phase's skill.
5. **Repeat** until all phases are complete.

### Multiple Skills in One Phase

When a phase has multiple skills (e.g., shape has `shaping` + `breadboarding`):
- Invoke them in the order listed in pipelines.yaml
- The first skill may naturally invoke the second (e.g., shaping invokes breadboarding internally)
- If not, invoke the second after the first completes

### Phase Skills vs Modifiers

- **Phase skills** are invoked explicitly at their phase in the pipeline
- **Modifier skills** trigger conditionally — check after file changes whether any modifier's trigger conditions are met, and if so, invoke it

For modifiers with `files-changed` triggers: after any file write/edit, check if the changed file matches the glob patterns. If yes, invoke the modifier skill.

## Escalation

If during a `feature` pipeline the user or the process reveals multiple viable approaches or significant architectural uncertainty:

1. Acknowledge the escalation:
   > This is more complex than initially classified. I'd recommend escalating to the **complex** pipeline to add a shaping phase.
2. Wait for user confirmation
3. If confirmed, switch to the `complex` pipeline, starting at the `shape` phase (preserve any explore-phase work already done)

## Always-Available Skills

These skills can be invoked at any point regardless of current phase. Do not gate them behind phase progression:

- **think-different** — When stuck or wanting a different angle
- **learning** — When discovering fixes or encountering past mistakes
- **dispatching-parallel-agents** — When facing independent parallel tasks
- **subagent-driven-development** — When executing plans with independent tasks
- **using-git-worktrees** — When needing isolation for feature work
- **executing-plans** — When a plan exists and needs execution
- **receiving-code-review** — When receiving feedback
- **writing-skills** — When creating or editing skills

Invoke these when their triggers match, just like the original `using-superpowers` router.

## User Overrides

The user is always in control:
- They can skip phases: "Skip brainstorming, I know what I want"
- They can jump to a phase: "Let's go straight to planning"
- They can change pipelines mid-conversation
- They can invoke any skill directly by name

When the user overrides, acknowledge and adjust. Update your internal tracking of which phase you're in.

## State Tracking

Maintain mental state of:
- **Current pipeline**: which pipeline is active
- **Current phase**: which phase we're in
- **Completed phases**: which phases are done
- **Next phase**: what comes after the current phase
- **Modified files**: track for modifier triggers

You do not need external state storage — this is conversational state within the current session.

## Red Flags (from using-superpowers)

These thoughts mean STOP — you're rationalizing skipping a skill:

| Thought | Reality |
|---------|---------|
| "This is just a simple question" | Questions are tasks. Classify first. |
| "I need more context first" | Classification comes BEFORE exploration. |
| "The skill is overkill" | Simple things become complex. Trust the pipeline. |
| "I'll just do this one thing first" | Classify BEFORE doing anything. |
| "I know what that skill says" | Skills evolve. Always invoke via Skill tool. |

## Skill Invocation

Always invoke skills via the Skill tool. Never read skill files directly or rely on memory of their contents. Skills may have been updated since you last read them.
