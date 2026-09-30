# Project Conventions (single source of truth for ALL agents)

OpenCode loads this file into every agent's context. When an agent file and this file
disagree, THIS file wins — report the conflict in your HANDOFF reason.

## 0. Modes and precedence

The orchestrator runs in one of four modes: `new-project`, `gap-fill`, `feature`, `bugfix`.
You receive `mode=` (and often `delta=true`) in your dispatch — follow your file's section for it.

**Existing projects:** if `PROJECT.md` exists at the repo root (written by codebase-scanner),
read it FIRST. For that project, its "Rules for NEW code", "Command map" and path map override
the defaults in this file (stack, data mechanism, styling, test runner, file paths, command names).
Sections §7 Git, §8 Sensitive data, and §10 HANDOFF always apply and are never overridden.
Never migrate existing code to this file's conventions unless the TASK explicitly says so.

**Delta runs** (`delta=true`): change only what the listed changed inputs require; keep ids,
numbering, file names and existing content stable; record what you changed in the file your
section names (e.g. `analysis/CHANGES.md`, `plan/generated/api-changes.md`).

## 1. Stack (pinned majors — do not upgrade without human approval)

React 19 · TypeScript 5 (strict) · Vite 6 · Redux Toolkit 2 (+ RTK Query) · react-redux 9 ·
react-router 7 · Tailwind CSS 3.4 · Vitest 3 + React Testing Library 16 + user-event 14 +
jest-dom 6 · MSW 2 · Storybook 8 · @faker-js/faker 9 · ESLint 9 (flat config) + Steiger (FSD linter)

Env vars (Vite): read ONLY through `src/shared/config/env.ts`, never `process.env`.
- `VITE_API_BASE_URL` — prefix for every API path ('' in dev → Vite proxy; `http://localhost` in tests)
- `VITE_API_MODE` — `mock` (all MSW) | `hybrid` (MSW for `x-mock` endpoints only) | `real` (no MSW)

## 2. Feature-Sliced Design

```
src/
├── app/        providers, store root, router, global styles import — composition only
├── pages/      one folder per route — composition only, zero business logic
├── widgets/    large self-contained UI blocks composed of features + entities
├── features/   user actions (filter-products, add-to-cart, login)
├── entities/   business nouns (product, cart, session, user)
└── shared/     api, ui kit, lib, config — no business knowledge
mocks/          MSW handlers, fixtures, factories (outside src/, not an FSD layer)
test/           Vitest setup + render helpers (outside src/, not an FSD layer)
```

Import matrix (a layer may import only from layers BELOW it):

| Layer | May import |
|---|---|
| app | pages, widgets, features, entities, shared |
| pages | widgets, features, entities, shared |
| widgets | features, entities, shared |
| features | entities, shared |
| entities | shared (cross-entity only via `@x` public API, and only if planned) |
| shared | npm packages only — never any layer above |

- No imports between two slices of the SAME layer (features/a → features/b is forbidden).
- Import another slice only through its `index.ts` public API, never deep paths.
- Segments inside a slice: `ui/`, `model/`, `api/`, `lib/`, `config/`. No `store/`, `types/`, `hooks/` segment names.
- Aliases: `@app @pages @widgets @features @entities @shared @mocks @test`.
- `RootState`, `AppStore`, `AppDispatch` are GLOBAL types declared in `src/app/store/types.d.ts`.
  Nothing below `app/` ever imports from `@app`.

## 3. Data contract (one mechanism, no exceptions)

- **Server data → RTK Query only.** `baseApi` lives in `src/shared/api/baseApi.ts`. Endpoints are
  added with `baseApi.injectEndpoints` in `entities/<e>/api/` (or `features/<f>/api/` for
  feature-only mutations). Components use the generated hooks (`useListProductsQuery`, …).
- **Client state → `createSlice`** in `entities/<e>/model/` or `features/<f>/model/`, combined in
  `src/app/store` with `combineSlices`. Read with `slice.selectors.*` via `useAppSelector`.
- `createAsyncThunk` only for multi-step workflows RTK Query can't express, and only when the TASK names it.
- Forbidden in components: `fetch`, `axios`, `useState`/`useEffect` holding server data, importing `baseApi`.
- All API errors are normalized to `ApiError` (`src/shared/api/errors.ts`). Show them with `getErrorMessage(error)`.
- Every server-data consumer handles 4 states: loading · error (with retry) · empty · success.

## 4. Styling

- Tailwind utilities only. Colors via SEMANTIC tokens (`bg-surface`, `text-fg`, `bg-primary`,
  `text-danger`, `border-border`), which map to CSS variables in `src/shared/ui/theme/tokens.css`.
- Forbidden in `.tsx`: `style=`, hex/rgb literals, CSS modules, raw palette classes (`bg-blue-600`),
  dynamically built class names (`bg-${color}`) — use a map of full class strings instead.
- Dark mode = `class` on `<html>`; semantic tokens switch automatically, so `dark:` is rarely needed.
- Mobile-first breakpoints (`sm: md: lg: xl:`).

## 5. Testing

- Vitest + RTL, role-first queries (`getByRole`, `getByLabelText`), `user-event` for interaction.
- Render connected components with `renderWithProviders` from `@test/test-utils` (fresh store per test).
- API is mocked ONLY by MSW (`@mocks/node`); per-test overrides via `server.use(...scenarios.x)`.
  Never `vi.mock` the API layer. `onUnhandledRequest: 'error'`.
- No snapshot tests. Deterministic data only (seeded faker fixtures). No real timers without fake timers.
- Coverage (enforced in vite.config.ts): global 70%; `src/features/**` and `src/widgets/**` 80%.
- **Valid RED** = a test fails on an assertion, OR fails because a file listed as `new` in the
  TASK's Files table doesn't exist yet. Any other failure (setup crash, missing mock, syntax) is an invalid RED.

## 6. Quality gates (commands every agent uses — never invent others)

| Command | Checks |
|---|---|
| `npm run typecheck` | `tsc --noEmit` strict |
| `npm run lint` | ESLint (no-any, no-console, no `style=`, no fetch/axios in tsx, max 250 lines per .tsx, a11y, hooks) + Steiger (FSD) |
| `npm test` | Vitest run |
| `npm run coverage` | Vitest + thresholds |
| `npm run build` | Vite production build |
| `npm run build-storybook` | Storybook builds (all stories compile) |

Existing projects: use the command map in PROJECT.md. app-bootstrap's retrofit run adds missing
scripts under these names (as aliases to the project's own tools) so permissions keep working.

## 7. Git

- One working branch per run, created by app-bootstrap: `feat/<run-name>`. Never commit to `main`.
- Commit your own files at the end of your step: `git commit -m "TASK-NNN(<agent>): <summary>"`.
- Never: `git push`, `--force`, `reset --hard`, `rebase`, `checkout -- .`, `clean -fd`. Humans push.

## 8. Sensitive data (organization policy — applies to code, mocks, fixtures, logs, docs)

Never generate, store, log, or display: SSN, Aadhaar, PAN, passport numbers, driver's licence
numbers, credit card numbers, or patient records — not even fake-looking ones. If an API field
requires such a value, mocks use the literal `"REDACTED"` and UI shows a masked placeholder.
Mock emails use `@example.com`; mock phones use obviously fictional numbers.

## 9. Logging

No `console.*` anywhere in `src/` except `src/shared/lib/logger/`. Stories use `fn()` from `@storybook/test`.

## 10. HANDOFF (every subagent's LAST line — exactly one line)

```
HANDOFF: status=<STATUS> next=orchestrator task=<TASK-NNN|none> reason="<one sentence>"
```

`next` is ALWAYS `orchestrator`; routing lives only in the orchestrator's table.
Ids: `TASK-NNN` for planned work, `BUG-NNN` for bug fixes (same loop, same rules).
Allowed statuses: `DONE READY PASS CONCERNS FAIL-MECH FAIL-DESIGN BLOCKED BLOCKED-DESIGN
BLOCKED-BUG BLOCKED-TEST NEEDS-RESEARCH DUPLICATE FIXED VERIFY-FAIL-DESIGN LOGGED`

## 11. Key files (who owns what — edit only files you own)

| Path | Owner |
|---|---|
| `analysis/**` | wireframe-analyzer |
| `plan/generated/api-spec.yaml`, `plan/generated/backend-gaps.md` | hybrid-api-config |
| `plan/generated/endpoints-map.json`, `src/shared/api/generated/**`, `notes/research/api-schema.md` | schema-parser |
| `mocks/**` | mock-data-generator |
| `plan/fsd-structure.md`, `plan/component-owners.md`, `plan/api-integration-points.md` | fsd-planner |
| `plan/tasks/**`, `plan/PROGRESS.md` | architect (the `status` column only: test-engineer → `in-progress`, reviewer → `done`) |
| `src/shared/api/*.ts` (not generated/), `src/app/store/**`, `src/shared/lib/store/**`, `src/*/*/api/**`, `src/*/*/model/**`, `plan/store-design.md` | store-architect |
| `tailwind.config.ts`, `src/shared/ui/theme/**`, `plan/design-system.md` | styling-engineer |
| `test/**`, `src/**/*.test.ts(x)` | test-engineer |
| presentational `ui/` files + `*.stories.tsx` (per TASK Files table) | component-generator |
| connected `ui/` files, `src/pages/**` (per TASK Files table) | coder |
| root configs, `.storybook/**`, `src/app/{main,App,router}.tsx`, `src/app/providers/**`, `src/shared/config/**`, `src/shared/lib/logger/**` | app-bootstrap |
| `PROJECT.md`, `notes/research/codebase-map.md` | codebase-scanner |
| `plan/coverage-matrix.md` | coverage-checker |
| `analysis/CHANGES.md` | wireframe-analyzer (delta runs) |
| `plan/generated/api-changes.md` | hybrid-api-config (delta runs) |
| `tools/**` (repo helper scripts) | app-bootstrap |
| `notes/research/**` (except api-schema.md, codebase-map.md) | researcher |
| `notes/memory/reflections/**` | reflector |
| `plan/history/TASK-NNN.md` | append-only, any agent working on that task |
