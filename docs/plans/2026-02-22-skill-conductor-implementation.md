# Skill Conductor Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a skill composition framework that aggregates skills from multiple sources into configurable pipelines with a runtime router.

**Architecture:** An aggregator repo vendors external skills via git submodules, defines pipelines in YAML, and provides a conductor skill that replaces `using-superpowers` as the entry-point router. An install script symlinks everything into `~/.claude/skills/`.

**Tech Stack:** Bash (install/uninstall scripts), YAML (pipeline config), Markdown (skill definitions)

---

### Task 1: Project Scaffolding

**Files:**
- Create: `.gitignore`
- Create: `README.md`

**Step 1: Create .gitignore**

```gitignore
.DS_Store
*.swp
*.swo
*~
```

**Step 2: Create README.md**

```markdown
# Skill Conductor

A skill composition framework for Claude Code. Aggregates skills from multiple sources into configurable pipelines with a runtime router.

## Quick Start

```bash
./install.sh
```

## Documentation

- [Design Doc](docs/plans/2026-02-22-skill-conductor-design.md)
- [Implementation Plan](docs/plans/2026-02-22-skill-conductor-implementation.md)
```

**Step 3: Commit**

```bash
git add .gitignore README.md
git commit -m "chore: add gitignore and readme"
```

---

### Task 2: Add Vendor Submodules

**Files:**
- Create: `vendor/` (via git submodule)

**Step 1: Add superpowers submodule**

Note: superpowers is installed via Claude Code's plugin system at `~/.claude/plugins/cache/claude-plugins-official/superpowers/`. We don't vendor it as a submodule — it's managed by the plugin system. The conductor references its skills by name; they're already available via the Skill tool.

So we only submodule the skills NOT already available through the plugin system.

```bash
git submodule add https://github.com/rjs/shaping-skills.git vendor/shaping-skills
```

**Step 2: Add review-loop submodule**

```bash
git submodule add https://github.com/hamelsmu/claude-review-loop.git vendor/review-loop
```

**Step 3: Pin submodules to current HEAD**

```bash
cd vendor/shaping-skills && git log --oneline -1 && cd ../..
cd vendor/review-loop && git log --oneline -1 && cd ../..
```

Record the commits in vendor.lock (next task).

**Step 4: Commit**

```bash
git add .gitmodules vendor/
git commit -m "chore: add vendor submodules for shaping-skills and review-loop"
```

---

### Task 3: Create vendor.lock

**Files:**
- Create: `vendor.lock`

**Step 1: Write vendor.lock**

After running Task 2, use the actual commit hashes from the submodules.

```markdown
# Vendor Lock
# Records pinned versions for external skill dependencies.
# Git submodules do the actual pinning; this file is human-readable context.

## shaping-skills
- **Repo:** https://github.com/rjs/shaping-skills
- **Pinned:** <commit-hash> (initial pin, 2026-02-22)
- **Skills provided:** shaping, breadboarding, breadboard-reflection
- **Notes:** Also includes a PostToolUse hook (shaping-ripple.sh) — not installed by default

## review-loop
- **Repo:** https://github.com/hamelsmu/claude-review-loop
- **Pinned:** <commit-hash> (initial pin, 2026-02-22)
- **Skills provided:** review-loop (via adapter)
- **Notes:** Requires Codex CLI (`npm install -g @openai/codex`) and jq. Uses hooks + Codex multi-agent. Needs adapter to work as a conductor phase.

## superpowers (not vendored)
- **Source:** Claude Code plugin system (~/.claude/plugins/cache/claude-plugins-official/superpowers/)
- **Version:** 4.3.1
- **Skills provided:** brainstorming, writing-plans, test-driven-development, systematic-debugging, verification-before-completion, requesting-code-review, finishing-a-development-branch, think-different, learning, dispatching-parallel-agents, subagent-driven-development, using-git-worktrees, executing-plans, receiving-code-review, writing-skills
- **Notes:** Managed by plugin system, not vendored. Conductor references these skills by name.
```

Replace `<commit-hash>` with actual hashes from Task 2.

**Step 2: Commit**

```bash
git add vendor.lock
git commit -m "chore: add vendor.lock with pinned versions"
```

---

### Task 4: Create pipelines.yaml

**Files:**
- Create: `pipelines.yaml`

**Step 1: Write pipelines.yaml**

```yaml
# Skill Conductor Pipeline Configuration
# Defines phases, skill mappings, and named workflows.

phases:
  explore:
    order: 10
    description: Understand the problem space
  shape:
    order: 20
    description: Define requirements and solution approaches
  plan:
    order: 30
    description: Create implementation plan
  build:
    order: 40
    description: Implement the solution
  verify:
    order: 50
    description: Verify correctness and quality
  review:
    order: 60
    description: Independent review
  finish:
    order: 70
    description: Integrate and ship

skills:
  # --- Explore phase ---
  brainstorming:
    source: plugin/superpowers
    phase: explore
    type: phase

  # --- Shape phase ---
  shaping:
    source: vendor/shaping-skills
    phase: shape
    type: phase

  breadboarding:
    source: vendor/shaping-skills
    phase: shape
    type: phase

  # --- Plan phase ---
  writing-plans:
    source: plugin/superpowers
    phase: plan
    type: phase

  # --- Build phase ---
  test-driven-development:
    source: plugin/superpowers
    phase: build
    type: phase

  systematic-debugging:
    source: plugin/superpowers
    phase: build
    type: phase

  # --- Verify phase ---
  verification-before-completion:
    source: plugin/superpowers
    phase: verify
    type: phase

  ui-val:
    source: external
    phase: verify
    type: modifier
    triggers:
      files-changed:
        - "*.tsx"
        - "*.css"
        - "*.html"
        - "*.vue"
        - "*.svelte"

  # --- Review phase ---
  review-loop:
    source: vendor/review-loop
    phase: review
    type: phase
    adapter: review-loop-adapter

  requesting-code-review:
    source: plugin/superpowers
    phase: review
    type: phase

  # --- Finish phase ---
  finishing-a-development-branch:
    source: plugin/superpowers
    phase: finish
    type: phase

always-available:
  - think-different
  - learning
  - dispatching-parallel-agents
  - subagent-driven-development
  - using-git-worktrees
  - executing-plans
  - receiving-code-review
  - writing-skills

pipelines:
  small-fix:
    description: Bug fix or small tweak with clear approach
    phases: [build, verify, finish]
    skills:
      build:
        - systematic-debugging
        - test-driven-development
      verify:
        - verification-before-completion
      finish:
        - finishing-a-development-branch

  feature:
    description: New feature with clear requirements
    phases: [explore, plan, build, verify, review, finish]
    skills:
      explore:
        - brainstorming
      plan:
        - writing-plans
      build:
        - test-driven-development
      verify:
        - verification-before-completion
      review:
        - requesting-code-review
      finish:
        - finishing-a-development-branch

  complex:
    description: Multi-approach problem, 0-to-1, or significant architectural work
    phases: [explore, shape, plan, build, verify, review, finish]
    skills:
      explore:
        - brainstorming
      shape:
        - shaping
        - breadboarding
      plan:
        - writing-plans
      build:
        - test-driven-development
      verify:
        - verification-before-completion
      review:
        - review-loop
      finish:
        - finishing-a-development-branch
```

**Step 2: Commit**

```bash
git add pipelines.yaml
git commit -m "feat: add pipeline configuration with phases, skills, and workflows"
```

---

### Task 5: Write the Conductor Skill

This is the core deliverable — the runtime router that replaces `using-superpowers`.

**Files:**
- Create: `skills/conductor/skill.md`

**Step 1: Write skill.md**

The conductor skill is a markdown file with YAML frontmatter. It contains instructions for Claude to:
1. Read pipelines.yaml at conversation start
2. Classify the incoming task
3. Present the pipeline choice to the user
4. Track phase progression and invoke skills in order
5. Handle modifiers and escalation

```markdown
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
```

**Step 2: Verify the skill loads correctly**

After install (Task 8), test by starting a new Claude Code session and checking that the conductor skill appears in the available skills list.

**Step 3: Commit**

```bash
git add skills/conductor/skill.md
git commit -m "feat: add conductor skill — runtime router for skill pipelines"
```

---

### Task 6: Write the Review-Loop Adapter

**Files:**
- Create: `adapters/review-loop-adapter/skill.md`

**Step 1: Create adapters directory**

```bash
mkdir -p adapters/review-loop-adapter
```

**Step 2: Write adapter skill.md**

This adapter wraps review-loop's setup process into a skill that the conductor can invoke during the review phase.

```markdown
---
name: review-loop-adapter
description: "Adapter for hamelsmu/claude-review-loop. Sets up an automated Codex review loop during the review phase. Requires Codex CLI and jq."
---

# Review Loop (Adapter)

This skill adapts the [claude-review-loop](https://github.com/hamelsmu/claude-review-loop) plugin for use as a conductor pipeline phase.

## Prerequisites

Before using this skill, ensure:
- **Codex CLI** is installed: `npm install -g @openai/codex`
- **jq** is installed: `brew install jq`

If either is missing, inform the user and skip this phase.

## Process

When invoked during the review phase:

1. **Check prerequisites** — Run `command -v codex` and `command -v jq` to verify both are installed.

2. **Generate review ID** — Create a unique identifier for this review session:
   ```bash
   REVIEW_ID="$(date +%Y%m%d-%H%M%S)-$(openssl rand -hex 3)"
   ```

3. **Ensure Codex multi-agent is enabled** — Check `~/.codex/config.toml` for `multi_agent = true` under `[features]`. Add it if missing.

4. **Run Codex review** — Use the Codex MCP tool (mcp__codex__codex) to run a code review on the current diff:
   ```
   Review the changes in this project. Run git diff to see what changed.
   Focus on: code quality, test coverage, security (OWASP top 10), architecture.
   Write your review to reviews/review-<REVIEW_ID>.md
   ```

5. **Present review results** — Read the generated review file and present findings to the user.

6. **Address feedback** — If the review identifies issues, work with the user to address them before moving to the next phase.

## Fallback

If Codex is not available, fall back to the `requesting-code-review` skill from superpowers (invoke it via the Skill tool). Inform the user:

> Codex CLI not found — falling back to manual code review via requesting-code-review skill.
```

**Step 3: Commit**

```bash
git add adapters/review-loop-adapter/
git commit -m "feat: add review-loop adapter for conductor pipeline integration"
```

---

### Task 7: Write install.sh

**Files:**
- Create: `install.sh`

**Step 1: Write install.sh**

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
SKILLS_DIR="${HOME}/.claude/skills"

echo "=== Skill Conductor Installer ==="
echo ""

# 1. Initialize/update git submodules
echo "[1/3] Updating vendor submodules..."
cd "$SCRIPT_DIR"
git submodule update --init --recursive
echo "  Done."

# 2. Ensure skills directory exists
mkdir -p "$SKILLS_DIR"

# 3. Define what to symlink
declare -A SKILL_LINKS=(
    ["conductor"]="$SCRIPT_DIR/skills/conductor"
    ["shaping"]="$SCRIPT_DIR/vendor/shaping-skills/shaping"
    ["breadboarding"]="$SCRIPT_DIR/vendor/shaping-skills/breadboarding"
    ["breadboard-reflection"]="$SCRIPT_DIR/vendor/shaping-skills/breadboard-reflection"
    ["review-loop-adapter"]="$SCRIPT_DIR/adapters/review-loop-adapter"
)

# 4. Create symlinks
echo ""
echo "[2/3] Symlinking skills to $SKILLS_DIR..."

for skill_name in "${!SKILL_LINKS[@]}"; do
    source_path="${SKILL_LINKS[$skill_name]}"
    target_path="$SKILLS_DIR/$skill_name"

    if [ -L "$target_path" ]; then
        # Already a symlink — check if it points to us
        existing_target="$(readlink "$target_path")"
        if [ "$existing_target" = "$source_path" ]; then
            echo "  $skill_name: already linked (skipped)"
            continue
        else
            echo "  $skill_name: symlink exists → $existing_target"
            read -p "    Override? [y/N] " -n 1 -r
            echo
            if [[ $REPLY =~ ^[Yy]$ ]]; then
                rm "$target_path"
            else
                echo "    Skipped."
                continue
            fi
        fi
    elif [ -e "$target_path" ]; then
        echo "  $skill_name: exists at $target_path (not a symlink)"
        read -p "    Override? [y/N] " -n 1 -r
        echo
        if [[ $REPLY =~ ^[Yy]$ ]]; then
            echo "    Backing up to ${target_path}.bak"
            mv "$target_path" "${target_path}.bak"
        else
            echo "    Skipped."
            continue
        fi
    fi

    ln -s "$source_path" "$target_path"
    echo "  $skill_name: linked"
done

# 5. Summary
echo ""
echo "[3/3] Installation complete."
echo ""
echo "Skills installed to $SKILLS_DIR:"
for skill_name in "${!SKILL_LINKS[@]}"; do
    if [ -L "$SKILLS_DIR/$skill_name" ]; then
        echo "  ✓ $skill_name"
    else
        echo "  ✗ $skill_name (not installed)"
    fi
done
echo ""
echo "Note: superpowers skills are loaded via Claude Code plugin system (not symlinked)."
echo "The conductor skill will reference them by name at runtime."
echo ""
echo "To uninstall: ./uninstall.sh"
```

**Step 2: Make executable**

```bash
chmod +x install.sh
```

**Step 3: Commit**

```bash
git add install.sh
git commit -m "feat: add install script with conflict detection and backup"
```

---

### Task 8: Write uninstall.sh

**Files:**
- Create: `uninstall.sh`

**Step 1: Write uninstall.sh**

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
SKILLS_DIR="${HOME}/.claude/skills"

echo "=== Skill Conductor Uninstaller ==="
echo ""

SKILL_NAMES=(conductor shaping breadboarding breadboard-reflection review-loop-adapter)

for skill_name in "${SKILL_NAMES[@]}"; do
    target_path="$SKILLS_DIR/$skill_name"

    if [ -L "$target_path" ]; then
        existing_target="$(readlink "$target_path")"
        # Only remove if it points to our project
        if [[ "$existing_target" == "$SCRIPT_DIR"* ]]; then
            rm "$target_path"
            echo "  Removed: $skill_name"

            # Restore backup if it exists
            if [ -e "${target_path}.bak" ]; then
                mv "${target_path}.bak" "$target_path"
                echo "  Restored: ${skill_name}.bak → $skill_name"
            fi
        else
            echo "  Skipped: $skill_name (points to $existing_target, not ours)"
        fi
    else
        echo "  Skipped: $skill_name (not a symlink)"
    fi
done

echo ""
echo "Uninstall complete. Superpowers plugin skills are unaffected."
```

**Step 2: Make executable**

```bash
chmod +x uninstall.sh
```

**Step 3: Commit**

```bash
git add uninstall.sh
git commit -m "feat: add uninstall script with backup restoration"
```

---

### Task 9: Test the Installation

**Step 1: Run install.sh**

```bash
cd /Users/divyekant/Projects/skill-conductor
./install.sh
```

Expected: All 5 skills symlinked to `~/.claude/skills/`, no conflicts (ui-val is the only existing skill and it's not in our list).

**Step 2: Verify symlinks**

```bash
ls -la ~/.claude/skills/
```

Expected:
```
conductor -> /Users/divyekant/Projects/skill-conductor/skills/conductor
shaping -> /Users/divyekant/Projects/skill-conductor/vendor/shaping-skills/shaping
breadboarding -> /Users/divyekant/Projects/skill-conductor/vendor/shaping-skills/breadboarding
breadboard-reflection -> /Users/divyekant/Projects/skill-conductor/vendor/shaping-skills/breadboard-reflection
review-loop-adapter -> /Users/divyekant/Projects/skill-conductor/adapters/review-loop-adapter
ui-val (pre-existing, untouched)
```

**Step 3: Verify skill files are readable through symlinks**

```bash
head -5 ~/.claude/skills/conductor/skill.md
head -5 ~/.claude/skills/shaping/SKILL.md
head -5 ~/.claude/skills/breadboarding/skill.md
```

Expected: YAML frontmatter visible for each.

**Step 4: Test uninstall**

```bash
./uninstall.sh
ls -la ~/.claude/skills/
```

Expected: Only `ui-val` remains. Our 5 symlinks removed.

**Step 5: Re-install for use**

```bash
./install.sh
```

**Step 6: Commit any fixes if needed**

---

### Task 10: End-to-End Verification

**Step 1: Start a new Claude Code session**

Open a new Claude Code conversation and verify:
- The conductor skill appears in the available skills list
- Shaping, breadboarding, breadboard-reflection skills appear
- Review-loop-adapter appears

**Step 2: Test classification**

Give the conductor a task like "Add a search bar to the dashboard" and verify it:
- Classifies as `feature`
- Presents the correct pipeline
- Asks for confirmation

**Step 3: Test escalation**

During a feature pipeline, say "Actually there are multiple approaches here, let's shape this" and verify the conductor escalates to `complex`.

**Step 4: Commit final state**

```bash
cd /Users/divyekant/Projects/skill-conductor
git add -A
git commit -m "chore: finalize skill conductor v1"
```
