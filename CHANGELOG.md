# Agent team — revision changelog

## New file: AGENTS.md
Single source of truth for conventions: stack pins, FSD import matrix, data contract, styling,
testing, quality-gate commands, git rules, sensitive-data policy, HANDOFF format, file ownership.
OpenCode loads `AGENTS.md` from the project root into every agent, so put it at the repo root.
Agent files now reference it instead of repeating (and contradicting) each other.

## Architecture decisions made (review these first)
| Decision | Was | Now | Why |
|---|---|---|---|
| Server data | hand-written thunks + normalized slices | RTK Query (`baseApi` + `injectEndpoints`) | loading/error/cache/tags built in; removes most store bugs |
| Client state | nested `entities/features` reducers | `combineSlices` + `slice.selectors` | nested plain objects in `configureStore` don't work |
| RootState | imported from `@app` everywhere | global types in `app/store/types.d.ts` | removes the upward-import FSD violation |
| Test runner | Jest + ts-jest | Vitest (config inside vite.config.ts) | same aliases/env as Vite; no ESM/jsdom/MSW polyfill pain |
| Types from spec | LLM-written interfaces | `openapi-typescript` (deterministic) + alias file | no hallucinated fields |
| Spec format | custom JSON | valid OpenAPI 3 + `x-mock` / `x-proposed` | schema-parser, redocly and MSW all need real OpenAPI |
| Mocks location | `src/__mocks__` | root `mocks/` and `test/` | outside FSD layers (Steiger), no clash with Vitest `__mocks__` |
| Dev API modes | `NODE_ENV` + `REACT_APP_`/`VITE_` mix | `VITE_API_MODE` = mock / hybrid / real | hybrid = MSW only for mock endpoints, real ones pass through |
| FSD layers | no widgets, `store/types/hooks` segments | + `widgets/`, `ui/model/api/lib/config` | standard FSD; Steiger understands it |
| FSD validation | qartez at plan time (empty graph) | Steiger + ESLint in `npm run lint`; qartez optional | deterministic, works without MCP |
| Styling | hex tokens + `dark:` everywhere + CSS modules | semantic CSS-variable tokens, Tailwind 3.4 pinned | dark mode automatic; one styling mechanism |
| Contracts | prose rules | ESLint rules (max-lines 250, no fetch/axios/style/hex/console/any) | machine-enforced |

If you prefer to keep Jest or hand-written thunks, tell me and I'll revert those parts only.

## Workflow changes (orchestrator)
- Phase 1 order: wireframe-analyzer → hybrid-api-config → schema-parser → mock-data-generator → fsd-planner.
- mock-data-generator ALWAYS runs (tests need handlers even with a complete backend).
- Store + styling are a Phase 3a foundation (the only parallel step), not per-task work.
- Explicit per-TASK loop: test-engineer (RED) → component-generator → coder (GREEN) → reviewer(task).
- app-bootstrap Run 2 moved to after all TASKs; reviewer runs again at release scope.
- Every subagent returns `next=orchestrator`; full routing table incl. BLOCKED-TEST, NEEDS-RESEARCH from any agent, loop guards.
- Gate 1 rejection routes to the agent that produced the rejected artifact (not architect).

## Per-file changes
| File | Main fixes |
|---|---|
| 00b-hybrid-api-config | OpenAPI output; runs after wireframe-analyzer; query-param gaps ≠ new endpoints; backend-gaps.md; no-spec mode; removed localStorage idea; valid HANDOFFs |
| 01-schema-parser | openapi-typescript; bash allowlist for lint/tsc; carries mock flag into endpoints-map.json |
| 02-wireframe-analyzer | can now write `analysis/**`; API-neutral data needs; new design-tokens.md |
| 03-fsd-planner | widgets layer; component Kind (presentational/connected/page); RTK Query integration table |
| 04-architect | TASK Files table with owner per file; sizing/order rules; no longer writes store-design.md |
| 05-store-architect | RTK Query baseApi (auth, 401 event, error normalization); fixed selectors; makeStore for tests |
| 5_5-mock-data-generator | typed handlers for every endpoint; seeded faker; db + resetDb; test scenarios; browser/node split; PII rule |
| 06-component-generator | presentational only; tokens; stories with `fn()` + fixtures; can't edit tests |
| 07-styling-engineer | CSS-variable semantic tokens; Tailwind 3.4 config fixed; contrast check; runs before components |
| 08-test-engineer | Vitest; true RED-first per TASK with valid-RED rules; fresh store per test; fixed example bugs |
| 09-coder | connected components + pages; bash denies for destructive git/npm install; Cortex removed |
| 10-reviewer | bash allowlist; command-backed gates A–F; diff-vs-Files scope check; fixed grep syntax; PII scan |
| 11-orchestrator | see workflow changes above |
| 12-app-bootstrap | pinned deps; all configs incl. ESLint/Steiger/Storybook; `msw init`; branch; error boundaries; env module |
| 13-researcher | source search order; can't overwrite api-schema.md |
| 14-reflector | dedup + ESCALATE on recurrence; proposes concrete prompt edits |
| 15-debugger | can't edit tests (BLOCKED-TEST instead); known-failure table for this stack |

## Please verify in your OpenCode version
1. **Permission rule order.** My understanding is that OpenCode applies the LAST matching pattern,
   so every file now lists `"*"` FIRST and specific rules after it. Your original files had
   `"*": deny` LAST, which under last-match-wins would deny everything. Check your version's
   permissions docs; if it is first-match-wins instead, move `"*"` to the end of each block.
2. **bash can bypass edit permissions** (e.g. `sed -i`). Coder/debugger/app-bootstrap have broad
   bash; the reviewer's diff-vs-Files-table check is the real enforcement.
3. **Model tiering.** All agents still use `opencode-go/deepseek-v4-flash`; comments mark
   architect, fsd-planner, reviewer, debugger and orchestrator as candidates for a stronger model,
   and reviewer ideally a different model than coder.
4. **qartez tool names** (`qartez_diff_impact`, `qartez_cochange`, `qartez_unused`) — confirm they
   exist in your qartez build; agents treat qartez as optional now.
5. **Version pins** (React 19, Vite 6, Vitest 3, Storybook 8, Tailwind 3.4) — adjust in AGENTS.md
   §1 and app-bootstrap if your org standardises on others.

---

# Revision 2 — modes, scanner, coverage, FSD doc

## New
| File | Purpose |
|---|---|
| 16-codebase-scanner.md | Read-only scan of an existing project → `PROJECT.md` (real conventions, command map, baseline failures) |
| 17-coverage-checker.md | Traces every HTML element, UI state, endpoint and data need → component → slice → TASK → file → test; writes `plan/coverage-matrix.md` |
| PLAYBOOK.md | How to install and run the team, with copy-paste prompts for each mode |

## Changed
| File | Change |
|---|---|
| AGENTS.md | §0 modes + `PROJECT.md` precedence (never migrate old code unasked) + delta rules; BUG-NNN ids; new owned files |
| 11-orchestrator | Intake step; four modes (new-project, gap-fill, feature, bugfix) with their own routes; coverage checks before every gate; hybrid-api-config now always runs so later YAML versions can be diffed |
| 00b-hybrid-api-config | Delta mode: merges new backend YAML into the baseline, flips mock → real, records contract drift in `api-changes.md` |
| 01-schema-parser | Delta: stable alias names, reports files that stop compiling |
| 02-wireframe-analyzer | `Source:` selector per component (for tracing); delta mode + `analysis/CHANGES.md` |
| 03-fsd-planner | Client FSD doc REQUIRED in new-project mode and authoritative; compliance table; delta/feature placement from PROJECT.md |
| 04-architect | Delta TASKs from coverage gaps (`Closes gaps:`); BUG-NNN template |
| 05-store-architect | Delta from api-changes.md; extends an existing project's own data mechanism instead of adding RTK Query |
| 5_5-mock-data-generator | Delta: new handlers, real endpoints leave `mockOnlyHandlers` |
| 08-test-engineer | Bugfix regression-test rule (must fail on current code for the reported reason) |
| 10-reviewer | Compares against PROJECT.md baseline — pre-existing failures aren't blamed on the agents |
| 12-app-bootstrap | Run R (retrofit): adds missing standard npm scripts as aliases, never new tools; ships `tools/list-html-elements.mjs` |
| 15-debugger | `mode=diagnose` for bug reports: root cause first, no fix before the regression test |

## Important: agent names
OpenCode names agents after their file names. Rename the files as listed in PLAYBOOK.md,
otherwise the orchestrator's dispatches (`coder`, `reviewer`, …) won't find them.
