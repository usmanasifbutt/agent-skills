# Project Structure, Components, Hooks, Styling & Naming

## Project structure (the multi-portal frontend)

One React codebase serves many branded portals. Layers, from most-shared to most-specific —
put code as high (shared) as it can correctly live:

```
frontend/
├── core/           # Core platform: components, API clients, routes, state, i18n, server (BFF)
├── shared-ui/      # Generic UI components + utilities + hooks shared across ALL products
├── brand-a/        # Brand-A-specific extensions
├── brand-b/        # Brand-B-shared code
├── <brand>-<country>/   # Country apps: locale, static assets, overrides
├── packages/       # Internal packages
└── e2e/            # WebdriverIO E2E tests
```

Each app: `{app}/` source (with `index.ts`), `locale/`, `static/`, `package.json`, the
settings config (runtime settings), the i18n config, `tsconfig.json`.

**Why:** an expand-once philosophy — shared behavior lives in the shared core / shared UI
library, portal differences are thin overrides. Before adding code, ask which layer it belongs
in; a component used by more than one product goes in the shared UI library, platform plumbing
in the shared core.

**Imports** are by package path, not deep relative paths across layers:
`import { SearchPage } from 'core/search'`, `import { Button } from 'shared-ui/components'`,
`import styles from './styles/component.module.css'`.

## Feature-folder organization

A feature is a self-contained folder with a public surface:

```
feature/
├── index.ts              # public exports (import from the folder, not deep files)
├── featureComponent.tsx  # main component (camelCase file, PascalCase default export)
├── types.ts              # TypeScript types
├── state/                # redux slice + selectors (state/index.ts, selectors.ts, hooks.ts)
├── __tests__/            # colocated tests
├── __mocks__/            # colocated module mocks
├── getDefaultSortByValue.ts  # small single-purpose logic files (see note)
└── styles/
    └── featureComponent.module.css
```

**Why:** `index.ts` is the contract — consumers import `from 'core/feature'` (never a deep
path), so internals can be refactored freely. Components (`.tsx`), hooks (`useX.ts`), pure
logic (`.ts`), and route handlers (`routes/`) sit flat in the feature dir; large features add
sub-folders (`search/pagination/`, `search/mapBasedSearch/`).

**Note on `helpers.ts`:** a common instinct is to add a `helpers.ts` per feature, but the
preferred practice here is **zero** — logic is split into **many small single-purpose files
named after their function** (`computeHighestLevel.ts`, `getDefaultSortByValue.ts`,
`resetFilters.ts`). Follow that: one focused file per function, not a catch-all `helpers.ts`.

## Components

```tsx
import * as React from 'react';              // ALWAYS namespace import, never `import React`
import classNames from 'classnames';
import styles from './styles/myComponent.module.css';

type Props = {
    readonly children: React.ReactNode;      // props are `readonly`; use `type`, never `interface`
    readonly variant?: 'primary' | 'secondary';
    readonly disabled?: boolean;
    readonly onClick?: (event: React.MouseEvent<HTMLButtonElement>) => void;
};

const MyComponent = ({ children, variant = 'primary', disabled, onClick }: Props) => (
    <button
        className={classNames(styles.button, {
            [styles.primary]: variant === 'primary',
            [styles.disabled]: disabled,
        })}
        onClick={onClick}
        disabled={disabled}
    >
        {children}
    </button>
);

export default MyComponent;
```

Conventions and **why**:
- **`import * as React from 'react'`** — the enforced import style across the codebase
  (consistency; works regardless of esModuleInterop settings).
- **Props typed in a `Props` type, `readonly`, `type` over `interface`** — readonly documents
  that components don't mutate props; `type` composes (unions, `&`) more predictably than
  `interface` and the team standardized on it.
- **Extend HTML props** with `React.ComponentProps<'button'> & {...}` rather than re-declaring
  native attributes.
- **`React.forwardRef<HTMLInputElement, Props>`** for components that must expose a DOM ref
  (inputs, focus management).
- **Default exports** for the main component; named exports via the feature's `index.ts`.

## Hooks

- **Namespace-prefix every built-in hook:** `React.useState`, `React.useEffect`,
  `React.useRef`, `React.useMemo`, `React.useCallback` — consistent with the namespace import.
- **Custom hooks are named `useX`** and live with the feature (or in the shared UI library if shared).
- Correct dependency arrays; memoize (`useMemo`/`useCallback`) computed values and callbacks
  passed to memoized children (see the performance reference for when it's worth it).

## Styling — CSS Modules only

- **CSS Modules, `.module.css` extension**, in a `styles/` subdirectory named after the
  component file (`myComponent.tsx → ./styles/myComponent.module.css`), imported as `styles`:
  `import styles from './styles/myComponent.module.css'`.
- **`classnames` for all conditional classes** — `classNames(styles.base, { [styles.active]:
  isActive })`. Never build className strings by hand.
- **Theme tokens are CSS variables** — `var(--color-text)`, `var(--border-radius)`; don't
  hard-code colors/spacing.
- **Before removing/renaming a CSS class or module file, grep the codebase** — modules are
  sometimes imported elsewhere to reuse classes.
- **No CSS-in-JS.** (See the UI reference for the definitive "not used" list — no
  styled-components/emotion.)

## Naming conventions

- **Files:** camelCase — `myComponent.tsx`, `radioGroup.tsx`, `helpers.ts`,
  `styles/myComponent.module.css`. (The file is camelCase even though the default-exported
  component is PascalCase.)
- **Components:** PascalCase. **Hooks:** `useCamelCase`. **Props type:** always `Props`.
- **Enums:** PascalCase name, PascalCase members (`RouteNames.RECORD_DETAILS` uses
  UPPER_SNAKE members in the router). **CSS classes:** camelCase (`styles.searchButton`).
- **API paths & JSON keys:** camelCase (`/api/myRecords`, not kebab) — matches the backend.
- **Avoid abbreviations**; write the full word.

## TypeScript discipline

- **`npm run lint:types` must be clean** — master has zero errors, so any error is yours.
- **No `// @ts-expect-error` / `@ts-ignore` / `@ts-nocheck`** — fix the underlying type.
- **No `as` type assertions** — use type guards and narrowing. (If a proper fix needs a large
  refactor, raise it as a separate PR.)
- ES modules only; TypeScript always; code that runs in both browser and Node (SSR).
