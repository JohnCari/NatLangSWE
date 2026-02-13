# Coding Standards

Read the relevant natlangswe/ files before writing code.

## Reference Files

- natlangswe/languages/typescript/POWER_OF_10.md
- natlangswe/frameworks/sveltekit/POWER_OF_10.md
- natlangswe/frameworks/svelte5-runes/POWER_OF_10.md
- natlangswe/frameworks/tailwindcss/POWER_OF_10.md
- natlangswe/databases/postgresql/POWER_OF_10.md
- natlangswe/toolchains/bun/POWER_OF_10.md
- natlangswe/frameworks/sveltekit/PREFERENCES.md
- natlangswe/patterns/COMMENTS.md
- natlangswe/patterns/MINIMAL_DEPENDENCIES.md
- natlangswe/patterns/FEATURE_SLICED_DESIGN.md

## Non-Negotiable Defaults

- Always Svelte 5 Runes — never legacy `$:`, `export let`, or writable stores
- Always strict TypeScript — never `any`, always `strictNullChecks`
- Always Bun — never npm/yarn/pnpm
- Always Biome — never ESLint + Prettier
- Always server-first (`+page.server.ts`) — client `$state` only for interactivity
