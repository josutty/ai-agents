# Manual run sheet — OpenCode (you are the orchestrator)

Put this file in your project root and keep it open next to OpenCode.

## How every step works
1. Start a **new session** for each agent (`/new`), so instructions don't mix.
2. Press **Tab** until the bottom line shows **Build**.
3. Copy the prompt for the step, fill in the `< >` parts, send.
4. When it finishes, read its **last line** (the HANDOFF) and use the table below.

Every prompt starts with these two lines (already included below):
```
Follow @.opencode/agents/<AGENT>.md exactly, and @AGENTS.md.
You are ONLY this agent: edit only what its file allows, don't do the next agent's work, finish with the HANDOFF line.
```

## Shorter way to call an agent
If `@<agent-name>` works in your OpenCode (subagents are listed there), you can type just:
```
@wireframe-analyzer Mode: new-project. UX HTML: inputs/ux/**/*.html. Client FSD doc: inputs/fsd-spec.md.
```
The agent's own file already says what to do; the long prompts below are the safest form on weaker models.
`lite` and `fast` are orchestrator settings — when you run agents by hand you simply skip the steps yourself
(e.g. no reviewer after a purely visual task, one review per 4 tasks).

## What the last line means
| HANDOFF says | What you do |
|---|---|
| DONE / READY / PASS / LOGGED / FIXED | go to the next step |
| CONCERNS | write it down for the final check, then next step |
| NEEDS-RESEARCH | run **researcher** with that reason, then run the same agent again |
| BLOCKED | read the reason, fix what it asks (a file, a decision), run the same agent again |
| BLOCKED-TEST | run **test-engineer**: "fix the test named in: <reason>", then the agent that complained |
| BLOCKED-BUG | run **debugger** `mode=fix` for that task, then **coder** again |
| FAIL-MECH | run **coder** (or component-generator) again: "fix the reviewer's findings in plan/history/<TASK>.md" |
| anything ending in -DESIGN | run **reflector**, then **architect** with the reflection, then restart the task at test-engineer |

You are the guard: never approve `git push`, `rm`, `git add .`, or edits outside the agent's job.

---

## A. New project

### Setup
**app-bootstrap**
```
Follow @.opencode/agents/app-bootstrap.md exactly, and @AGENTS.md.
You are ONLY this agent: edit only what its file allows, don't do the next agent's work, finish with the HANDOFF line.
Mode: new-project. Run 1 (Phase 0). Branch name: feat/<project-name>.
```

### Planning (one after another)
**wireframe-analyzer**
```
Follow @.opencode/agents/wireframe-analyzer.md exactly, and @AGENTS.md.
You are ONLY this agent: edit only what its file allows, don't do the next agent's work, finish with the HANDOFF line.
Mode: new-project. UX HTML: inputs/ux/**/*.html. Client FSD doc: inputs/fsd-spec.md.
```
**hybrid-api-config**
```
Follow @.opencode/agents/hybrid-api-config.md exactly, and @AGENTS.md.
You are ONLY this agent: edit only what its file allows, don't do the next agent's work, finish with the HANDOFF line.
Mode: new-project. Backend spec: inputs/api/backend-api.yaml (partial).
```
**schema-parser**
```
Follow @.opencode/agents/schema-parser.md exactly, and @AGENTS.md.
You are ONLY this agent: edit only what its file allows, don't do the next agent's work, finish with the HANDOFF line.
Mode: new-project. Spec: plan/generated/api-spec.yaml.
```
**mock-data-generator**
```
Follow @.opencode/agents/mock-data-generator.md exactly, and @AGENTS.md.
You are ONLY this agent: edit only what its file allows, don't do the next agent's work, finish with the HANDOFF line.
Mode: new-project.
```
**fsd-planner**
```
Follow @.opencode/agents/fsd-planner.md exactly, and @AGENTS.md.
You are ONLY this agent: edit only what its file allows, don't do the next agent's work, finish with the HANDOFF line.
Mode: new-project. Client FSD doc: inputs/fsd-spec.md.
```
**coverage-checker**
```
Follow @.opencode/agents/coverage-checker.md exactly, and @AGENTS.md.
You are ONLY this agent: edit only what its file allows, don't do the next agent's work, finish with the HANDOFF line.
Mode: new-project. stage=plan. UX HTML: inputs/ux/**/*.html.
```

### 🚪 Gate 1 — you check
Open `plan/coverage-matrix.md` (gaps first), `analysis/components.md`, `analysis/capabilities.md`,
`plan/library-decisions.md` (approve the libraries), `analysis/display-rules.md` (answer any open question),
`plan/fsd-structure.md`, `plan/generated/backend-gaps.md`. Something wrong → run the agent that wrote that file again, telling it what's wrong.

### Install the approved libraries (after you approve Gate 1)
**app-bootstrap** (Run L)
```
Follow @.opencode/agents/app-bootstrap.md exactly, and @AGENTS.md.
You are ONLY this agent: edit only what its file allows, don't do the next agent's work, finish with the HANDOFF line.
Mode: new-project. Run L (install only the approved rows of plan/library-decisions.md).
```

### Tasks
**architect**
```
Follow @.opencode/agents/architect.md exactly, and @AGENTS.md.
You are ONLY this agent: edit only what its file allows, don't do the next agent's work, finish with the HANDOFF line.
Mode: new-project. Phase 2.
```
**coverage-checker**
```
Follow @.opencode/agents/coverage-checker.md exactly, and @AGENTS.md.
You are ONLY this agent: edit only what its file allows, don't do the next agent's work, finish with the HANDOFF line.
Mode: new-project. stage=tasks.
```

### 🚪 Gate 2 — you check
Open `plan/PROGRESS.md` and 2–3 files in `plan/tasks/`. Small tasks? Props and states match the screens?

### Foundation
**store-architect**
```
Follow @.opencode/agents/store-architect.md exactly, and @AGENTS.md.
You are ONLY this agent: edit only what its file allows, don't do the next agent's work, finish with the HANDOFF line.
Mode: new-project. Phase 3a.
```
**styling-engineer**
```
Follow @.opencode/agents/styling-engineer.md exactly, and @AGENTS.md.
You are ONLY this agent: edit only what its file allows, don't do the next agent's work, finish with the HANDOFF line.
Mode: new-project. Phase 3a.
```

### Build — repeat for EVERY task in `plan/PROGRESS.md`, top to bottom
**Which agents does a task need?** Open `plan/tasks/TASK-NNN.md` → the **Files** table → the
**owner** column. Run only the agents named there, always in this order:

**test-engineer** (always first)
```
Follow @.opencode/agents/test-engineer.md exactly, and @AGENTS.md.
You are ONLY this agent: edit only what its file allows, don't do the next agent's work, finish with the HANDOFF line.
Mode: new-project. TASK-001.
```
**component-generator** (only if it owns rows)
```
Follow @.opencode/agents/component-generator.md exactly, and @AGENTS.md.
You are ONLY this agent: edit only what its file allows, don't do the next agent's work, finish with the HANDOFF line.
Mode: new-project. TASK-001.
```
**coder** (only if it owns rows)
```
Follow @.opencode/agents/coder.md exactly, and @AGENTS.md.
You are ONLY this agent: edit only what its file allows, don't do the next agent's work, finish with the HANDOFF line.
Mode: new-project. TASK-001.
```
**reviewer** (always last)
```
Follow @.opencode/agents/reviewer.md exactly, and @AGENTS.md.
You are ONLY this agent: edit only what its file allows, don't do the next agent's work, finish with the HANDOFF line.
Mode: new-project. TASK-001. scope=task.
```
**app-bootstrap** (only if the task created a page in `src/pages/`)
```
Follow @.opencode/agents/app-bootstrap.md exactly, and @AGENTS.md.
You are ONLY this agent: edit only what its file allows, don't do the next agent's work, finish with the HANDOFF line.
Mode: new-project. Run 2, incremental.
```
Then the next task (TASK-002 …).

### Finish
**app-bootstrap** → `Mode: new-project. Run 2, final.`
**reviewer** → `Mode: new-project. scope=release.`
**coverage-checker** → `Mode: new-project. stage=release. UX HTML: inputs/ux/**/*.html.`
(each with the same two first lines)

### 🚪 Gate 3 — you check, then push
`npm run dev`: click every screen in light + dark, then once with the keyboard only. Then push the branch.

---

## B. Feature (existing project)
Same two first lines on every prompt. In order:
1. **codebase-scanner** → `Mode: feature.`
2. **app-bootstrap** → `Mode: feature. Run B. Branch name: feat/<feature-name>.`
3. Only if there's a NEW API file: **hybrid-api-config** → `Mode: feature. delta=true. Backend spec: inputs/api/<file>.yaml`,
   then **schema-parser** → `Mode: feature. delta=true. Spec: plan/generated/api-spec.yaml`,
   then **mock-data-generator** → `Mode: feature. delta=true.`
4. Only if there are NEW screens: **wireframe-analyzer** → `Mode: feature. delta=true. UX HTML: inputs/ux/<new files>`
5. **fsd-planner** → `Mode: feature. Feature: <what to add>.`
6. **coverage-checker** → `Mode: feature. stage=plan.` (only with new screens)
7. 🚪 Gate 1
8. **architect** → `Mode: feature. Feature: <what to add>.`
9. 🚪 Gate 2
10. **store-architect** → `Mode: feature. delta=true.` (only if new endpoints or shared state)
    **styling-engineer** → `Mode: feature. delta=true.` (only if new colors/tokens)
11. Build loop for every new task (same as A, with `Mode: feature.`)
12. **app-bootstrap** → `Mode: feature. Run 2, final.` (only if new pages) → **reviewer** `scope=release` → **coverage-checker** `stage=release` → 🚪 Gate 3

## C. Bug fix
Same two first lines on every prompt. In order:
1. **codebase-scanner** → `Mode: bugfix.`
2. **app-bootstrap** → `Mode: bugfix. Run B. Branch name: fix/<short-name>.`
3. **debugger** → `Mode: bugfix. mode=diagnose. Bug: <steps>. Expected: <…>. Actual: <…>.`
4. **architect** → `Mode: bugfix. Diagnosis: plan/history/BUG-NNN.md.`
5. **test-engineer** → `Mode: bugfix. BUG-NNN.` (writes a test that fails because of the bug)
6. **coder** → `Mode: bugfix. BUG-NNN.`
7. **reviewer** → `Mode: bugfix. BUG-NNN. scope=task.` → 🚪 Gate 3

## D. Quick change (max 3 files)
1. **codebase-scanner** → `Mode: quick.`
2. **app-bootstrap** → `Mode: quick. Run B. Branch name: feat/<short-name>.`
3. **architect** → `Mode: quick. mode=quick. Change: <what to change>.` (if it says "not quick", use B or C)
4. **test-engineer** → `Mode: quick. TASK-NNN.` (only if the task's Files table has test rows)
5. **component-generator** or **coder** → `Mode: quick. TASK-NNN.` (whichever owns the files)
6. **reviewer** → `Mode: quick. TASK-NNN. scope=task.` → 🚪 Gate 3

## E. Requirement changed (gap-fill)
Like B, with `Mode: gap-fill.` and `delta=true` everywhere. Update the input files first (edit the same
HTML file, don't create a v2), then start at step 3 or 4.

---
**Lost your place?** Open `plan/PROGRESS.md` (which tasks are done) and the end of
`plan/history/TASK-NNN.md` (the last HANDOFF line), then continue from the matching step.
