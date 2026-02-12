# The Power of 10 Rules for Safety-Critical TailwindCSS Code

## Background

The **Power of 10 Rules** were created in 2006 by **Gerard J. Holzmann** at NASA's Jet Propulsion Laboratory (JPL) Laboratory for Reliable Software. These rules were designed for writing safety-critical code in C that could be effectively analyzed by static analysis tools.

The rules were incorporated into JPL's institutional coding standard and used for major missions including the **Mars Science Laboratory** (Curiosity Rover, 2012).

> *"If these rules seem draconian at first, bear in mind that they are meant to make it possible to check safety-critical code where human lives can very literally depend on its correctness."* — Gerard Holzmann

**Note:** This adaptation focuses on utility-first CSS patterns with TailwindCSS, covering design system constraints, responsive design, dark mode, component extraction, type-safe variants, and avoiding common pitfalls in production applications.

---

## The Original 10 Rules (C Language)

| # | Rule |
|---|------|
| 1 | Restrict all code to very simple control flow constructs—no `goto`, `setjmp`, `longjmp`, or recursion |
| 2 | Give all loops a fixed upper bound provable by static analysis |
| 3 | Do not use dynamic memory allocation after initialization |
| 4 | No function longer than one printed page (~60 lines) |
| 5 | Assertion density: minimum 2 assertions per function |
| 6 | Declare all data objects at the smallest possible scope |
| 7 | Check all return values and validate all function parameters |
| 8 | Limit preprocessor to header includes and simple macros |
| 9 | Restrict pointers: single dereference only, no function pointers |
| 10 | Compile with all warnings enabled; use static analyzers daily |

---

## The Power of 10 Rules — TailwindCSS Edition

### Rule 1: Simple Control Flow — Consistent Class Ordering, No Conflicting Utilities

**Original Intent:** Eliminate complex control flow that impedes static analysis and reasoning.

**TailwindCSS Adaptation:**

```html
<!-- BAD: Conflicting utilities (unpredictable output) -->
<div class="w-full w-1/2 p-4 p-8">
  <!-- Which width applies? Which padding? -->
</div>

<!-- BAD: Random class order (hard to scan/review) -->
<div class="text-blue-500 p-4 hover:bg-gray-100 flex mt-2 w-full items-center">
  Inconsistent ordering
</div>

<!-- BAD: Overriding utility classes -->
<div class="bg-red-500 bg-blue-500">
  <!-- bg-blue-500 wins, but this is confusing -->
</div>

<!-- GOOD: Consistent ordering (layout → spacing → sizing → typography → visual) -->
<div class="flex items-center w-full mt-2 p-4 text-blue-500 hover:bg-gray-100">
  Predictable, scannable ordering
</div>

<!-- GOOD: Single responsibility per utility category -->
<div class="
  flex flex-col items-start justify-between
  w-full max-w-4xl
  p-6 space-y-4
  text-base leading-relaxed
  bg-white border border-gray-200 rounded-lg shadow-sm
">
  Clear, non-conflicting utilities
</div>

<!-- GOOD: Explicit order using prettier-plugin-tailwindcss -->
<button class="
  inline-flex items-center justify-center
  rounded-md
  px-4 py-2
  text-sm font-medium
  text-white
  bg-blue-600
  hover:bg-blue-700
  focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2
  disabled:opacity-50 disabled:cursor-not-allowed
">
  Submit
</button>
```

**Recommended Class Order:**
1. Layout: `flex`, `grid`, `block`, `inline-flex`, `table`
2. Positioning: `absolute`, `relative`, `fixed`, `sticky`, `top-*`, `left-*`
3. Display/Visibility: `hidden`, `invisible`, `visible`
4. Flexbox/Grid: `items-*`, `justify-*`, `gap-*`, `grid-cols-*`
5. Spacing: `p-*`, `m-*`, `space-*`
6. Sizing: `w-*`, `h-*`, `max-w-*`, `min-h-*`
7. Typography: `text-*`, `font-*`, `leading-*`, `tracking-*`
8. Visual: `bg-*`, `border-*`, `rounded-*`, `shadow-*`
9. Effects: `opacity-*`, `transition-*`, `transform-*`
10. Interactivity: `hover:*`, `focus:*`, `active:*`, `disabled:*`

**Guidelines:**
- No conflicting utilities in the same class list
- Use `prettier-plugin-tailwindcss` for automatic consistent ordering
- One utility per style property (no `w-full w-1/2` duplication)
- Group related utilities visually with line breaks for complex components
- Use variant prefixes systematically: `hover:`, `focus:`, `dark:`, `md:`, etc.

---

### Rule 2: Bounded Design Tokens — Finite Scales, No Arbitrary Values

**Original Intent:** Ensure all loops terminate with provable upper bounds.

**TailwindCSS Adaptation:**

```html
<!-- BAD: Arbitrary values everywhere (infinite design system) -->
<div class="w-[347px] h-[219px] p-[23px] text-[17.5px] bg-[#3b82f6]">
  <!-- No consistency, impossible to audit -->
</div>

<!-- BAD: Custom values bypassing design tokens -->
<div class="mt-[calc(100vh-87px)] text-[clamp(1rem,2.5vw,3rem)]">
  <!-- Complex, unverifiable values -->
</div>

<!-- BAD: Unbounded color palette -->
<div class="bg-[#ff6b9d] border-[#a8e6cf] text-[#ffd700]">
  <!-- Random hex colors, no system -->
</div>

<!-- GOOD: Using design system tokens only -->
<div class="w-96 h-64 p-6 text-lg bg-blue-500">
  <!-- All values from predefined scale -->
</div>

<!-- GOOD: Constrained spacing scale -->
<div class="mt-4 mb-8 px-6 py-12">
  <!-- 1rem, 2rem, 1.5rem, 3rem — predictable spacing -->
</div>

<!-- GOOD: Semantic color tokens -->
<div class="bg-primary-500 text-gray-900 border-gray-200 hover:bg-primary-600">
  <!-- Uses theme colors -->
</div>
```

**tailwind.config.js — Constrained Design System:**
```javascript
/** @type {import('tailwindcss').Config} */
export default {
  theme: {
    // Constrained color palette
    colors: {
      transparent: 'transparent',
      current: 'currentColor',
      white: '#ffffff',
      black: '#000000',

      gray: {
        50: '#f9fafb',
        100: '#f3f4f6',
        200: '#e5e7eb',
        300: '#d1d5db',
        400: '#9ca3af',
        500: '#6b7280',
        600: '#4b5563',
        700: '#374151',
        800: '#1f2937',
        900: '#111827',
        950: '#030712',
      },

      primary: {
        50: '#eff6ff',
        500: '#3b82f6',
        600: '#2563eb',
        700: '#1d4ed8',
        900: '#1e3a8a',
      },

      danger: {
        50: '#fef2f2',
        500: '#ef4444',
        600: '#dc2626',
        700: '#b91c1c',
      },

      success: {
        50: '#f0fdf4',
        500: '#22c55e',
        600: '#16a34a',
        700: '#15803d',
      },
    },

    // Bounded font size scale
    fontSize: {
      xs: ['0.75rem', { lineHeight: '1rem' }],
      sm: ['0.875rem', { lineHeight: '1.25rem' }],
      base: ['1rem', { lineHeight: '1.5rem' }],
      lg: ['1.125rem', { lineHeight: '1.75rem' }],
      xl: ['1.25rem', { lineHeight: '1.75rem' }],
      '2xl': ['1.5rem', { lineHeight: '2rem' }],
      '3xl': ['1.875rem', { lineHeight: '2.25rem' }],
      '4xl': ['2.25rem', { lineHeight: '2.5rem' }],
    },

    // Constrained border radius
    borderRadius: {
      none: '0',
      sm: '0.125rem',
      DEFAULT: '0.25rem',
      md: '0.375rem',
      lg: '0.5rem',
      xl: '0.75rem',
      '2xl': '1rem',
      full: '9999px',
    },
  },
};
```

**Guidelines:**
- Use design system tokens exclusively (no `[...]` arbitrary values in production)
- Define bounded color palettes in `tailwind.config.js` (5-10 semantic colors max)
- Constrain spacing scale to standard increments
- Limit font sizes to 6-8 predefined steps
- Use `eslint-plugin-tailwindcss` to enforce design token usage
- Allow arbitrary values only in prototyping, never in production

---

### Rule 3: Controlled Allocation — Component Extraction Over Class Bloat

**Original Intent:** Prevent unbounded memory growth and allocation failures.

**TailwindCSS Adaptation:**

```html
<!-- BAD: Inline class bloat (200+ characters) -->
<button class="inline-flex items-center justify-center rounded-md border border-transparent bg-blue-600 px-4 py-2 text-sm font-medium text-white shadow-sm hover:bg-blue-700 focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2 disabled:cursor-not-allowed disabled:opacity-50 active:scale-95 transition-all duration-150 ease-in-out">
  Submit
</button>

<!-- BAD: Repeated complex class strings -->
<div class="flex items-center justify-between rounded-lg border border-gray-200 bg-white p-6 shadow-sm">...</div>
<div class="flex items-center justify-between rounded-lg border border-gray-200 bg-white p-6 shadow-sm">...</div>
<div class="flex items-center justify-between rounded-lg border border-gray-200 bg-white p-6 shadow-sm">...</div>

<!-- BAD: @apply overuse (defeats utility-first) -->
<style>
  .btn {
    @apply inline-flex items-center justify-center rounded-md border border-transparent
      px-4 py-2 text-sm font-medium shadow-sm focus:outline-none focus:ring-2
      focus:ring-offset-2 disabled:cursor-not-allowed disabled:opacity-50
      active:scale-95 transition-all duration-150 ease-in-out;
  }
  .btn-primary {
    @apply bg-blue-600 text-white hover:bg-blue-700 focus:ring-blue-500;
  }
  /* Don't recreate entire frameworks with @apply */
</style>
```

```typescript
// GOOD: Extract components with CVA (Class Variance Authority)
import { cva, type VariantProps } from 'class-variance-authority';

const button = cva(
  // Base styles (always applied)
  [
    'inline-flex items-center justify-center',
    'rounded-md',
    'text-sm font-medium',
    'transition-all duration-150',
    'focus:outline-none focus:ring-2 focus:ring-offset-2',
    'disabled:cursor-not-allowed disabled:opacity-50',
  ],
  {
    variants: {
      variant: {
        primary: 'bg-primary-600 text-white hover:bg-primary-700 focus:ring-primary-500',
        secondary: 'bg-gray-100 text-gray-900 hover:bg-gray-200 focus:ring-gray-500',
        danger: 'bg-danger-600 text-white hover:bg-danger-700 focus:ring-danger-500',
        ghost: 'bg-transparent text-gray-700 hover:bg-gray-100 focus:ring-gray-500',
      },
      size: {
        sm: 'px-3 py-1.5 text-xs',
        md: 'px-4 py-2 text-sm',
        lg: 'px-6 py-3 text-base',
      },
    },
    defaultVariants: {
      variant: 'primary',
      size: 'md',
    },
  }
);

type ButtonProps = VariantProps<typeof button> & {
  children: React.ReactNode;
  onClick?: () => void;
  disabled?: boolean;
};

export function Button({ variant, size, children, onClick, disabled }: ButtonProps) {
  return (
    <button
      className={button({ variant, size })}
      onClick={onClick}
      disabled={disabled}
    >
      {children}
    </button>
  );
}

// Usage: <Button variant="primary" size="lg">Submit</Button>
```

```tsx
// GOOD: Composition over duplication
function Card({ children, className = '' }: { children: React.ReactNode; className?: string }) {
  return (
    <div className={`rounded-lg border border-gray-200 bg-white p-6 shadow-sm ${className}`}>
      {children}
    </div>
  );
}

function CardHeader({ children }: { children: React.ReactNode }) {
  return (
    <div className="mb-4 flex items-center justify-between">
      {children}
    </div>
  );
}

function CardTitle({ children }: { children: React.ReactNode }) {
  return <h3 className="text-lg font-semibold text-gray-900">{children}</h3>;
}

// Usage:
<Card>
  <CardHeader>
    <CardTitle>Profile</CardTitle>
  </CardHeader>
  <p>User information goes here</p>
</Card>
```

```typescript
// GOOD: Lookup maps for dynamic styles (no string concatenation)
const statusStyles = {
  success: 'bg-success-50 text-success-700 border-success-200',
  warning: 'bg-yellow-50 text-yellow-700 border-yellow-200',
  error: 'bg-danger-50 text-danger-700 border-danger-200',
  info: 'bg-blue-50 text-blue-700 border-blue-200',
} as const;

type Status = keyof typeof statusStyles;

function StatusBadge({ status }: { status: Status }) {
  return (
    <span className={`inline-flex items-center rounded-full border px-3 py-1 text-xs font-medium ${statusStyles[status]}`}>
      {status}
    </span>
  );
}
```

**Guidelines:**
- Maximum 150-200 characters of classes per element before extracting a component
- Use CVA (class-variance-authority) or tailwind-variants for type-safe variant components
- Use `@apply` sparingly (max 3-5 instances in entire codebase, only for base resets)
- Extract components instead of creating class abstraction layers
- Use lookup objects for dynamic styles (never `bg-${color}-500` string concatenation)
- Composition over class bloat: build small, focused components

---

### Rule 4: Short Class Lists — Maximum 15 Utilities Per Element

**Original Intent:** Ensure functions are small enough to understand, test, and verify.

**TailwindCSS Adaptation:**

```html
<!-- BAD: 30+ utilities on single element -->
<div class="flex flex-col items-start justify-between w-full max-w-4xl mx-auto p-6 space-y-4 text-base leading-relaxed bg-white dark:bg-gray-900 border border-gray-200 dark:border-gray-700 rounded-lg shadow-sm hover:shadow-md transition-shadow duration-200 sm:flex-row sm:items-center md:p-8 lg:max-w-6xl xl:p-12">
  Too many utilities, hard to review
</div>

<!-- BAD: Complex responsive variations inline -->
<button class="px-2 py-1 text-xs sm:px-3 sm:py-1.5 sm:text-sm md:px-4 md:py-2 md:text-base lg:px-6 lg:py-3 lg:text-lg xl:px-8 xl:py-4 xl:text-xl">
  5 breakpoints × 3 properties = unmaintainable
</button>

<!-- GOOD: Extract to component when > 15 utilities -->
<ProductCard product={product} />

<!-- GOOD: Break down complex elements into sub-components -->
<div class="flex items-center justify-between p-6 bg-white border-b border-gray-200">
  <ProductHeader title={title} subtitle={subtitle} />
  <ProductActions id={id} />
</div>

<!-- GOOD: Semantic wrapper components for common layouts -->
<Container>
  <Stack spacing="lg">
    <Heading size="xl">{title}</Heading>
    <Text>{description}</Text>
  </Stack>
</Container>
```

```typescript
// GOOD: Component wrappers for complex layouts
interface ContainerProps {
  children: React.ReactNode;
  maxWidth?: 'sm' | 'md' | 'lg' | 'xl' | 'full';
}

const maxWidthMap = {
  sm: 'max-w-2xl',
  md: 'max-w-4xl',
  lg: 'max-w-6xl',
  xl: 'max-w-7xl',
  full: 'max-w-full',
};

function Container({ children, maxWidth = 'lg' }: ContainerProps) {
  return (
    <div className={`mx-auto w-full px-4 sm:px-6 lg:px-8 ${maxWidthMap[maxWidth]}`}>
      {children}
    </div>
  );
}

// GOOD: Spacing component
interface StackProps {
  children: React.ReactNode;
  spacing?: 'sm' | 'md' | 'lg' | 'xl';
  direction?: 'row' | 'col';
}

const spacingMap = {
  sm: 'gap-2',
  md: 'gap-4',
  lg: 'gap-6',
  xl: 'gap-8',
};

const directionMap = {
  row: 'flex-row',
  col: 'flex-col',
};

function Stack({ children, spacing = 'md', direction = 'col' }: StackProps) {
  return (
    <div className={`flex ${directionMap[direction]} ${spacingMap[spacing]}`}>
      {children}
    </div>
  );
}

// GOOD: Responsive grid component
function ResponsiveGrid({ children }: { children: React.ReactNode }) {
  return (
    <div className="grid grid-cols-1 gap-6 sm:grid-cols-2 lg:grid-cols-3">
      {children}
    </div>
  );
}
```

**Guidelines:**
- Maximum 10-15 utility classes per element before extraction
- Extract components when class lists exceed one line (~150 chars)
- Create semantic layout components (`Container`, `Stack`, `Grid`) for common patterns
- Break complex elements into smaller sub-components
- Use component composition over inline complexity
- Limit responsive variants (prefer 2-3 breakpoints: `sm:`, `md:`, `lg:`)

---

### Rule 5: Visual Regression Testing — Screenshot Tests for All Breakpoints

**Original Intent:** Defensive programming catches bugs early; assertions document invariants.

**TailwindCSS Adaptation:**

```typescript
// BAD: No visual testing (styles break silently)
// Manual QA only, regressions slip through

// BAD: Testing classes as strings (brittle)
test('button has correct classes', () => {
  render(<Button variant="primary" />);
  expect(screen.getByRole('button')).toHaveClass('bg-blue-600 text-white');
  // Breaks when implementation changes, doesn't test visual output
});

// GOOD: Playwright screenshot tests for components
// tests/components/Button.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Button Component', () => {
  test('primary variant - light mode', async ({ page }) => {
    await page.goto('/storybook-iframe?id=button--primary');
    await expect(page).toHaveScreenshot('button-primary-light.png');
  });

  test('primary variant - dark mode', async ({ page }) => {
    await page.emulateMedia({ colorScheme: 'dark' });
    await page.goto('/storybook-iframe?id=button--primary');
    await expect(page).toHaveScreenshot('button-primary-dark.png');
  });

  test('primary variant - hover state', async ({ page }) => {
    await page.goto('/storybook-iframe?id=button--primary');
    await page.hover('button');
    await expect(page).toHaveScreenshot('button-primary-hover.png');
  });

  test('primary variant - disabled state', async ({ page }) => {
    await page.goto('/storybook-iframe?id=button--disabled');
    await expect(page).toHaveScreenshot('button-primary-disabled.png');
  });
});

// GOOD: Responsive breakpoint testing
test.describe('Responsive Layout', () => {
  const breakpoints = [
    { name: 'mobile', width: 375, height: 667 },
    { name: 'tablet', width: 768, height: 1024 },
    { name: 'desktop', width: 1280, height: 720 },
    { name: 'wide', width: 1920, height: 1080 },
  ];

  for (const { name, width, height } of breakpoints) {
    test(`renders correctly at ${name} (${width}x${height})`, async ({ page }) => {
      await page.setViewportSize({ width, height });
      await page.goto('/products');
      await expect(page).toHaveScreenshot(`products-${name}.png`);
    });
  }
});

// GOOD: Accessibility testing with axe-core
import { injectAxe, checkA11y } from 'axe-playwright';

test.describe('Accessibility', () => {
  test('button meets WCAG AA', async ({ page }) => {
    await page.goto('/storybook-iframe?id=button--primary');
    await injectAxe(page);
    await checkA11y(page, null, {
      rules: {
        'color-contrast': { enabled: true },
      },
    });
  });
});
```

**Guidelines:**
- Use Playwright or Chromatic for visual regression testing
- Test all component variants (primary, secondary, danger, etc.)
- Test all states (default, hover, focus, disabled, active)
- Test light and dark modes
- Test 3-4 key breakpoints (mobile 375px, tablet 768px, desktop 1280px, wide 1920px)
- Test color contrast ratios programmatically (WCAG AA minimum 4.5:1)
- Integrate visual tests into CI pipeline

---

### Rule 6: Smallest Scope — Component-Scoped Styles, CSS Variables for Themes

**Original Intent:** Reduce state complexity and potential for misuse.

**TailwindCSS Adaptation:**

```css
/* BAD: Global utility overrides (affects entire app) */
@layer utilities {
  .text-primary {
    color: #3b82f6 !important;  /* Overrides everywhere */
  }
  .bg-custom {
    background: linear-gradient(to right, #ff0000, #00ff00) !important;
  }
}

/* BAD: Leaky global styles */
body {
  font-size: 14px;  /* Overrides Tailwind's text-base */
  color: #333;      /* Overrides text-gray-900 */
}

h1 {
  font-size: 32px;  /* Conflicts with text-3xl */
}
```

```html
<!-- BAD: Inline styles bypassing design system -->
<div style="background-color: #ff6b9d; padding: 23px;">
  Unthemed, unconstrained
</div>
```

```css
/* GOOD: CSS variables for theming (scoped to :root) */
@layer base {
  :root {
    --color-primary-50: 239 246 255;
    --color-primary-500: 59 130 246;
    --color-primary-600: 37 99 235;
    --color-primary-900: 30 58 138;

    --color-gray-50: 249 250 251;
    --color-gray-900: 17 24 39;
  }

  .dark {
    --color-primary-50: 30 58 138;
    --color-primary-500: 96 165 250;
    --color-primary-600: 59 130 246;
    --color-primary-900: 239 246 255;

    --color-gray-50: 17 24 39;
    --color-gray-900: 249 250 251;
  }
}
```

**tailwind.config.js — Using CSS Variables:**
```javascript
export default {
  theme: {
    extend: {
      colors: {
        primary: {
          50: 'rgb(var(--color-primary-50) / <alpha-value>)',
          500: 'rgb(var(--color-primary-500) / <alpha-value>)',
          600: 'rgb(var(--color-primary-600) / <alpha-value>)',
          900: 'rgb(var(--color-primary-900) / <alpha-value>)',
        },
      },
    },
  },
};
```

```html
<!-- GOOD: Component-scoped utilities -->
<article class="prose prose-lg max-w-none dark:prose-invert">
  <!-- Scoped to this article -->
  <h1>Heading</h1>
  <p>Content with proper typography</p>
</article>

<!-- GOOD: Data attributes for component-specific variants -->
<div data-state="open" class="data-[state=open]:opacity-100 data-[state=closed]:opacity-0">
  Scoped animation state
</div>
```

```typescript
// GOOD: Component-level variant classes (no global pollution)
import { tv } from 'tailwind-variants';

const alert = tv({
  base: 'rounded-lg border p-4',
  variants: {
    variant: {
      info: 'border-blue-200 bg-blue-50 text-blue-900 dark:border-blue-800 dark:bg-blue-950 dark:text-blue-100',
      success: 'border-success-200 bg-success-50 text-success-900',
      warning: 'border-yellow-200 bg-yellow-50 text-yellow-900',
      error: 'border-danger-200 bg-danger-50 text-danger-900',
    },
  },
});

// Scoped to Alert component only
export function Alert({ variant, children }: AlertProps) {
  return <div className={alert({ variant })}>{children}</div>;
}
```

**Guidelines:**
- No global utility overrides with `@layer utilities` (keep utilities pure)
- Use CSS variables in `:root` for theme-level values
- Scope all custom styles to components, not global selectors
- Avoid mixing inline `style` with Tailwind (pick one system)
- Use `dark:` variant for dark mode, not separate CSS files
- Component-level variants stay in component code (CVA/tailwind-variants)
- Limit global `@layer base` to true resets only

---

### Rule 7: Check All Outputs — Test Responsive, Dark Mode, RTL, Accessibility

**Original Intent:** Never ignore errors; verify inputs at trust boundaries.

**TailwindCSS Adaptation:**

```html
<!-- BAD: Missing dark mode styles -->
<div class="bg-white text-gray-900">
  <!-- Invisible in dark mode! -->
</div>

<!-- BAD: No focus styles (accessibility violation) -->
<button class="bg-blue-500 text-white">
  <!-- Keyboard users can't see focus -->
</button>

<!-- BAD: Hard-coded LTR spacing (breaks in RTL) -->
<div class="ml-4">
  <!-- Should be ms-4 (margin-start) for RTL support -->
</div>

<!-- BAD: Missing responsive breakpoints -->
<div class="grid grid-cols-4">
  <!-- Breaks on mobile (4 columns too narrow) -->
</div>

<!-- BAD: Color-only information (accessibility issue) -->
<span class="text-red-500">Error</span>
<!-- Screen readers don't know this is an error -->

<!-- GOOD: Dark mode support -->
<div class="bg-white text-gray-900 dark:bg-gray-900 dark:text-gray-50">
  Works in both modes
</div>

<!-- GOOD: Comprehensive focus styles -->
<button class="
  bg-blue-600 text-white
  hover:bg-blue-700
  focus:outline-none focus:ring-2 focus:ring-blue-500 focus:ring-offset-2
  dark:focus:ring-offset-gray-900
">
  Accessible focus indicator
</button>

<!-- GOOD: RTL support with logical properties -->
<div class="ms-4 me-2">
  <!-- margin-inline-start and margin-inline-end -->
</div>

<!-- GOOD: Responsive grid -->
<div class="grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
  <!-- Adapts to screen size -->
</div>

<!-- GOOD: Semantic error with icon and aria -->
<div class="flex items-center gap-2 text-danger-600" role="alert">
  <AlertCircleIcon class="h-5 w-5" aria-hidden="true" />
  <span>Error: Invalid input</span>
</div>
```

```typescript
// GOOD: Accessibility-first input component
interface InputProps {
  label: string;
  error?: string;
  required?: boolean;
}

function Input({ label, error, required }: InputProps) {
  const id = useId();
  const errorId = `${id}-error`;

  return (
    <div>
      <label htmlFor={id} className="block text-sm font-medium text-gray-700 dark:text-gray-300">
        {label}
        {required && <span className="text-danger-500" aria-label="required">*</span>}
      </label>

      <input
        id={id}
        type="text"
        aria-invalid={!!error}
        aria-describedby={error ? errorId : undefined}
        className={`
          mt-1 block w-full rounded-md border px-3 py-2 shadow-sm
          focus:outline-none focus:ring-2 focus:ring-offset-2
          ${error
            ? 'border-danger-300 focus:border-danger-500 focus:ring-danger-500'
            : 'border-gray-300 focus:border-primary-500 focus:ring-primary-500'
          }
          dark:bg-gray-800 dark:border-gray-600 dark:text-gray-100
        `}
      />

      {error && (
        <p id={errorId} className="mt-1 text-sm text-danger-600 dark:text-danger-400" role="alert">
          {error}
        </p>
      )}
    </div>
  );
}
```

**Guidelines:**
- Always include `dark:` variants for backgrounds and text colors
- Test focus states with keyboard navigation (use `focus:ring-*`)
- Use logical properties (`ms-*`, `me-*`, `ps-*`, `pe-*`) for RTL support
- Test responsive layouts at 3-4 breakpoints (375px, 768px, 1280px, 1920px)
- Validate color contrast ratios (WCAG AA: 4.5:1 normal, 3:1 large text)
- Include semantic HTML and ARIA attributes alongside visual styles
- Provide focus-visible styles (not just focus)

---

### Rule 8: Limit Metaprogramming — No Dynamic Class Construction, Use Lookup Maps

**Original Intent:** Avoid constructs that create unmaintainable, unanalyzable code.

**TailwindCSS Adaptation:**

```typescript
// BAD: Dynamic class construction (PurgeCSS/Tailwind can't detect)
function Badge({ color }: { color: string }) {
  return (
    <span className={`bg-${color}-100 text-${color}-800 border-${color}-200`}>
      {/* These classes won't be generated! */}
    </span>
  );
}

// BAD: String concatenation for variants
const buttonClass = 'bg-' + variant + '-600 text-white';
// Won't work! Tailwind needs complete class names.

// BAD: Template literals for dynamic styles
const spacing = `p-${size}`;  // p-${size} is not a valid class

// BAD: Conditional logic spaghetti
const className = `
  ${isPrimary ? 'bg-blue-600' : isSecondary ? 'bg-gray-600' : isDanger ? 'bg-red-600' : 'bg-green-600'}
  ${isSmall ? 'px-2 py-1 text-xs' : isMedium ? 'px-4 py-2 text-sm' : 'px-6 py-3 text-base'}
  ${isRounded ? 'rounded-full' : 'rounded-md'}
`;
```

```typescript
// GOOD: Explicit lookup maps (Tailwind sees all classes)
const colorStyles = {
  blue: 'bg-blue-100 text-blue-800 border-blue-200',
  green: 'bg-green-100 text-green-800 border-green-200',
  red: 'bg-red-100 text-red-800 border-red-200',
  yellow: 'bg-yellow-100 text-yellow-800 border-yellow-200',
} as const;

type Color = keyof typeof colorStyles;

function Badge({ color }: { color: Color }) {
  return (
    <span className={`inline-flex items-center rounded-full px-3 py-1 text-xs font-medium ${colorStyles[color]}`}>
      {color}
    </span>
  );
}

// GOOD: CVA for complex variants
import { cva } from 'class-variance-authority';

const button = cva(
  'inline-flex items-center justify-center font-medium transition-colors focus:outline-none focus:ring-2 focus:ring-offset-2 disabled:opacity-50 disabled:pointer-events-none',
  {
    variants: {
      variant: {
        primary: 'bg-primary-600 text-white hover:bg-primary-700 focus:ring-primary-500',
        secondary: 'bg-gray-100 text-gray-900 hover:bg-gray-200 focus:ring-gray-500',
        danger: 'bg-danger-600 text-white hover:bg-danger-700 focus:ring-danger-500',
        ghost: 'bg-transparent hover:bg-gray-100 focus:ring-gray-500',
      },
      size: {
        sm: 'h-8 px-3 text-xs',
        md: 'h-10 px-4 text-sm',
        lg: 'h-12 px-6 text-base',
      },
    },
    compoundVariants: [
      {
        variant: 'primary',
        size: 'lg',
        className: 'shadow-lg',
      },
    ],
    defaultVariants: {
      variant: 'primary',
      size: 'md',
    },
  }
);

// GOOD: clsx/cn for conditional classes (explicit strings only)
import { clsx } from 'clsx';
import { twMerge } from 'tailwind-merge';

function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}

function Alert({ isError, children }: { isError: boolean; children: React.ReactNode }) {
  return (
    <div
      className={cn(
        'rounded-lg border p-4',
        isError
          ? 'border-danger-200 bg-danger-50 text-danger-900'
          : 'border-blue-200 bg-blue-50 text-blue-900'
      )}
    >
      {children}
    </div>
  );
}
```

**Guidelines:**
- Never use string interpolation for class names (`bg-${color}-500`)
- Use lookup objects with complete class strings
- Use CVA or tailwind-variants for variant-heavy components
- All class names must be statically analyzable (visible in source code)
- Use `clsx` + `twMerge` (cn helper) for conditional application only
- Avoid `safelist` unless working with CMS/external content
- Prefer exhaustive variant definitions over dynamic construction

---

### Rule 9: Type Safety — Type-Safe Variants with CVA, No String Concatenation

**Original Intent:** (C: Restrict pointer usage for safety)

**TailwindCSS Adaptation:**

```typescript
// BAD: Untyped class strings
function Button({ variant, size, children }: any) {
  return (
    <button className={`btn-${variant} btn-${size}`}>
      {children}
    </button>
  );
}

// Usage: <Button variant="primry" size="mdium" /> — typos not caught!

// BAD: Loose className prop typing
function Card({ className }: { className?: string }) {
  return <div className={`card ${className}`} />;
}

// Usage: <Card className="not-a-real-tailwind-class" /> — no validation
```

```typescript
// GOOD: CVA with inferred TypeScript types
import { cva, type VariantProps } from 'class-variance-authority';

const button = cva(
  'inline-flex items-center justify-center font-medium transition-colors',
  {
    variants: {
      variant: {
        primary: 'bg-primary-600 text-white hover:bg-primary-700',
        secondary: 'bg-gray-200 text-gray-900 hover:bg-gray-300',
        danger: 'bg-danger-600 text-white hover:bg-danger-700',
        ghost: 'bg-transparent hover:bg-gray-100',
      },
      size: {
        sm: 'h-8 px-3 text-xs',
        md: 'h-10 px-4 text-sm',
        lg: 'h-12 px-6 text-base',
      },
    },
    defaultVariants: {
      variant: 'primary',
      size: 'md',
    },
  }
);

// Type-safe props inferred from CVA config
interface ButtonProps extends VariantProps<typeof button> {
  children: React.ReactNode;
  onClick?: () => void;
  disabled?: boolean;
}

export function Button({ variant, size, children, onClick, disabled }: ButtonProps) {
  return (
    <button
      className={button({ variant, size })}
      onClick={onClick}
      disabled={disabled}
    >
      {children}
    </button>
  );
}

// Usage: <Button variant="primry" /> — TypeScript error!
//        <Button variant="primary" size="xlarge" /> — TypeScript error!

// GOOD: Explicit variant lookup with const assertion
const statusVariants = {
  success: 'bg-success-50 text-success-700 border-success-200',
  warning: 'bg-yellow-50 text-yellow-700 border-yellow-200',
  error: 'bg-danger-50 text-danger-700 border-danger-200',
  info: 'bg-blue-50 text-blue-700 border-blue-200',
} as const;

type Status = keyof typeof statusVariants;

interface StatusBadgeProps {
  status: Status;
  children: React.ReactNode;
}

export function StatusBadge({ status, children }: StatusBadgeProps) {
  return (
    <span className={`inline-flex items-center rounded-full border px-3 py-1 text-xs font-medium ${statusVariants[status]}`}>
      {children}
    </span>
  );
}

// Usage: <StatusBadge status="pending" /> — TypeScript error (not in union)!

// GOOD: Type-safe responsive props
type Spacing = 'none' | 'sm' | 'md' | 'lg' | 'xl';

const spacingMap: Record<Spacing, string> = {
  none: 'p-0',
  sm: 'p-2',
  md: 'p-4',
  lg: 'p-6',
  xl: 'p-8',
};

interface BoxProps {
  children: React.ReactNode;
  padding?: Spacing;
}

function Box({ children, padding = 'md' }: BoxProps) {
  return <div className={spacingMap[padding]}>{children}</div>;
}
```

**ESLint Configuration:**
```javascript
// eslint.config.js
export default [
  {
    plugins: {
      tailwindcss: require('eslint-plugin-tailwindcss'),
    },
    rules: {
      'tailwindcss/classnames-order': 'warn',
      'tailwindcss/no-custom-classname': 'warn',
      'tailwindcss/no-contradicting-classname': 'error',
    },
  },
];
```

**Guidelines:**
- Use CVA or tailwind-variants for type-safe variant props
- Infer TypeScript types with `VariantProps<typeof variant>`
- Use `as const` assertions for lookup objects
- Validate variant values at compile time (no `any` or `string`)
- Avoid loose `className?: string` props without validation
- Use ESLint plugin-tailwindcss for static class name validation

---

### Rule 10: Static Analysis — Prettier Plugin, Tailwind IntelliSense, Purge Verification

**Original Intent:** Catch issues at development time; use every available tool.

**TailwindCSS Adaptation:**

**prettier.config.js:**
```javascript
export default {
  plugins: ['prettier-plugin-tailwindcss'],
  tailwindConfig: './tailwind.config.js',
  tailwindFunctions: ['clsx', 'cn', 'cva'],
};
```

**eslint.config.js:**
```javascript
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import tailwindcss from 'eslint-plugin-tailwindcss';

export default tseslint.config(
  js.configs.recommended,
  ...tseslint.configs.strict,
  {
    plugins: { tailwindcss },
    rules: {
      'tailwindcss/classnames-order': 'warn',
      'tailwindcss/enforces-negative-arbitrary-values': 'error',
      'tailwindcss/enforces-shorthand': 'warn',
      'tailwindcss/no-arbitrary-value': 'warn',
      'tailwindcss/no-contradicting-classname': 'error',
      'tailwindcss/no-custom-classname': ['warn', {
        whitelist: ['prose', 'prose-lg'],
      }],
      '@typescript-eslint/no-explicit-any': 'error',
    },
    settings: {
      tailwindcss: {
        callees: ['clsx', 'cn', 'cva'],
        config: 'tailwind.config.js',
      },
    },
  }
);
```

**tailwind.config.js — Production Optimization:**
```javascript
export default {
  content: [
    './index.html',
    './src/**/*.{js,ts,jsx,tsx,svelte}',
    // Be explicit about where Tailwind should look
  ],
  theme: {
    extend: {},
  },
  plugins: [
    require('@tailwindcss/forms'),
    require('@tailwindcss/typography'),
  ],
  future: {
    hoverOnlyWhenSupported: true,
  },
};
```

**VSCode settings.json:**
```json
{
  "editor.quickSuggestions": {
    "strings": true
  },
  "tailwindCSS.experimental.classRegex": [
    ["cva\\(([^)]*)\\)", "[\"'`]([^\"'`]*).*?[\"'`]"],
    ["cn\\(([^)]*)\\)", "(?:'|\"|`)([^']*)(?:'|\"|`)"]
  ],
  "tailwindCSS.validate": true,
  "css.validate": false
}
```

**CI Pipeline:**
```bash
# Required CI checks
npx prettier --check src/           # Class ordering
npx eslint --max-warnings 0 src/    # Zero warnings policy
npx tsc --noEmit                    # Type checking
npx playwright test                 # Visual regression
npm run build                       # Build succeeds, purge works
```

**Verify Purge is Working:**
```bash
# Check final CSS size (should be < 20kb gzipped)
npm run build
ls -lh dist/assets/*.css

# Analyze bundle
npx vite-bundle-visualizer
```

**Guidelines:**
- Install `prettier-plugin-tailwindcss` for automatic class ordering
- Enable Tailwind IntelliSense VSCode extension for autocomplete
- Use `eslint-plugin-tailwindcss` with strict rules enabled
- Zero warnings policy in CI (`--max-warnings 0`)
- Verify CSS purge is working (final CSS < 20kb gzipped)
- Configure IntelliSense to recognize CVA/cn functions
- Run visual regression tests in CI
- Audit bundle size regularly

---

## Summary: TailwindCSS Adaptation

| # | Original Rule | TailwindCSS Guideline |
|---|---------------|------------------------|
| 1 | No goto/recursion | Consistent class order, no conflicting utilities |
| 2 | Fixed loop bounds | Bounded design tokens, no arbitrary values |
| 3 | No dynamic allocation | Component extraction over class bloat, use CVA |
| 4 | 60-line functions | Max 15 utilities per element, extract components |
| 5 | 2+ assertions/function | Visual regression tests, screenshot tests, a11y tests |
| 6 | Minimize scope | Component-scoped styles, CSS variables for themes |
| 7 | Check returns | Test responsive, dark mode, RTL, accessibility |
| 8 | Limit preprocessor | No dynamic class construction, use CVA/lookup maps |
| 9 | Restrict pointers | Type-safe variants with CVA, no string concatenation |
| 10 | All warnings enabled | prettier-plugin-tailwindcss, ESLint, purge verification |

---

## References

- [Original Power of 10 Paper](https://spinroot.com/gerard/pdf/P10.pdf) — Gerard Holzmann
- [Tailwind CSS Documentation](https://tailwindcss.com/docs) — Official Docs
- [Tailwind CSS Reusing Styles](https://tailwindcss.com/docs/reusing-styles) — Component Extraction
- [Class Variance Authority (CVA)](https://cva.style/docs) — Type-safe variants
- [tailwind-variants](https://www.tailwind-variants.org/) — Enhanced variant API
- [prettier-plugin-tailwindcss](https://github.com/tailwindlabs/prettier-plugin-tailwindcss) — Automatic class sorting
- [eslint-plugin-tailwindcss](https://github.com/francoismassart/eslint-plugin-tailwindcss) — Linting rules
- [Tailwind IntelliSense](https://marketplace.visualstudio.com/items?itemName=bradlc.vscode-tailwindcss) — VSCode extension
- [WCAG 2.1 Guidelines](https://www.w3.org/WAI/WCAG21/quickref/) — Accessibility standards
