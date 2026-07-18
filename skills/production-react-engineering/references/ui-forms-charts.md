# Reusable UI, Styling, Responsive, Accessibility, Forms, Validation, Charts

**Library verdicts up front (verified against every `package.json` + imports):**

| Concern | Standard | Verdict |
|---------|----------|---------|
| UI kit | hand-rolled primitives in `core/components` + `shared-ui/components` | **no third-party UI kit** |
| Material UI | — | **NOT used** |
| Tailwind | — | **NOT used** |
| styled-components / Emotion | — | **NOT used** |
| CSS | CSS Modules `.module.css` + `classnames` + PostCSS `$tokens`/CSS vars | ✅ |
| Forms | Formik (controlled) | ✅ |
| react-hook-form | — | **NOT used** |
| Validation | Yup + custom validators, localized messages | ✅ |
| Zod | server/tooling only, **never** UI | not in UI |
| Charts | a lightweight SVG charting library + a trends plugin | ✅ |
| recharts / chart.js / d3 / victory / nivo / highcharts | — | **none used** |
| Responsive | build-time desktop/mobile split + `deviceWidths.css` | ✅ (no runtime `useMediaQuery`) |

React 17 + TypeScript. Don't reach for a UI kit, Tailwind, CSS-in-JS, react-hook-form, or a
new charting lib — the house patterns below are the standard.

## Reusable UI component library

- **Hand-rolled primitives**, no external kit. Core (in `core/components/`):
  `Button`, `Text` (namespaced sizes `Text.Base/Small/Large/...`), `Box`, `Flex`, `Group`,
  `Input`, `PasswordInput`, `OTPInput`, `InputContainer`, `Select`, `Checkbox`, `Toggle`,
  `RadioGroup`, `Dialog`/`FloatingDialog`/`DialogWithHeader`, `Section`, `Container`, `Pill`,
  `Counter`, `MessageBox`, `ListEntry`, `LoadingSpinner`/`LoadingDots`/`LoadingAnimation`,
  `Header`, `BackButton`. Composites in `shared-ui/components/`: `MultiSelect`, `Chip`,
  `FiltersButton`, `Stepper`, `StatsBar`, `Toast`, `LocationSelector`, `Dropdown`.
- **Public surface is the barrel `index.ts`** re-exporting defaults as named exports (some as
  namespaces: `import * as Text`).
- **Conventions:** inline `type Props` with every field `readonly`; **boolean variant flags**
  rather than one `variant` enum (`Button` takes `secondary`, `tertiary`, `cta`, `danger`,
  `small`, `stretch`, …), flattened to classes with `classnames`; extend native props
  (`React.ComponentProps<'span'>`, `React.InputHTMLAttributes<HTMLInputElement>` + `Omit`);
  `React.forwardRef<HTMLDivElement, Props>(...)` + `.displayName` for layout primitives;
  **compose, don't duplicate** (`Button` renders its label via `Text.Base`; `Input` delegates
  chrome to `InputContainer`; `Dialog` wraps the lower-level modal).
- **Why:** one owned, composable primitive set keeps bundle size down, gives full control of
  RTL/theming/i18n, and avoids a heavyweight kit the multi-portal theming would fight.

```tsx
// group.tsx — forwardRef + CSS-variable pattern
const Group = React.forwardRef<HTMLDivElement, Props>(
    ({ children, className, template, gap, ...rest }, ref) => (
        <div {...rest} ref={ref}
             style={{ '--gap': gap, '--template': template, ...rest.style }}
             className={classNames(styles.group, className)}>{children}</div>
    ),
);
Group.displayName = 'Group';
```

## Styling

CSS Modules (`.module.css`) + `classnames` + PostCSS with SCSS-like `$` theme tokens and CSS
custom properties. Theme tokens live in a branding variables file (`$primaryColor`, spacing,
`$borderRadius`, `$inputHeight`); components layer runtime theming through CSS custom
properties **with token fallbacks**:
`background-color: var(--btn-bg-color, $primaryButtonBackgroundColor);`. RTL (e.g. Arabic) is
handled **at build time** via a build-time RTL PostCSS plugin — write LTR CSS, don't
hand-author RTL. See `structure-components-styling.md` for the full CSS-module conventions.

## Responsive design

- **Build-time platform split, not runtime.** Separate entrypoints (`main.desktop.ts` /
  `main.mobile.ts`) build separate bundles with separate page trees (`core/pages/desktop.ts`
  vs `pages/mobile.ts`), and features ship paired files (`homePageDesktop.tsx` /
  `homePageMobile.tsx`, `records.desktop.ts` / `records.mobile.ts`). Mobile entry starts with
  `import 'core/app/preload.mobile'` (must be first).
- **Shared breakpoint tokens** in `core/styles/deviceWidths.css`
  (`$deviceWidthMobile: 768px`, `$deviceWidthTablet: 950px`, `$deviceWidthDesktop: 1366px`);
  CSS modules use `@media (min-width: $deviceWidthTablet)`.
- **There is no runtime `useMediaQuery`/`matchMedia` hook** — device is decided at build/serve
  time, so components don't branch on viewport width in JS. **Why:** shipping only the
  relevant bundle keeps mobile payloads small and avoids hydration mismatches.

## Accessibility

- **Standard:** semantic HTML (`<main>`, `<header>`, `<section>`, `<nav>`, `<article>`),
  `aria-label` on interactive elements (primitives thread an `ariaLabel` prop to the DOM —
  `Button`, `Toggle`, `ListEntry`), `aria-selected`/`aria-haspopup`/`aria-live="polite"` for
  menus and carousels. **`eslint-plugin-jsx-a11y` is enabled** to enforce baseline rules.
- **Localize a11y strings** (``aria-label={t(i18n)`Close`}``) — this is the standard, though
  applied unevenly (feature code localizes; some shared primitives still have hardcoded
  English labels — fix these when you touch them).
- **Known gaps to improve, not copy:** keyboard handling is sparse at the primitive layer
  (e.g. `Input` uses `keyCode === 13` for Enter) and there's no focus-trap utility. When
  adding interactive components, add proper keyboard support and focus management.

## Forms — Formik

- **Formik is the sole form library** (controlled components). **No react-hook-form.**
- Field infrastructure in `core/form/fields.tsx` wraps Formik's `<Field>` with the
  design-system inputs and centralizes error surfacing (`touched && errors` gating), disabling
  inputs during submit (`disabled={form.isSubmitting}`). Reusable field components live in
  `core/forms/` (`textInput`, `priceInput`, `checkBox`, `submitButton`, `errorMessage`,
  plus mobile variants).

```tsx
const getFieldError = (form, fieldName) =>
    form.touched[fieldName] && form.errors[fieldName] ? form.errors[fieldName] : null;
```

## Validation — Yup

- **Yup schemas wired into Formik**, plus hand-written validators (password strength regexes)
  where a schema doesn't fit. **Zod is not used for UI validation** (only server/tooling).
  **No** validation logic scattered in components — schemas live in `core/form/validation.ts` /
  `contact/validationSchema.ts`.
- Domain rules are registered as custom Yup methods (`Yup.addMethod(Yup.string, 'phone', ...)`
  delegating to `isValidPhoneNumber`); **all messages are localized inline**
  (``.required(t(i18n)`Please enter your name`)``). Errors display via the `erroring`/
  `errorMessage` props on `Input`/`InputContainer` and dedicated `errorMessage`/`inputError`
  components.
- ⚠️ **Cross-repo divergence:** one frontend app pins **Yup 0.23** while the other pins **Yup
  1.7** — the schema API differs, so validation code is not portable verbatim between the two.

## Charts — a lightweight SVG charting library

- **A lightweight SVG charting library** plus a small trends plugin. **No recharts / chart.js /
  d3 / victory / nivo / highcharts.**
- Where: `core/trends/chart/` (`trendsChart.tsx`, `priceComparisonTrendsChart.tsx`), a report
  feature's `reportGraph.tsx` (custom Y-axis ticks + chart CSS), an analytics feature's metrics
  insights (custom chart plugins: `stepFunctionTooltip.ts`, `pointValues.ts`), and a typed
  wrapper `shared-ui/charts/Chart.tsx` (exposes `ChartType`, axis interpolation, sizing).
  KPI/stat tiles (`StatsBar`, `variancePill`) are plain components, not chart-lib based.
- **Why this library:** lightweight, SVG-based, themeable with CSS — fits the small-bundle,
  CSS-variable styling philosophy better than a heavy React charting lib. Reuse the existing
  wrapper/plugins rather than adding a new charting dependency.
