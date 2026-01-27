# NextJS Monorepo Preferences

> Optimized defaults for NextJS monorepo development

## Core Stack

| Tool | Purpose |
|------|---------|
| **NextJS (TSX)** | Latest stable, App Router |
| **Bun** | Package manager + runtime |
| **Turbopack** | Dev bundler (default in Next.js) |
| **Turborepo** | Monorepo orchestration |

## Server Components

**Default to React Server Components (RSC).** Only use Client Components when absolutely necessary.

### When to Use RSC (Default)
- Data fetching
- Static content
- Database queries
- Backend logic
- Components without interactivity

### When to Use Client Components (`'use client'`)
- Event handlers (onClick, onChange)
- useState, useEffect, useReducer
- Browser APIs (localStorage, window)
- Real-time updates
- Animations and transitions

### Client Boundary Placement

Keep client boundaries **as high as possible** in the component tree:

```tsx
// GOOD: Client boundary at the leaf
// app/page.tsx (Server Component)
export default function Page() {
  const data = await fetchData()  // Server-side fetch
  return (
    <div>
      <StaticHeader />           {/* Server Component */}
      <InteractiveButton />      {/* Client Component - leaf */}
    </div>
  )
}

// BAD: Client boundary too low (wraps everything)
'use client'
export default function Page() { ... }
```

**Push providers deep:**
```tsx
// GOOD: Provider wraps only what needs it
<ServerLayout>
  <ThemeProvider>        {/* Client boundary here */}
    <InteractiveContent />
  </ThemeProvider>
</ServerLayout>
```

## Tooling

### Biome (Recommended)
Faster linter + formatter than ESLint + Prettier combined.

```bash
bun add -D @biomejs/biome
bunx biome init
```

### Turborepo Remote Caching
Cache build artifacts across CI runs. Free on Vercel.

```bash
bunx turbo login
bunx turbo link
```

## Monorepo Structure

```
my-monorepo/
├── apps/
│   ├── web/                 # Main NextJS app
│   └── docs/                # Documentation site (Nextra)
├── packages/
│   ├── ui/                  # Shared Shadcn components
│   ├── typescript-config/   # Shared tsconfig.json
│   └── biome-config/        # Shared Biome config
├── turbo.json
├── package.json
└── bun.lockb
```

## Quick Start

```bash
# Create monorepo with Turborepo + Bun
bunx create-turbo@latest -m bun

# Or clone a starter
git clone https://github.com/gmickel/turborepo-shadcn-nextjs
cd turborepo-shadcn-nextjs
bun install
bun dev
```

## References

- [Next.js Server and Client Components](https://nextjs.org/docs/app/getting-started/server-and-client-components)
- [Turbopack API Reference](https://nextjs.org/docs/app/api-reference/turbopack)
- [Turborepo + Next.js Guide](https://turborepo.dev/docs/guides/frameworks/nextjs)
- [Biome Documentation](https://biomejs.dev/guides/getting-started/)
- [Bun-powered Turborepo Starter](https://github.com/gmickel/turborepo-shadcn-nextjs)
