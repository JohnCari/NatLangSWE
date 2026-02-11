# SvelteShip Stack

> Ship fast with Svelte — auth, database, and payments out of the box

---

## Contents

- [The Stack](#the-stack)
- [Why This Stack?](#why-this-stack)
- [Server vs Client](#server-vs-client)
- [Supabase Integration](#supabase-integration)
- [Stripe Checkout Integration](#stripe-checkout-integration)
- [When to Use SvelteShip](#when-to-use-svelteship)
- [Quick Start](#quick-start)
- [Resources](#resources)

---

## What is SvelteShip?

SvelteShip is a curated tech stack built around **SvelteKit** and **Supabase** for shipping products fast. Supabase handles both the backend and auth in one service, Stripe Checkout handles payments, and Vercel handles deployment. Every piece is chosen for minimal boilerplate, tight integration, and generous free tiers.

---

## The Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Hosting** | [Vercel](https://vercel.com) | Zero-config deployment, edge functions, preview URLs |
| **Frontend** | [SvelteKit](https://svelte.dev/docs/kit) | File-based routing, server load functions, form actions |
| **Styling** | [Tailwind CSS](https://tailwindcss.com) | Utility-first CSS, minimal bundle size |
| **Components** | [shadcn-svelte](https://shadcn-svelte.com) | Port of shadcn/ui for Svelte, accessible defaults |
| **Backend + Auth** | [Supabase](https://supabase.com) | PostgreSQL database, Auth, Row Level Security, Edge Functions |
| **Payments** | [Stripe Checkout](https://stripe.com/checkout) | Hosted checkout, subscriptions, customer portal |
| **Tooling** | [Bun](https://bun.sh) + [Turborepo](https://turborepo.dev) | Fast runtime, monorepo orchestration |
| **Linting** | [Biome](https://biomejs.dev) | Linter + formatter (10-20x faster than ESLint) |

---

## Why This Stack?

### Seamless Integration

| Integration | Benefit |
|-------------|---------|
| Vercel + SvelteKit | First-party adapter, automatic edge/serverless split |
| Supabase + SvelteKit | SSR auth helpers, server-side client in hooks |
| Supabase Auth + RLS | Auth and database security in one service |
| Stripe Checkout | No custom payment UI — redirect to Stripe-hosted page |

### Developer Experience

- TypeScript end-to-end with Svelte 5 Runes
- Svelte's compiler — less code, smaller bundles, faster runtime
- Type-safe database queries with Supabase generated types
- Preview deployments on every PR
- Form actions for progressive enhancement — forms work without JS

### Cost-Effective

| Service | Free Tier |
|---------|-----------|
| Vercel | Generous hobby tier |
| Supabase | 500MB database, 50,000 MAUs, 1GB storage |
| Stripe | Pay only on transactions (2.9% + 30c) |

### Fewer Moving Parts

Unlike stacks that split auth and database across services, SvelteShip uses Supabase for **both**. This means:

- One dashboard for users and data
- Auth sessions automatically scoped to database queries via RLS
- No syncing user data between auth provider and database
- One fewer service to configure, monitor, and pay for

---

## Server vs Client

**Default to server-side.** Only add client interactivity when necessary.

| Use Server (`+page.server.ts`) | Use Client (`$state`) |
|---------------------------------|-----------------------|
| Database queries via Supabase | Form inputs, UI toggles |
| Auth checks | Search/filter on loaded data |
| Data mutations via form actions | Animations and transitions |
| Stripe session creation | Real-time subscriptions |

> Keep components server-rendered by default. Use `$state` only for genuinely interactive UI.

See [SvelteKit Preferences](../../frameworks/sveltekit/PREFERENCES.md) for detailed Runes and server-first guidelines.

---

## Supabase Integration

### Server Client in Hooks

Set up the Supabase client in `hooks.server.ts` so every server load function and form action has access:

```typescript
// src/hooks.server.ts
import { createServerClient } from '@supabase/ssr';
import { type Handle, redirect } from '@sveltejs/kit';
import { PUBLIC_SUPABASE_URL, PUBLIC_SUPABASE_ANON_KEY } from '$env/static/public';

export const handle: Handle = async ({ event, resolve }) => {
  event.locals.supabase = createServerClient(
    PUBLIC_SUPABASE_URL,
    PUBLIC_SUPABASE_ANON_KEY,
    {
      cookies: {
        getAll: () => event.cookies.getAll(),
        setAll: (cookiesToSet) => {
          cookiesToSet.forEach(({ name, value, options }) => {
            event.cookies.set(name, value, { ...options, path: '/' });
          });
        },
      },
    },
  );

  event.locals.safeGetSession = async () => {
    const { data: { session } } = await event.locals.supabase.auth.getSession();
    if (!session) return { session: null, user: null };

    const { data: { user }, error } = await event.locals.supabase.auth.getUser();
    if (error) return { session: null, user: null };

    return { session, user };
  };

  return resolve(event, {
    filterSerializedResponseHeaders(name) {
      return name === 'content-range' || name === 'x-supabase-api-version';
    },
  });
};
```

### Type-Safe Queries

Generate types from your Supabase schema:

```bash
bunx supabase gen types typescript --project-id your-project-id > src/lib/types/database.ts
```

```typescript
// src/lib/server/db.ts
import type { Database } from '$lib/types/database';

export type Tables = Database['public']['Tables'];
export type UserRow = Tables['users']['Row'];
export type UserInsert = Tables['users']['Insert'];
```

### Auth-Gated Load Functions

```typescript
// routes/dashboard/+page.server.ts
import { redirect } from '@sveltejs/kit';
import type { PageServerLoad } from './$types';

export const load: PageServerLoad = async ({ locals }) => {
  const { session, user } = await locals.safeGetSession();
  if (!session) {
    redirect(303, '/login');
  }

  const { data: projects, error } = await locals.supabase
    .from('projects')
    .select('id, name, created_at')
    .eq('user_id', user.id)
    .order('created_at', { ascending: false })
    .limit(20);

  if (error) {
    console.error('Failed to load projects:', error);
    return { user, projects: [] };
  }

  return { user, projects };
};
```

### Auth with Form Actions

```typescript
// routes/login/+page.server.ts
import { fail, redirect } from '@sveltejs/kit';
import type { Actions } from './$types';

export const actions = {
  login: async ({ request, locals }) => {
    const formData = await request.formData();
    const email = formData.get('email') as string;
    const password = formData.get('password') as string;

    if (!email || !password) {
      return fail(400, { error: 'Email and password are required', email });
    }

    const { error } = await locals.supabase.auth.signInWithPassword({
      email,
      password,
    });

    if (error) {
      return fail(400, { error: error.message, email });
    }

    redirect(303, '/dashboard');
  },

  signup: async ({ request, locals }) => {
    const formData = await request.formData();
    const email = formData.get('email') as string;
    const password = formData.get('password') as string;

    if (!email || !password) {
      return fail(400, { error: 'Email and password are required', email });
    }

    const { error } = await locals.supabase.auth.signUp({
      email,
      password,
    });

    if (error) {
      return fail(400, { error: error.message, email });
    }

    return { success: true, message: 'Check your email for a confirmation link' };
  },

  logout: async ({ locals }) => {
    await locals.supabase.auth.signOut();
    redirect(303, '/');
  },
} satisfies Actions;
```

### Row Level Security

Always enable RLS. Users can only access their own data:

```sql
-- Enable RLS
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

-- Users can only see their own projects
CREATE POLICY "Users can view own projects"
  ON projects FOR SELECT
  USING (auth.uid() = user_id);

-- Users can only insert their own projects
CREATE POLICY "Users can create own projects"
  ON projects FOR INSERT
  WITH CHECK (auth.uid() = user_id);

-- Users can only update their own projects
CREATE POLICY "Users can update own projects"
  ON projects FOR UPDATE
  USING (auth.uid() = user_id);

-- Users can only delete their own projects
CREATE POLICY "Users can delete own projects"
  ON projects FOR DELETE
  USING (auth.uid() = user_id);
```

---

## Stripe Checkout Integration

### Flow Overview

```
User clicks "Subscribe"
  → SvelteKit form action creates Stripe Checkout Session
    → User redirected to Stripe-hosted checkout page
      → Stripe webhook confirms payment
        → Supabase updates user subscription status
```

### Create Checkout Session (Form Action)

```typescript
// routes/pricing/+page.server.ts
import Stripe from 'stripe';
import { redirect, fail } from '@sveltejs/kit';
import { STRIPE_SECRET_KEY } from '$env/static/private';
import type { Actions } from './$types';

const stripe = new Stripe(STRIPE_SECRET_KEY);

export const actions = {
  checkout: async ({ request, locals, url }) => {
    const { session, user } = await locals.safeGetSession();
    if (!session || !user) {
      redirect(303, '/login');
    }

    const formData = await request.formData();
    const priceId = formData.get('priceId') as string;

    if (!priceId) {
      return fail(400, { error: 'Missing price ID' });
    }

    // Get or create Stripe customer
    const { data: profile } = await locals.supabase
      .from('profiles')
      .select('stripe_customer_id')
      .eq('id', user.id)
      .single();

    let customerId = profile?.stripe_customer_id;

    if (!customerId) {
      const customer = await stripe.customers.create({
        email: user.email,
        metadata: { supabase_user_id: user.id },
      });
      customerId = customer.id;

      await locals.supabase
        .from('profiles')
        .update({ stripe_customer_id: customerId })
        .eq('id', user.id);
    }

    const checkoutSession = await stripe.checkout.sessions.create({
      customer: customerId,
      line_items: [{ price: priceId, quantity: 1 }],
      mode: 'subscription',
      success_url: `${url.origin}/dashboard?checkout=success`,
      cancel_url: `${url.origin}/pricing?checkout=cancelled`,
    });

    if (!checkoutSession.url) {
      return fail(500, { error: 'Failed to create checkout session' });
    }

    redirect(303, checkoutSession.url);
  },
} satisfies Actions;
```

### Webhook Handler

```typescript
// routes/api/webhooks/stripe/+server.ts
import Stripe from 'stripe';
import { json } from '@sveltejs/kit';
import { STRIPE_SECRET_KEY, STRIPE_WEBHOOK_SECRET } from '$env/static/private';
import { createClient } from '@supabase/supabase-js';
import { PUBLIC_SUPABASE_URL } from '$env/static/public';
import { SUPABASE_SERVICE_ROLE_KEY } from '$env/static/private';
import type { RequestHandler } from './$types';

const stripe = new Stripe(STRIPE_SECRET_KEY);

// Use service role for webhook — bypasses RLS
const supabaseAdmin = createClient(PUBLIC_SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY);

export const POST: RequestHandler = async ({ request }) => {
  const body = await request.text();
  const signature = request.headers.get('stripe-signature');

  if (!signature) {
    return json({ error: 'Missing signature' }, { status: 400 });
  }

  let event: Stripe.Event;

  try {
    event = stripe.webhooks.constructEvent(body, signature, STRIPE_WEBHOOK_SECRET);
  } catch (err) {
    console.error('Webhook signature verification failed:', err);
    return json({ error: 'Invalid signature' }, { status: 400 });
  }

  switch (event.type) {
    case 'checkout.session.completed': {
      const session = event.data.object as Stripe.Checkout.Session;
      await supabaseAdmin
        .from('profiles')
        .update({
          subscription_status: 'active',
          stripe_subscription_id: session.subscription as string,
        })
        .eq('stripe_customer_id', session.customer as string);
      break;
    }

    case 'customer.subscription.deleted': {
      const subscription = event.data.object as Stripe.Subscription;
      await supabaseAdmin
        .from('profiles')
        .update({ subscription_status: 'cancelled' })
        .eq('stripe_customer_id', subscription.customer as string);
      break;
    }
  }

  return json({ received: true });
};
```

### Customer Portal

```typescript
// routes/account/+page.server.ts
export const actions = {
  portal: async ({ locals, url }) => {
    const { session, user } = await locals.safeGetSession();
    if (!session || !user) {
      redirect(303, '/login');
    }

    const { data: profile } = await locals.supabase
      .from('profiles')
      .select('stripe_customer_id')
      .eq('id', user.id)
      .single();

    if (!profile?.stripe_customer_id) {
      return fail(400, { error: 'No billing account found' });
    }

    const portalSession = await stripe.billingPortal.sessions.create({
      customer: profile.stripe_customer_id,
      return_url: `${url.origin}/account`,
    });

    redirect(303, portalSession.url);
  },
} satisfies Actions;
```

---

## When to Use SvelteShip

### Use It When

- Validating a product idea quickly
- Building a SaaS with auth + payments
- Solo developer or small team
- You want fewer services to manage (Supabase = auth + DB)
- You need progressive enhancement (forms work without JS)
- You want the smallest possible client-side bundle

### Consider Alternatives When

- Enterprise compliance (SOC2, HIPAA) — unless on Supabase Pro
- Heavy backend processing or long-running jobs
- Offline-first mobile apps
- You need real-time collaborative editing (consider Convex)
- Full infrastructure control needed

---

## Project Structure

```
my-project/
├── apps/
│   └── web/                          # SvelteKit app
│       ├── src/
│       │   ├── lib/
│       │   │   ├── components/       # Shared Svelte components
│       │   │   ├── server/           # Server-only modules
│       │   │   │   └── db.ts         # Supabase typed helpers
│       │   │   ├── state/            # .svelte.ts reactive state
│       │   │   └── types/
│       │   │       └── database.ts   # Generated Supabase types
│       │   ├── routes/
│       │   │   ├── (auth)/
│       │   │   │   ├── login/
│       │   │   │   │   ├── +page.svelte
│       │   │   │   │   └── +page.server.ts
│       │   │   │   └── signup/
│       │   │   ├── (app)/
│       │   │   │   ├── dashboard/
│       │   │   │   │   ├── +page.svelte
│       │   │   │   │   └── +page.server.ts
│       │   │   │   └── account/
│       │   │   ├── api/
│       │   │   │   └── webhooks/
│       │   │   │       └── stripe/
│       │   │   │           └── +server.ts
│       │   │   ├── pricing/
│       │   │   │   ├── +page.svelte
│       │   │   │   └── +page.server.ts
│       │   │   ├── +layout.svelte
│       │   │   ├── +layout.server.ts
│       │   │   └── +page.svelte
│       │   ├── hooks.server.ts       # Supabase client + auth
│       │   └── app.d.ts              # Typed locals
│       ├── svelte.config.js
│       └── vite.config.ts
├── packages/
│   ├── ui/                           # Shared shadcn-svelte components
│   ├── typescript-config/
│   └── biome-config/
├── supabase/
│   ├── migrations/                   # SQL migrations
│   └── config.toml
├── .mcp.json                         # Svelte MCP server
├── turbo.json
├── package.json
└── bun.lockb
```

### App Types (`app.d.ts`)

```typescript
import type { Session, SupabaseClient, User } from '@supabase/supabase-js';
import type { Database } from '$lib/types/database';

declare global {
  namespace App {
    interface Locals {
      supabase: SupabaseClient<Database>;
      safeGetSession: () => Promise<{
        session: Session | null;
        user: User | null;
      }>;
    }
    interface PageData {
      session: Session | null;
    }
  }
}

export {};
```

---

## Quick Start

### 1. Create Monorepo

```bash
bunx create-turbo@latest -m bun
```

### 2. Create SvelteKit App

```bash
cd apps
bun create svelte@latest web
cd web
bun add -D @sveltejs/adapter-vercel
```

### 3. Add Supabase

```bash
bun add @supabase/supabase-js @supabase/ssr
bunx supabase init
bunx supabase gen types typescript --project-id your-project-id > src/lib/types/database.ts
```

### 4. Add Stripe

```bash
bun add stripe
```

### 5. Add UI

```bash
bunx shadcn-svelte@latest init
```

### 6. Add Biome

```bash
bun add -D @biomejs/biome && bunx biome init
```

### 7. Deploy

Connect GitHub repo to [Vercel](https://vercel.com). Set environment variables:

```
PUBLIC_SUPABASE_URL=
PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_SERVICE_ROLE_KEY=
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
```

### 8. Ship It

---

## See Also

- [SvelteKit Power of 10](../../frameworks/sveltekit/POWER_OF_10.md) — Safety-critical coding rules
- [SvelteKit Preferences](../../frameworks/sveltekit/PREFERENCES.md) — Runes, tooling, monorepo config
- [TypeScript Power of 10](../../languages/typescript/POWER_OF_10.md) — TypeScript best practices

---

## Resources

| Category | Links |
|----------|-------|
| **Framework** | [SvelteKit](https://svelte.dev/docs/kit) ・ [Svelte 5 Runes](https://svelte.dev/docs/svelte/what-are-runes) ・ [Tailwind](https://tailwindcss.com/docs) |
| **Components** | [shadcn-svelte](https://shadcn-svelte.com/docs) |
| **Backend + Auth** | [Supabase Docs](https://supabase.com/docs) ・ [Supabase SSR](https://supabase.com/docs/guides/auth/server-side/sveltekit) ・ [Supabase RLS](https://supabase.com/docs/guides/database/postgres/row-level-security) |
| **Payments** | [Stripe Checkout](https://stripe.com/docs/checkout) ・ [Stripe Webhooks](https://stripe.com/docs/webhooks) ・ [Customer Portal](https://stripe.com/docs/billing/subscriptions/integrating-customer-portal) |
| **Tooling** | [Bun](https://bun.sh/docs) ・ [Turborepo](https://turborepo.dev/docs) ・ [Biome](https://biomejs.dev) ・ [Vercel Adapter](https://svelte.dev/docs/kit/adapter-vercel) |
