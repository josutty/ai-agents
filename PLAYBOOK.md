# Playbook — how to run the agent team

## One-time setup (per machine)
1. Copy the agent files into your OpenCode agents folder (project `.opencode/agent/` or the global
   one — check your OpenCode version's docs for the exact folder name). **Rename them**: OpenCode
   uses the file name as the agent name, and the orchestrator dispatches by these names:

| File | Save as |
|---|---|
| 00b-hybrid-api-config.md | hybrid-api-config.md |
| 01-schema-parser.md | schema-parser.md |
| 02-wireframe-analyzer.md | wireframe-analyzer.md |
| 03-fsd-planner.md | fsd-planner.md |
| 04-architect-enhanced.md | architect.md |
| 05-store-architect.md | store-architect.md |
| 5_5-mock-data-generator.md | mock-data-generator.md |
| 06-component-generator.md | component-generator.md |
| 07-styling-engineer.md | styling-engineer.md |
| 08-test-engineer.md | test-engineer.md |
| 09-coder-enhanced.md | coder.md |
| 10-reviewer-enhanced.md | reviewer.md |
| 11-orchestrator-enhanced.md | orchestrator.md |
| 12-app-bootstrap.md | app-bootstrap.md |
| 13-researcher.md | researcher.md |
| 14-reflector.md | reflector.md |
| 15-debugger.md | debugger.md |
| 16-codebase-scanner.md | codebase-scanner.md |
| 17-coverage-checker.md | coverage-checker.md |
2. Copy `AGENTS.md` into the ROOT of every project repo you run the team on.
3. Select the **orchestrator** as the primary agent. You only ever talk to the orchestrator.

## Put inputs in the same place every time
```
inputs/
├── fsd-spec.md          client's FSD document (any format → save as .md)
├── ux/                  one .html per screen (products.html, cart.html…)
└── api/backend-api.yaml backend team's YAML (partial or complete)
```

## Mode 1 — new project
```
Mode: new-project.
Inputs: inputs/fsd-spec.md, inputs/ux/*.html, inputs/api/backend-api.yaml (partial).
Branch name: feat/shop-v1.
Run Phase 0 and Phase 1 only, then stop at Gate 1.
```
At Gate 1: open `plan/coverage-matrix.md` first → every gap either fixed or marked out-of-scope by you.
Send `plan/generated/backend-gaps.md` to the backend team.
Then: `Gate 1 approved. Run Phase 2 and stop at Gate 2.` → review tasks →
`Gate 2 approved. Run Phase 3 and 4.`

Tip for your first project: start with 1–2 screens. Look at every file the agents write.

## Mode 2 — something is missing (project built by this team)
Found it yourself:
```
Mode: gap-fill. The Products page has no sort dropdown (it's in inputs/ux/products.html),
and the Empty state on the catalog never shows.
```
Backend sent a new YAML:
```
Mode: gap-fill. New backend spec: inputs/api/backend-api-v2.yaml. Update the project.
```
Not sure what's missing:
```
Mode: gap-fill. Run coverage-checker at stage=release and show me the gaps before doing anything.
```

## Mode 3 — new feature in an existing project (any project)
```
Mode: feature. Add a "Wishlist" feature: heart button on product cards, wishlist page.
UX: inputs/ux/wishlist.html. API: inputs/api/wishlist.yaml (partial).
```
The first run on a project not built by this team creates `PROJECT.md`.
**Read PROJECT.md before approving anything** — if it describes the project wrongly, every agent
will follow the wrong rules. Correct it by telling the orchestrator what's wrong.

## Mode 4 — bug fix
Give steps, expected, actual:
```
Mode: bugfix.
Bug: on /products, when the list fails to load, clicking Retry does nothing.
Steps: set VITE_API_MODE=mock, block /api/products in devtools, reload, click Retry.
Expected: list reloads. Actual: nothing happens, no network request.
```
The team diagnoses first, writes a test that fails because of the bug, then fixes it.

**When NOT to use the team:** a one-line fix, a text change, or renaming something. Use a single
coding assistant chat instead — with AGENTS.md (and PROJECT.md) in the repo it follows the same rules.

## Useful commands to the orchestrator
- `Where are we?` — it reads plan/PROGRESS.md and reports.
- `Resume.` — continues from the last HANDOFF after a crash or a new session.
- `Stop after the current TASK.` — safe pause point.
- `Show me the gaps.` — runs coverage-checker.

## What to check at each gate (5-minute version)
| Gate | Open these | Ask yourself |
|---|---|---|
| 1 | coverage-matrix.md, analysis/components.md, backend-gaps.md, fsd-structure.md compliance table | Is every screen element there? Does placement follow the client's FSD doc? |
| 2 | PROGRESS.md, 2–3 TASK files | Is each TASK small? Are the props and states what the UX shows? |
| 3 | reviewer result, coverage-matrix.md | Run `npm run dev` and click through every screen, light + dark. |
