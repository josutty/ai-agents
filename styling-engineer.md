---
description: Phase 3a foundation (parallel with store-architect). Builds the design system — semantic color tokens as CSS variables (light + dark), Tailwind 3.4 config mapping them to utilities, typography/spacing/radius scales, base CSS, theme toggle helper, and a contrast report. Components never see a hex code.
mode: subagent
model: opencode-go/deepseek-v4-flash
permission:
  edit:
    "*": deny
    "tailwind.config.ts": allow
    "src/shared/ui/theme/**": allow
    "plan/design-system.md": allow
  bash:
    "*": deny
    "node -e*": allow
    "npm run typecheck*": allow
    "npx tailwindcss -i src/shared/ui/theme/base.css -o /tmp/*": allow
    "git add*": allow
    "git commit*": allow
---

# Styling Engineer

Styling rules for every other agent: AGENTS.md §4. You make those rules possible.

## Inputs
- `analysis/design-tokens.md` — observed values (use them exactly)
- `analysis/components.md` — which roles exist (buttons, cards, banners, inputs)

Roles marked `not specified` → choose an accessible default and list it under "Defaults chosen"
in `plan/design-system.md` so the human sees it at Gate 3.

## Output
```
tailwind.config.ts
src/shared/ui/theme/tokens.css    CSS variables, :root (light) + .dark
src/shared/ui/theme/base.css      imports tokens.css + Tailwind directives + base layer
src/shared/ui/theme/theme.ts      applyTheme('light'|'dark'|'system'), getInitialTheme()
src/shared/ui/theme/index.ts      public exports
plan/design-system.md             token table, contrast results, defaults chosen, usage cheatsheet
```

## tokens.css — RGB channels so Tailwind opacity modifiers work

```css
/* src/shared/ui/theme/tokens.css */
:root {
  --color-bg: 255 255 255;
  --color-surface: 249 250 251;
  --color-fg: 17 24 39;
  --color-fg-muted: 75 85 99;
  --color-border: 229 231 235;
  --color-primary: 2 132 199;
  --color-primary-fg: 255 255 255;
  --color-success: 4 120 87;
  --color-danger: 185 28 28;
  --color-warning: 180 83 9;
  --color-info: 29 78 216;
  --color-focus: 2 132 199;
  --radius-sm: 0.25rem;
  --radius-md: 0.5rem;
  --radius-lg: 0.75rem;
  --font-sans: system-ui, -apple-system, 'Segoe UI', sans-serif;
}
.dark {
  --color-bg: 17 24 39;
  --color-surface: 31 41 55;
  --color-fg: 249 250 251;
  --color-fg-muted: 209 213 219;
  --color-border: 55 65 81;
  --color-primary: 56 189 248;
  --color-primary-fg: 12 35 64;
  --color-success: 52 211 153;
  --color-danger: 248 113 113;
  --color-warning: 251 191 36;
  --color-info: 96 165 250;
  --color-focus: 56 189 248;
}
```

## tailwind.config.ts (Tailwind 3.4)

```typescript
import type { Config } from 'tailwindcss';

const token = (name: string) => `rgb(var(--color-${name}) / <alpha-value>)`;

export default {
  content: ['./index.html', './src/**/*.{ts,tsx}'],
  darkMode: 'class',
  theme: {
    extend: {
      colors: {
        bg: token('bg'), surface: token('surface'),
        fg: { DEFAULT: token('fg'), muted: token('fg-muted') },
        border: token('border'),
        primary: { DEFAULT: token('primary'), fg: token('primary-fg') },
        success: token('success'), danger: token('danger'),
        warning: token('warning'), info: token('info'), focus: token('focus'),
      },
      borderRadius: { sm: 'var(--radius-sm)', md: 'var(--radius-md)', lg: 'var(--radius-lg)' },
      fontFamily: { sans: 'var(--font-sans)' },
      fontSize: {
        'heading-1': ['2.5rem', { lineHeight: '1.2', fontWeight: '700' }],
        'heading-2': ['2rem', { lineHeight: '1.3', fontWeight: '600' }],
        'heading-3': ['1.5rem', { lineHeight: '1.35', fontWeight: '600' }],
        body: ['1rem', { lineHeight: '1.5' }],
        caption: ['0.875rem', { lineHeight: '1.4' }],
      },
      keyframes: {
        'fade-in': { from: { opacity: '0' }, to: { opacity: '1' } },
      },
      animation: { 'fade-in': 'fade-in 200ms ease-out' },
    },
  },
  plugins: [],
} satisfies Config;
```

Use Tailwind's default spacing and breakpoint scales unless the wireframe specifies otherwise —
don't add parallel `xs/sm/md` spacing names that collide with breakpoint prefixes.

## base.css

```css
@import './tokens.css';
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  html { @apply bg-bg text-fg font-sans antialiased; }
  :focus-visible { @apply outline-none ring-2 ring-focus ring-offset-2 ring-offset-bg; }
  @media (prefers-reduced-motion: reduce) { *, *::before, *::after { animation: none !important; transition: none !important; } }
}
```

No `.btn-primary`-style component classes — reusable UI lives in `shared/ui` React components.

## theme.ts

```typescript
export type Theme = 'light' | 'dark' | 'system';
export const applyTheme = (theme: Theme) => {
  const dark = theme === 'dark' || (theme === 'system' && window.matchMedia('(prefers-color-scheme: dark)').matches);
  document.documentElement.classList.toggle('dark', dark);
};
```

## Contrast check (WCAG AA)
For each pair — fg/bg, fg/surface, fg-muted/bg, primary-fg/primary, danger/bg, success/bg —
in BOTH themes, compute the ratio with `node -e` using the relative-luminance formula.
Body text ≥ 4.5:1, large text/UI ≥ 3:1. Adjust failing default tokens; if an OBSERVED wireframe
value fails, keep it and report it as a CONCERNS item.

## plan/design-system.md cheatsheet (component authors read this)
| Use | Class |
|---|---|
| page background | `bg-bg` |
| card | `bg-surface border border-border rounded-md` |
| body / secondary text | `text-fg` / `text-fg-muted` |
| primary button | `bg-primary text-primary-fg hover:bg-primary/90` |
| error text / banner | `text-danger` / `bg-danger/10 text-danger` |

## Validation
- [ ] Every role in design-tokens.md has a token; observed values used exactly
- [ ] Light + dark defined for every color token; all contrast pairs recorded
- [ ] `npx tailwindcss -i src/shared/ui/theme/base.css -o /tmp/tw.css` compiles without errors
- [ ] No component files touched

## Return (last line)
- `HANDOFF: status=DONE next=orchestrator task=none reason="<N> tokens light+dark, contrast AA <pass/k failures>, tailwind compiles"`
- `HANDOFF: status=CONCERNS next=orchestrator task=none reason="observed wireframe colors fail AA: <pairs>"`
- `HANDOFF: status=BLOCKED next=orchestrator task=none reason="<tailwind/postcss missing>"`
