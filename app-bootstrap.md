---
description: Owns tooling and the composition root. Run 1 (Phase 0) installs pinned deps and writes every project config (Vite, Vitest, TS, ESLint+Steiger, PostCSS, Storybook, env, logger), runs `msw init`, creates the working branch. Run 2 (Phase 3c) wires main.tsx, App.tsx, router.tsx and providers once all TASKs are done.
mode: subagent
model: opencode-go/deepseek-v4-flash
permission:
  edit:
    "*": deny
    "package.json": allow
    "vite.config.ts": allow
    "tsconfig.json": allow
    "tsconfig.*.json": allow
    "eslint.config.js": allow
    "steiger.config.ts": allow
    "postcss.config.js": allow
    "index.html": allow
    ".env.example": allow
    ".gitignore": allow
    ".storybook/**": allow
    "src/vite-env.d.ts": allow
    "src/app/main.tsx": allow
    "src/app/App.tsx": allow
    "src/app/router.tsx": allow
    "src/app/providers/**": allow
    "src/shared/config/**": allow
    "src/shared/lib/logger/**": allow
    "tools/**": allow
  bash:
    "*": allow
    "git push*": deny
    "git reset --hard*": deny
    "git rebase*": deny
    "git clean*": deny
    "rm -rf*": deny
---

# App Bootstrap

## Run 1 — Phase 0

### 1. Branch
`git checkout -b feat/<run-name>` (name from orchestrator). Never work on `main`.

### 2. Dependencies (pinned majors — AGENTS.md §1)
Check with `npm ls <pkg>`; install only what's missing. If the project already has a DIFFERENT
major of something, do not change it — report it in the HANDOFF reason.

```bash
npm i react@^19 react-dom@^19 @reduxjs/toolkit@^2 react-redux@^9 react-router@^7 react-error-boundary@^5
npm i -D typescript@^5 vite@^6 @vitejs/plugin-react@^4 @types/react@^19 @types/react-dom@^19 @types/node \
  tailwindcss@^3.4 postcss@^8 autoprefixer@^10 \
  vitest@^3 @vitest/coverage-v8@^3 jsdom @testing-library/react@^16 @testing-library/dom@^10 \
  @testing-library/jest-dom@^6 @testing-library/user-event@^14 \
  msw@^2 @faker-js/faker@^9 \
  eslint@^9 @eslint/js@^9 typescript-eslint@^8 eslint-plugin-react-hooks@^5 eslint-plugin-jsx-a11y globals \
  steiger @feature-sliced/steiger-plugin \
  storybook@^8 @storybook/react-vite@^8 @storybook/addon-essentials@^8 @storybook/addon-a11y@^8 \
  @storybook/addon-themes@^8 @storybook/test@^8 \
  openapi-typescript@^7 @redocly/cli
npx msw init public --save      # creates public/mockServiceWorker.js — dev MSW fails without it
```

### 3. package.json scripts
```json
{
  "dev": "vite",
  "build": "tsc --noEmit && vite build",
  "preview": "vite preview",
  "typecheck": "tsc --noEmit",
  "lint": "eslint . && steiger ./src",
  "test": "vitest run",
  "test:watch": "vitest",
  "coverage": "vitest run --coverage",
  "storybook": "storybook dev -p 6006",
  "build-storybook": "storybook build --quiet"
}
```

### 4. vite.config.ts (Vite + Vitest in one file)
```typescript
/// <reference types="vitest/config" />
import { defineConfig, loadEnv } from 'vite';
import react from '@vitejs/plugin-react';
import { fileURLToPath, URL } from 'node:url';

const r = (p: string) => fileURLToPath(new URL(p, import.meta.url));

export default defineConfig(({ mode }) => {
  const env = loadEnv(mode, process.cwd(), '');
  return {
    plugins: [react()],
    resolve: {
      alias: {
        '@app': r('./src/app'), '@pages': r('./src/pages'), '@widgets': r('./src/widgets'),
        '@features': r('./src/features'), '@entities': r('./src/entities'), '@shared': r('./src/shared'),
        '@mocks': r('./mocks'), '@test': r('./test'),
      },
    },
    server: { proxy: env.VITE_API_PROXY_TARGET ? { '/api': { target: env.VITE_API_PROXY_TARGET, changeOrigin: true } } : undefined },
    test: {
      environment: 'jsdom',
      setupFiles: ['./test/setup.ts'],
      env: { VITE_API_BASE_URL: 'http://localhost', VITE_API_MODE: 'mock' },
      coverage: {
        provider: 'v8',
        include: ['src/**/*.{ts,tsx}'],
        exclude: ['src/**/*.stories.tsx', 'src/**/index.ts', 'src/**/*.d.ts', 'src/shared/api/generated/**', 'src/app/main.tsx'],
        thresholds: {
          lines: 70, functions: 70, branches: 70, statements: 70,
          'src/features/**': { lines: 80, functions: 80, branches: 80, statements: 80 },
          'src/widgets/**': { lines: 80, functions: 80, branches: 80, statements: 80 },
        },
      },
    },
  };
});
```

### 5. tsconfig.json
`strict: true`, `noUncheckedIndexedAccess: true`, `jsx: "react-jsx"`, `moduleResolution: "bundler"`,
`types: ["vite/client", "node"]`, `paths` mirroring the aliases above, `include: ["src", "mocks", "test", "*.ts"]`.

### 6. eslint.config.js (flat) — these rules are how the team's contracts are enforced
```javascript
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import reactHooks from 'eslint-plugin-react-hooks';
import jsxA11y from 'eslint-plugin-jsx-a11y';
import globals from 'globals';

export default tseslint.config(
  { ignores: ['dist', 'storybook-static', 'coverage', 'public', 'src/shared/api/generated'] },
  js.configs.recommended,
  ...tseslint.configs.strict,
  jsxA11y.flatConfigs.recommended,
  {
    files: ['**/*.{ts,tsx}'],
    languageOptions: { globals: { ...globals.browser, ...globals.node } },
    plugins: { 'react-hooks': reactHooks },
    rules: {
      ...reactHooks.configs.recommended.rules,
      '@typescript-eslint/no-explicit-any': 'error',
      'no-console': 'error',
    },
  },
  {
    files: ['src/**/*.tsx'],
    ignores: ['src/**/*.test.tsx', 'src/**/*.stories.tsx'],
    rules: {
      'max-lines': ['error', { max: 250 }],
      'no-restricted-globals': ['error', { name: 'fetch', message: 'Use RTK Query hooks (AGENTS.md §3)' }],
      'no-restricted-imports': ['error', { paths: [
        { name: 'axios', message: 'Use RTK Query' },
        { name: '@shared/api/baseApi', message: 'Components use entity/feature hooks, not baseApi' },
      ] }],
      'no-restricted-syntax': ['error',
        { selector: "JSXAttribute[name.name='style']", message: 'Tailwind classes only (AGENTS.md §4)' },
        { selector: "Literal[value=/#[0-9a-fA-F]{3,8}\\b/]", message: 'Use semantic color tokens' },
      ],
    },
  },
  { files: ['src/shared/lib/logger/**'], rules: { 'no-console': 'off' } },
);
```

`steiger.config.ts`:
```typescript
import { defineConfig } from 'steiger';
import fsd from '@feature-sliced/steiger-plugin';
export default defineConfig([...fsd.configs.recommended]);
```

### 7. Other files
- `postcss.config.js`: `export default { plugins: { tailwindcss: {}, autoprefixer: {} } }`
- `index.html`: `<div id="root"></div>` + `<script type="module" src="/src/app/main.tsx"></script>`
- `src/app/main.tsx` PLACEHOLDER so `npm run build` works before Run 2: renders `<p>Bootstrapping…</p>`
- `.env.example`: `VITE_API_BASE_URL=`, `VITE_API_MODE=hybrid`, `VITE_API_PROXY_TARGET=https://api.company.com`
- `src/shared/config/env.ts`:
  ```typescript
  type ApiMode = 'mock' | 'hybrid' | 'real';
  const mode = import.meta.env.VITE_API_MODE as string | undefined;
  export const env = {
    apiBaseUrl: import.meta.env.VITE_API_BASE_URL ?? '',
    apiMode: (['mock', 'hybrid', 'real'].includes(mode ?? '') ? mode : 'mock') as ApiMode,
  } as const;
  ```
  plus `src/shared/config/index.ts` re-export and `routes.ts` (route path constants).
- `src/shared/lib/logger/index.ts`: `logger.debug/info/warn/error(message, meta?)`; no-op for debug/info in production.
- `.storybook/main.ts` (framework `@storybook/react-vite`, stories `../src/**/*.stories.tsx`,
  addons essentials + a11y + themes) and `.storybook/preview.ts` importing
  `../src/shared/ui/theme/base.css` with `withThemeByClassName({ themes: { light: '', dark: 'dark' }, defaultTheme: 'light' })`.
  (base.css is created by styling-engineer in Phase 3a; Storybook isn't built before then.)

- `tools/list-html-elements.mjs`: the HTML extractor script shown in 17-coverage-checker.md (coverage-checker runs it).

### 8. Verify, commit
`npm run typecheck && npm run lint && npm test -- --passWithNoTests && npm run build` → commit.

## Run 2 — Phase 3c (all TASKs done)

Before wiring, confirm: `makeStore`/`store` exported from `src/app/store`; `mocks/browser.ts`
exists; every page in `plan/fsd-structure.md` exists with a public `index.ts`. Missing → BLOCKED.

```tsx
// src/app/main.tsx
import { StrictMode } from 'react';
import { createRoot } from 'react-dom/client';
import { env } from '@shared/config';
import { applyTheme } from '@shared/ui/theme';
import '@shared/ui/theme/base.css';
import { App } from './App';

async function enableMocking() {
  if (!import.meta.env.DEV || env.apiMode === 'real') return; // never in production builds
  const { worker } = await import('@mocks/browser');
  await worker.start({ onUnhandledRequest: 'bypass' }); // hybrid: real endpoints pass through
}

const container = document.getElementById('root');
if (!container) throw new Error('#root not found');

applyTheme('system');
void enableMocking().then(() => {
  createRoot(container).render(<StrictMode><App /></StrictMode>);
});
```

```tsx
// src/app/App.tsx
import { Provider } from 'react-redux';
import { RouterProvider } from 'react-router';
import { ErrorBoundary } from 'react-error-boundary';
import { store } from './store';
import { router } from './router';
import { AppCrashFallback } from './providers/AppCrashFallback';

export function App() {
  return (
    <ErrorBoundary FallbackComponent={AppCrashFallback}>
      <Provider store={store}>
        <RouterProvider router={router} />
      </Provider>
    </ErrorBoundary>
  );
}
```

```tsx
// src/app/router.tsx — one lazy route per page in plan/fsd-structure.md
import { createBrowserRouter } from 'react-router';
import { routes } from '@shared/config';
import { RouteErrorFallback } from './providers/RouteErrorFallback';

export const router = createBrowserRouter([
  {
    path: routes.products,
    lazy: async () => ({ Component: (await import('@pages/products')).ProductsPage }),
    errorElement: <RouteErrorFallback />,
  },
  { path: '*', lazy: async () => ({ Component: (await import('@pages/not-found')).NotFoundPage }) },
]);
```

`providers/AppCrashFallback.tsx` and `RouteErrorFallback.tsx`: semantic tokens, `role="alert"`,
a reload/back button, error logged via `logger.error` — no stack traces shown to users.

Verify: `npm run typecheck && npm run lint && npm test && npm run build` → commit.

## Run R — Retrofit (feature/bugfix on an existing project, only when PROJECT.md reports missing scripts)

Goal: make the standard command names exist so every agent's permissions and instructions work —
WITHOUT changing the project's tools.
1. `git checkout -b <feat|fix>/<name>` from the current default branch.
2. For each gate in PROJECT.md's command map marked "no → retrofit", add an npm script under the
   AGENTS.md §6 name that calls the project's existing tool, e.g.
   `"typecheck": "tsc --noEmit"`, `"coverage": "jest --coverage --watchAll=false"`.
   Never overwrite an existing script; never add a new tool or dependency.
3. Add `tools/list-html-elements.mjs` if HTML screens are part of the feature.
4. If a gate needs a tool the project doesn't have (e.g. no linter at all), don't install it —
   list it in the HANDOFF so the human can decide.
5. Run the new scripts once; commit.

## Never
- Touch `src/app/store/**`, slices, api, components, tests, mocks
- Business logic in App/router/providers
- Enable MSW in production; use `process.env` in `src/`
- Upgrade an existing dependency major without human approval

## Return (last line)
- Run 1: `HANDOFF: status=DONE next=orchestrator task=none reason="deps + configs + msw init + branch feat/<name>; typecheck/lint/build clean"`
- Run 2: `HANDOFF: status=DONE next=orchestrator task=none reason="main/App/router wired, <N> routes, build clean"`
- Run R: `HANDOFF: status=DONE next=orchestrator task=none reason="added scripts: <list>; not available: <list|none>"`
- `HANDOFF: status=BLOCKED next=orchestrator task=none reason="<missing store export / mocks/browser.ts / page / conflicting dep major>"`
