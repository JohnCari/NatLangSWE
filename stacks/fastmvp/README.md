# FastMVP Stack

> Ship your MVP in days, not months

## What is FastMVP?

FastMVP is a curated tech stack optimized for building **Minimum Viable Products** (MVP) and **Minimum Evolvable Products** (MEP). Every component is chosen for developer velocity, seamless integration, and generous free tiers.

## The Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Hosting** | [Vercel](https://vercel.com) | Zero-config deployment, edge functions, preview URLs |
| **Frontend** | [NextJS (TSX)](https://nextjs.org) | App Router, Server Components, API routes |
| **Backend** | [Convex](https://convex.dev) or [Supabase](https://supabase.com) | Real-time database, serverless functions |
| **Auth** | [Clerk](https://clerk.com) | Drop-in authentication, user management |
| **Payments** | [Stripe Checkout](https://stripe.com/checkout) | Hosted payment pages, subscriptions |

## Why This Stack?

### Seamless Integration
- Vercel + NextJS: First-party support, automatic optimizations
- Clerk + NextJS: Official SDK with middleware, Server Components support
- Convex/Supabase + Clerk: Built-in auth adapters
- Stripe Checkout: No custom payment UI needed

### Developer Experience
- TypeScript end-to-end
- Hot reload everywhere
- Type-safe database queries (Convex) or auto-generated types (Supabase)
- Preview deployments on every PR

### Cost-Effective for MVPs
- Vercel: Generous hobby tier
- Convex: Free tier handles most MVP traffic
- Supabase: Free tier with 500MB database
- Clerk: 10,000 MAUs free
- Stripe: Pay only on transactions

## Convex vs Supabase

| Factor | Convex | Supabase |
|--------|--------|----------|
| **Best for** | Real-time apps, complex queries | CRUD apps, PostgreSQL familiarity |
| **Database** | Document-based, reactive | PostgreSQL (relational) |
| **Functions** | TypeScript, runs on their infra | Edge Functions (Deno) |
| **Real-time** | Built-in, automatic | Requires setup |
| **Learning curve** | New paradigm | Familiar SQL |

**Choose Convex** for real-time features, collaborative apps, or when you want reactive queries out of the box.

**Choose Supabase** for traditional CRUD, complex relational data, or when you need raw SQL access.

## When to Use FastMVP

**Use it when:**
- Validating a product idea quickly
- Building a SaaS with auth and payments
- Solo developer or small team
- You need to ship in weeks

**Consider alternatives when:**
- Enterprise compliance requirements (SOC2, HIPAA)
- Heavy backend processing (use dedicated servers)
- Offline-first mobile apps
- You need full infrastructure control

## Quick Start Checklist

1. **Create NextJS app**: `npx create-next-app@latest --typescript`
2. **Deploy to Vercel**: Connect GitHub repo
3. **Add Clerk**: `npm install @clerk/nextjs` + wrap app in `<ClerkProvider>`
4. **Add backend**:
   - Convex: `npx convex dev`
   - Supabase: Create project, install `@supabase/supabase-js`
5. **Add Stripe**: Create Checkout Session from API route
6. **Ship it**

## Resources

- [NextJS Documentation](https://nextjs.org/docs)
- [Vercel Documentation](https://vercel.com/docs)
- [Convex Documentation](https://docs.convex.dev)
- [Supabase Documentation](https://supabase.com/docs)
- [Clerk Documentation](https://clerk.com/docs)
- [Stripe Checkout Documentation](https://stripe.com/docs/checkout)
