# The Power of 10 Rules for Safety-Critical Svelte 5 Runes Code

## Background

The **Power of 10 Rules** were created in 2006 by **Gerard J. Holzmann** at NASA's Jet Propulsion Laboratory (JPL) Laboratory for Reliable Software. These rules were designed for writing safety-critical code in C that could be effectively analyzed by static analysis tools.

The rules were incorporated into JPL's institutional coding standard and used for major missions including the **Mars Science Laboratory** (Curiosity Rover, 2012).

> *"If these rules seem draconian at first, bear in mind that they are meant to make it possible to check safety-critical code where human lives can very literally depend on its correctness."* — Gerard Holzmann

**Note:** This adaptation focuses on the **Svelte 5 reactivity system** (runes: `$state`, `$derived`, `$effect`, `$props`, `$bindable`, `$inspect`) and **component patterns** including snippets, `.svelte.ts` modules, and class-based state. For SvelteKit-specific server patterns (load functions, form actions, server routes), see the SvelteKit Power of 10.

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

## The Power of 10 Rules — Svelte 5 Runes Edition

### Rule 1: Simple Control Flow — No Recursive Components, Guard Clauses

**Original Intent:** Eliminate complex control flow that impedes static analysis and can cause stack overflows.

**Svelte 5 Adaptation:**

```svelte
<!-- BAD: Recursive component (stack overflow risk) -->
<!-- TreeNode.svelte -->
<script lang="ts">
  import TreeNode from './TreeNode.svelte';

  let { node }: { node: TreeNode } = $props();
</script>

<div>
  {node.label}
  {#each node.children as child (child.id)}
    <TreeNode node={child} />  <!-- Unbounded recursion -->
  {/each}
</div>

<!-- BAD: Deeply nested conditionals -->
<script lang="ts">
  let { user, settings, permissions }: Props = $props();
</script>

{#if user}
  {#if settings.isActive}
    {#if permissions.canView}
      {#if user.isPremium}
        <PremiumProfile {user} />
      {/if}
    {/if}
  {/if}
{/if}

<!-- GOOD: Guard clauses with flat conditionals -->
<script lang="ts">
  let { user, settings, permissions }: Props = $props();

  const shouldShowProfile = $derived(
    user && settings.isActive && permissions.canView
  );
</script>

{#if !user}
  <p>No user found</p>
{:else if !settings.isActive}
  <InactiveMessage />
{:else if !permissions.canView}
  <AccessDenied />
{:else}
  <Profile {user} />
{/if}

<!-- GOOD: Iterative tree rendering with bounded depth -->
<script lang="ts">
  const MAX_DEPTH = 5;

  let { nodes }: { nodes: TreeNode[] } = $props();

  interface FlatNode extends TreeNode {
    depth: number;
  }

  function flattenTree(items: TreeNode[], depth = 0): FlatNode[] {
    if (depth >= MAX_DEPTH) return [];

    const result: FlatNode[] = [];
    for (const node of items) {
      result.push({ ...node, depth });
      if (node.children && node.children.length > 0) {
        result.push(...flattenTree(node.children, depth + 1));
      }
    }
    return result;
  }

  const flatNodes = $derived(flattenTree(nodes));
</script>

{#each flatNodes as node (node.id)}
  <div style:margin-left="{node.depth * 16}px">
    {node.label}
  </div>
{/each}

<!-- GOOD: Simple snippet logic -->
<script lang="ts">
  let { items }: { items: Item[] } = $props();
</script>

{#snippet itemRow(item: Item)}
  <tr>
    <td>{item.name}</td>
    <td>{item.price}</td>
  </tr>
{/snippet}

<table>
  {#each items as item (item.id)}
    {@render itemRow(item)}
  {/each}
</table>
```

**Guidelines:**
- No recursive components — flatten trees iteratively with depth limits
- Maximum 3-4 levels of `{#if}` nesting — use guard clauses or derived guards
- Extract complex conditional logic into `$derived` values
- Use `{#snippet}` for reusable template fragments without complex logic
- Prefer flat component structures over deeply nested blocks

---

### Rule 2: Bounded Loops — Limit {#each} Iterations

**Original Intent:** Ensure all loops terminate and can be analyzed statically.

**Svelte 5 Adaptation:**

```svelte
<!-- BAD: Unbounded {#each} block -->
<script lang="ts">
  let { users }: { users: User[] } = $props();
</script>

{#each users as user (user.id)}
  <UserCard {user} />  <!-- Could render 100,000 DOM nodes -->
{/each}

<!-- BAD: Derived state without bounds -->
<script lang="ts">
  let { items }: { items: Item[] } = $props();
  let query = $state('');

  const filtered = $derived(
    items.filter(item =>
      item.name.toLowerCase().includes(query.toLowerCase())
    )  // No limit on results
  );
</script>

{#each filtered as item (item.id)}
  <ItemCard {item} />
{/each}

<!-- GOOD: Bounded {#each} with slice -->
<script lang="ts">
  const MAX_VISIBLE = 100;

  let { users }: { users: User[] } = $props();

  const boundedUsers = $derived(users.slice(0, MAX_VISIBLE));
</script>

{#each boundedUsers as user (user.id)}
  <UserCard {user} />
{/each}

{#if users.length > MAX_VISIBLE}
  <p class="overflow-message">
    {users.length - MAX_VISIBLE} more users not shown.
    <a href="/users/all">View all</a>
  </p>
{/if}

<!-- GOOD: Paginated list -->
<script lang="ts">
  const PAGE_SIZE = 20;

  let { items }: { items: Item[] } = $props();
  let currentPage = $state(1);

  const totalPages = $derived(Math.ceil(items.length / PAGE_SIZE));
  const paginatedItems = $derived(
    items.slice((currentPage - 1) * PAGE_SIZE, currentPage * PAGE_SIZE)
  );
</script>

{#each paginatedItems as item (item.id)}
  <ItemCard {item} />
{/each}

<nav>
  <button
    disabled={currentPage === 1}
    onclick={() => currentPage--}
  >
    Previous
  </button>
  <span>Page {currentPage} of {totalPages}</span>
  <button
    disabled={currentPage === totalPages}
    onclick={() => currentPage++}
  >
    Next
  </button>
</nav>

<!-- GOOD: Virtualized list for large datasets -->
<script lang="ts">
  import { createVirtualizer } from '@tanstack/svelte-virtual';

  let { items }: { items: Item[] } = $props();
  let scrollElement: HTMLDivElement;

  const virtualizer = createVirtualizer({
    count: items.length,
    getScrollElement: () => scrollElement,
    estimateSize: () => 50,
    overscan: 5,
  });

  const virtualItems = $derived(virtualizer.getVirtualItems());
  const totalSize = $derived(virtualizer.getTotalSize());
</script>

<div bind:this={scrollElement} style="height: 400px; overflow: auto;">
  <div style="height: {totalSize}px; position: relative;">
    {#each virtualItems as virtualRow (virtualRow.key)}
      <div
        style="position: absolute; top: 0; left: 0; width: 100%; height: {virtualRow.size}px; transform: translateY({virtualRow.start}px);"
      >
        <ItemCard item={items[virtualRow.index]} />
      </div>
    {/each}
  </div>
</div>
```

**Guidelines:**
- Always use `.slice(0, MAX)` before `{#each}` for arrays of unknown size
- Define bounds as constants at module level
- Use virtualization for lists > 100 items
- Assert collection sizes in derived values during development
- Implement pagination for large datasets

---

### Rule 3: Controlled Memory — Minimize $state, Maximize $derived

**Original Intent:** Prevent unbounded memory growth and allocation failures.

**Svelte 5 Adaptation:**

```svelte
<!-- BAD: Duplicating derived state in $state -->
<script lang="ts">
  let { products }: { products: Product[] } = $props();

  let searchTerm = $state('');
  let filteredProducts = $state(products);  // Duplicate state!

  $effect(() => {
    // Storing derived data = memory waste + sync bugs
    filteredProducts = products.filter(p =>
      p.name.toLowerCase().includes(searchTerm.toLowerCase())
    );
  });
</script>

<!-- BAD: Multiple $state for derived values -->
<script lang="ts">
  let count = $state(10);
  let doubled = $state(20);  // Should be derived!
  let quadrupled = $state(40);  // Should be derived!

  $effect(() => {
    doubled = count * 2;
    quadrupled = count * 4;
  });
</script>

<!-- GOOD: Derive values with $derived (no extra state) -->
<script lang="ts">
  let { products }: { products: Product[] } = $props();

  let searchTerm = $state('');

  // Derived during reactivity — no state sync needed
  const filteredProducts = $derived(
    products.filter(p =>
      p.name.toLowerCase().includes(searchTerm.toLowerCase())
    )
  );
</script>

<input bind:value={searchTerm} placeholder="Search..." />

{#each filteredProducts as product (product.id)}
  <ProductCard {product} />
{/each}

<!-- GOOD: Chained derived values -->
<script lang="ts">
  let count = $state(10);

  const doubled = $derived(count * 2);
  const quadrupled = $derived(doubled * 2);
  const isLarge = $derived(count > 100);

  // All derived — no manual synchronization
</script>

<p>Count: {count}</p>
<p>Doubled: {doubled}</p>
<p>Quadrupled: {quadrupled}</p>
{#if isLarge}
  <p class="warning">Count is large!</p>
{/if}
```

```typescript
// GOOD: Extract to .svelte.ts for reusable reactive state
// lib/state/search.svelte.ts
export function createSearch<T>(
  items: () => T[],
  key: (item: T) => string,
) {
  let query = $state('');

  const filtered = $derived(
    items().filter(item =>
      key(item).toLowerCase().includes(query.toLowerCase())
    )
  );

  return {
    get query() { return query; },
    set query(v: string) { query = v; },
    get filtered() { return filtered; },
  };
}
```

```svelte
<!-- Usage in component -->
<script lang="ts">
  import { createSearch } from '$lib/state/search.svelte';

  let { products }: { products: Product[] } = $props();

  const search = createSearch(() => products, (p) => p.name);
</script>

<input bind:value={search.query} />

{#each search.filtered as product (product.id)}
  <ProductCard {product} />
{/each}
```

**Guidelines:**
- Minimize `$state` — ask "can I derive this?" first
- Never store derived data in `$state` (causes sync bugs)
- Use `$derived` for any computed values
- Extract reusable reactive logic into `.svelte.ts` modules
- One source of truth: derive views, don't duplicate data

---

### Rule 4: Short Components — 60 Lines Maximum

**Original Intent:** Ensure functions are small enough to understand, test, and verify.

**Svelte 5 Adaptation:**

```svelte
<!-- BAD: Monolithic component -->
<!-- UserDashboard.svelte (300+ lines) -->
<script lang="ts">
  let { userId }: { userId: string } = $props();

  // 20 $state declarations...
  let user = $state<User | null>(null);
  let posts = $state<Post[]>([]);
  let isLoading = $state(true);
  let error = $state<Error | null>(null);
  // ... 15 more $state calls

  // 15 derived values...
  const userName = $derived(user?.name ?? 'Unknown');
  // ... more derived

  // 10 event handlers...
  function handleEdit() { /* ... */ }
  function handleDelete() { /* ... */ }
  // ... more handlers

  // Multiple effects...
  $effect(() => { /* fetch user */ });
  $effect(() => { /* fetch posts */ });
</script>

<!-- 200 lines of complex markup -->

<!-- GOOD: Decomposed into focused components -->
<!-- UserDashboard.svelte (~40 lines) -->
<script lang="ts">
  let { userId }: { userId: string } = $props();
</script>

<main class="dashboard">
  <UserHeader {userId} />
  <UserStats {userId} />
  <UserPosts {userId} />
  <UserActivity {userId} />
</main>

<!-- GOOD: Focused child component -->
<!-- UserHeader.svelte (~50 lines) -->
<script lang="ts">
  import { useUser } from '$lib/hooks/useUser.svelte';

  let { userId }: { userId: string } = $props();

  const { user, isLoading, error } = useUser(userId);
</script>

{#if isLoading}
  <HeaderSkeleton />
{:else if error}
  <ErrorMessage {error} />
{:else if user}
  <header class="user-header">
    <img src={user.avatarUrl} alt={user.name} />
    <h1>{user.name}</h1>
    <p>{user.email}</p>
  </header>
{/if}
```

```typescript
// GOOD: Extract logic into .svelte.ts files
// lib/hooks/useUser.svelte.ts
export function useUser(userId: string) {
  let user = $state<User | null>(null);
  let isLoading = $state(true);
  let error = $state<Error | null>(null);

  $effect(() => {
    isLoading = true;
    error = null;

    fetchUser(userId)
      .then(data => { user = data; })
      .catch(err => { error = err; })
      .finally(() => { isLoading = false; });
  });

  return {
    get user() { return user; },
    get isLoading() { return isLoading; },
    get error() { return error; },
  };
}
```

```typescript
// GOOD: Extract utilities into plain .ts modules
// lib/utils/validation.ts
export function validateEmail(email: string): boolean {
  console.assert(typeof email === 'string', 'Email must be a string');
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}

export function validatePassword(password: string): boolean {
  console.assert(password.length >= 8, 'Password must be at least 8 chars');
  return password.length >= 8 && /[A-Z]/.test(password) && /[0-9]/.test(password);
}
```

**Guidelines:**
- Maximum 60 lines per `.svelte` component (script + markup + style)
- Extract custom hooks into `.svelte.ts` files (≤50 lines each)
- Extract utilities into plain `.ts` modules
- One component per file
- Keep page components as thin orchestrators

---

### Rule 5: Validation — Type Safety + Zod at Boundaries

**Original Intent:** Defensive programming catches bugs early; assertions document invariants.

**Svelte 5 Adaptation:**

```svelte
<!-- BAD: No prop validation -->
<script lang="ts">
  let { price, currency } = $props();  // No types!
</script>

<span>{currency}{price.toFixed(2)}</span>  <!-- Crashes if price is string -->

<!-- BAD: Trusting external data -->
<script lang="ts">
  let data = $state<User | null>(null);

  $effect(() => {
    fetch('/api/user')
      .then(res => res.json())
      .then(json => { data = json; });  // No validation!
  });
</script>

<!-- GOOD: Typed $props with validation -->
<script lang="ts">
  interface PriceDisplayProps {
    price: number;
    currency: string;
    locale?: string;
  }

  let {
    price,
    currency,
    locale = 'en-US',
  }: PriceDisplayProps = $props();

  $inspect(price);  // Debug in development

  console.assert(typeof price === 'number', `price must be number, got ${typeof price}`);
  console.assert(price >= 0, `price must be non-negative, got ${price}`);
</script>

<span>
  {new Intl.NumberFormat(locale, {
    style: 'currency',
    currency,
  }).format(price)}
</span>

<!-- GOOD: Zod validation for external data -->
<script lang="ts">
  import { z } from 'zod';

  const UserSchema = z.object({
    id: z.string().uuid(),
    email: z.string().email(),
    name: z.string().min(1).max(100),
    role: z.enum(['admin', 'user', 'guest']),
  });

  type User = z.infer<typeof UserSchema>;

  let user = $state<User | null>(null);
  let error = $state<string | null>(null);

  $effect(() => {
    fetch('/api/user')
      .then(res => res.json())
      .then(json => {
        const result = UserSchema.safeParse(json);
        if (result.success) {
          user = result.data;
        } else {
          console.error('Invalid API response:', result.error);
          error = 'Invalid user data received';
        }
      })
      .catch(err => {
        error = err.message;
      });
  });
</script>

{#if error}
  <p class="error">{error}</p>
{:else if user}
  <UserProfile {user} />
{/if}
```

```typescript
// GOOD: Validation utility in .svelte.ts
// lib/state/form.svelte.ts
import { z } from 'zod';

export const ContactFormSchema = z.object({
  email: z.string().email('Invalid email address'),
  message: z.string().min(10, 'Message must be at least 10 characters'),
});

export type ContactFormData = z.infer<typeof ContactFormSchema>;

export function createContactForm() {
  let email = $state('');
  let message = $state('');
  let errors = $state<Record<string, string[]>>({});

  function validate(): boolean {
    const result = ContactFormSchema.safeParse({ email, message });
    if (result.success) {
      errors = {};
      return true;
    }
    errors = result.error.flatten().fieldErrors as Record<string, string[]>;
    return false;
  }

  async function submit() {
    if (!validate()) return;

    await fetch('/api/contact', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ email, message }),
    });
  }

  return {
    get email() { return email; },
    set email(v: string) { email = v; },
    get message() { return message; },
    set message(v: string) { message = v; },
    get errors() { return errors; },
    submit,
  };
}
```

**Guidelines:**
- Validate ALL external inputs with Zod/Valibot
- Type all `$props()` explicitly with interfaces
- Use `$inspect()` for debugging reactive values in development
- Add `console.assert()` for invariants (stripped in production)
- Use `safeParse` — don't throw on invalid data
- Infer TypeScript types from Zod schemas

---

### Rule 6: Minimal Scope — Colocate State, Use Context Sparingly

**Original Intent:** Reduce state complexity and potential for misuse.

**Svelte 5 Adaptation:**

```typescript
// BAD: Global .svelte.ts state for local concerns
// lib/state/global.svelte.ts
export let isModalOpen = $state(false);  // Only used in one component!
export let modalData = $state<any>(null);
```

```svelte
<!-- BAD: Prop drilling through many layers -->
<!-- App.svelte -->
<script lang="ts">
  let user = $state<User | null>(null);
</script>

<Layout {user} />

<!-- Layout.svelte -->
<script lang="ts">
  let { user }: { user: User | null } = $props();
</script>

<Sidebar {user} />

<!-- Sidebar.svelte -->
<script lang="ts">
  let { user }: { user: User | null } = $props();
</script>

<UserMenu {user} />

<!-- UserMenu.svelte — finally used here, 4 levels deep! -->
<script lang="ts">
  let { user }: { user: User | null } = $props();
</script>

<!-- GOOD: Colocate state where it's used -->
<!-- Modal.svelte -->
<script lang="ts">
  let isOpen = $state(false);  // Local to Modal only
</script>

<button onclick={() => isOpen = true}>Open</button>

{#if isOpen}
  <dialog open>
    <ModalContent />
    <button onclick={() => isOpen = false}>Close</button>
  </dialog>
{/if}
```

```typescript
// GOOD: Context for truly shared state
// lib/context/user.svelte.ts
import { getContext, setContext } from 'svelte';

const USER_KEY = Symbol('user');

export function createUserContext(initialUser: User | null) {
  let user = $state<User | null>(initialUser);

  const context = {
    get user() { return user; },
    set user(v: User | null) { user = v; },
  };

  setContext(USER_KEY, context);
  return context;
}

export function getUserContext() {
  const context = getContext<ReturnType<typeof createUserContext>>(USER_KEY);
  if (!context) {
    throw new Error('getUserContext must be called within UserProvider');
  }
  return context;
}
```

```svelte
<!-- App.svelte — set context once at top -->
<script lang="ts">
  import { createUserContext } from '$lib/context/user.svelte';

  let { user }: { user: User | null } = $props();

  createUserContext(user);
</script>

<Layout>
  {@render children()}
</Layout>

<!-- UserMenu.svelte — access anywhere deep in tree -->
<script lang="ts">
  import { getUserContext } from '$lib/context/user.svelte';

  const userContext = getUserContext();
</script>

{#if userContext.user}
  <p>Welcome, {userContext.user.name}!</p>
{/if}
```

```typescript
// GOOD: Class-based state for complex logic
// lib/state/cart.svelte.ts
export class CartState {
  items = $state<CartItem[]>([]);

  get total() {
    return this.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }

  get isEmpty() {
    return this.items.length === 0;
  }

  add(item: CartItem) {
    console.assert(item.quantity > 0, 'Quantity must be positive');
    this.items.push(item);
  }

  remove(id: string) {
    this.items = this.items.filter(item => item.id !== id);
  }

  clear() {
    this.items = [];
  }
}
```

**Guidelines:**
- Colocate `$state` with the component that uses it
- Use Svelte context only for data genuinely shared across a subtree
- Extract reusable reactive logic into `.svelte.ts` modules
- Use class-based state for complex stateful logic
- Avoid global reactive state — prefer props and context

---

### Rule 7: Check Returns — Handle All States

**Original Intent:** Never ignore errors; verify inputs at trust boundaries.

**Svelte 5 Adaptation:**

```svelte
<!-- BAD: Not handling null/undefined -->
<script lang="ts">
  let { user }: { user: User } = $props();
</script>

<div>{user.name}</div>  <!-- Crashes if user is null! -->

<!-- BAD: Ignoring async states -->
<script lang="ts">
  let data = $state<Data | null>(null);

  $effect(() => {
    fetchData().then(d => { data = d; });
    // No loading or error handling
  });
</script>

<div>{data.value}</div>  <!-- Crashes while loading! -->

<!-- GOOD: Handle all states explicitly -->
<script lang="ts">
  let { user }: { user: User | null } = $props();
</script>

{#if !user}
  <p>No user found</p>
{:else}
  <div class="user-profile">
    <h1>{user.name}</h1>
    <p>{user.email}</p>
  </div>
{/if}

<!-- GOOD: Handle loading, error, and success states -->
<script lang="ts">
  type AsyncState<T> =
    | { status: 'idle' }
    | { status: 'loading' }
    | { status: 'error'; error: Error }
    | { status: 'success'; data: T };

  let state = $state<AsyncState<Data>>({ status: 'idle' });

  $effect(() => {
    state = { status: 'loading' };

    fetchData()
      .then(data => { state = { status: 'success', data }; })
      .catch(error => { state = { status: 'error', error }; });
  });
</script>

{#if state.status === 'loading'}
  <LoadingSkeleton />
{:else if state.status === 'error'}
  <ErrorMessage error={state.error} />
{:else if state.status === 'success'}
  <DataDisplay data={state.data} />
{:else}
  <p>No data loaded yet</p>
{/if}

<!-- GOOD: Handle empty collections -->
<script lang="ts">
  let { items }: { items: Item[] } = $props();
</script>

{#if items.length === 0}
  <EmptyState message="No items found" />
{:else}
  <ul>
    {#each items as item (item.id)}
      <ItemCard {item} />
    {/each}
  </ul>
{/if}

<!-- GOOD: Null checks with fallbacks -->
<script lang="ts">
  let { user }: { user: User | null } = $props();
</script>

{#if !user}
  <DefaultAvatar />
{:else if !user.avatarUrl}
  <InitialsAvatar name={user.name} />
{:else}
  <img src={user.avatarUrl} alt={user.name} />
{/if}
```

```typescript
// GOOD: Typed async helper in .svelte.ts
// lib/hooks/useAsync.svelte.ts
export type AsyncState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'error'; error: Error }
  | { status: 'success'; data: T };

export function useAsync<T>(fn: () => Promise<T>) {
  let state = $state<AsyncState<T>>({ status: 'idle' });

  async function execute() {
    state = { status: 'loading' };
    try {
      const data = await fn();
      state = { status: 'success', data };
    } catch (error) {
      state = { status: 'error', error: error as Error };
    }
  }

  return {
    get state() { return state; },
    execute,
  };
}
```

**Guidelines:**
- Always handle `null`/`undefined` with `{#if}` blocks
- Use discriminated unions for async state (not separate booleans)
- Handle loading, error, success, and empty states explicitly
- Never use non-null assertion (`!`) — handle null explicitly
- Provide retry mechanisms for recoverable errors

---

### Rule 8: Limit Metaprogramming — No {@html}, Explicit APIs

**Original Intent:** Avoid constructs that create unmaintainable, unanalyzable code.

**Svelte 5 Adaptation:**

```svelte
<!-- BAD: XSS vulnerability with {@html} -->
<script lang="ts">
  let { userContent }: { userContent: string } = $props();
</script>

<div>{@html userContent}</div>  <!-- User-controlled HTML! -->

<!-- BAD: Dynamic component construction -->
<script lang="ts">
  const componentMap: Record<string, any> = {
    alert: AlertComponent,
    info: InfoComponent,
  };

  let { type }: { type: string } = $props();

  const Component = componentMap[type];  // Untyped!
</script>

<svelte:component this={Component} />

<!-- BAD: eval or dynamic code -->
<script lang="ts">
  let { code }: { code: string } = $props();

  $effect(() => {
    eval(code);  // Never!
  });
</script>

<!-- GOOD: Sanitize if HTML is absolutely required -->
<script lang="ts">
  import DOMPurify from 'isomorphic-dompurify';

  let { html }: { html: string } = $props();

  const sanitized = $derived(
    DOMPurify.sanitize(html, {
      ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'p', 'br', 'a'],
      ALLOWED_ATTR: ['href', 'target', 'rel'],
    })
  );
</script>

<div>{@html sanitized}</div>

<!-- GOOD: Use Svelte components instead of HTML strings -->
<script lang="ts">
  let { text, bold }: { text: string; bold: boolean } = $props();
</script>

{#if bold}
  <strong>{text}</strong>
{:else}
  <span>{text}</span>
{/if}

<!-- GOOD: Explicit component mapping with known set -->
<script lang="ts">
  import Alert from './Alert.svelte';
  import Info from './Info.svelte';
  import Warning from './Warning.svelte';

  const components = {
    alert: Alert,
    info: Info,
    warning: Warning,
  } as const;

  let { type }: { type: keyof typeof components } = $props();

  const Component = $derived(components[type]);
</script>

<Component />

<!-- GOOD: Snippets for composition -->
<script lang="ts">
  import type { Snippet } from 'svelte';

  let {
    header,
    content,
    footer,
  }: {
    header: Snippet;
    content: Snippet;
    footer?: Snippet;
  } = $props();
</script>

<div class="card">
  <header>{@render header()}</header>
  <main>{@render content()}</main>
  {#if footer}
    <footer>{@render footer()}</footer>
  {/if}
</div>
```

**Guidelines:**
- **Never use `{@html}` with user content**
- If HTML is required, sanitize with DOMPurify (whitelist only)
- Never use `eval()` or `new Function()`
- Use explicit component maps with typed keys
- Prefer Svelte snippets over complex metaprogramming
- Avoid `<svelte:component>` with untyped references

---

### Rule 9: Type Safety — Strict TypeScript, Typed $props

**Original Intent:** (C: Restrict pointer usage for safety)

**Svelte 5 Adaptation:**

```svelte
<!-- BAD: Using any -->
<script lang="ts">
  let { user }: { user: any } = $props();
</script>

<div>{user.name}</div>  <!-- No type safety -->

<!-- BAD: Type assertions without validation -->
<script lang="ts">
  let data = fetchData() as User;  // Dangerous assumption
</script>

<!-- BAD: Untyped event handlers -->
<script lang="ts">
  function handleClick(e) {  // Implicit any
    console.log(e.target.value);
  }
</script>

<button onclick={handleClick}>Click</button>

<!-- GOOD: Strictly typed $props -->
<script lang="ts">
  interface UserCardProps {
    user: User;
    onSelect?: (user: User) => void;
    className?: string;
  }

  let {
    user,
    onSelect,
    className = '',
  }: UserCardProps = $props();
</script>

<div class={className} onclick={() => onSelect?.(user)}>
  {user.name}
</div>

<!-- GOOD: Typed event handlers -->
<script lang="ts">
  let { onSearch }: { onSearch: (query: string) => void } = $props();

  let query = $state('');

  function handleChange(e: Event) {
    const target = e.target as HTMLInputElement;
    query = target.value;
    onSearch(target.value);
  }

  function handleSubmit(e: SubmitEvent) {
    e.preventDefault();
    const form = e.target as HTMLFormElement;
    const formData = new FormData(form);
    onSearch(formData.get('query') as string);
  }
</script>

<form onsubmit={handleSubmit}>
  <input name="query" value={query} oninput={handleChange} />
</form>

<!-- GOOD: $bindable with types -->
<script lang="ts">
  interface InputProps {
    value: string;
    placeholder?: string;
  }

  let { value = $bindable(''), placeholder }: InputProps = $props();
</script>

<input bind:value {placeholder} />

<!-- GOOD: Branded types for IDs -->
<script lang="ts">
  type UserId = string & { readonly brand: unique symbol };
  type PostId = string & { readonly brand: unique symbol };

  interface PostViewProps {
    userId: UserId;
    postId: PostId;
  }

  let { userId, postId }: PostViewProps = $props();

  // Can't accidentally swap userId and postId
</script>
```

```typescript
// GOOD: Generic .svelte.ts with type inference
// lib/state/storage.svelte.ts
export function useLocalStorage<T>(key: string, initialValue: T) {
  let value = $state<T>(initialValue);

  $effect(() => {
    const stored = localStorage.getItem(key);
    if (stored) {
      value = JSON.parse(stored) as T;
    }
  });

  $effect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  });

  return {
    get value() { return value; },
    set value(v: T) { value = v; },
  };
}

// Usage: types are inferred
const theme = useLocalStorage('theme', 'light');
//     ^? { value: string }
```

**tsconfig.json:**
```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}
```

**Guidelines:**
- Enable all strict TypeScript options
- Never use `any` — use `unknown` and validate
- Type all `$props()` explicitly with interfaces
- Type all event handlers explicitly
- Use generics for reusable `.svelte.ts` modules
- Use branded types for IDs
- Type `$bindable()` props explicitly

---

### Rule 10: Static Analysis — svelte-check, Biome, Zero Warnings

**Original Intent:** Catch issues at development time; use every available tool.

**Svelte 5 Adaptation:**

**biome.json:**
```json
{
  "$schema": "https://biomejs.dev/schemas/1.9.0/schema.json",
  "organizeImports": {
    "enabled": true
  },
  "linter": {
    "enabled": true,
    "rules": {
      "recommended": true,
      "suspicious": {
        "noExplicitAny": "error"
      },
      "complexity": {
        "noExcessiveCognitiveComplexity": "error"
      },
      "security": {
        "noGlobalEval": "error",
        "noDangerouslySetInnerHtml": "error"
      }
    }
  },
  "formatter": {
    "enabled": true,
    "indentStyle": "tab",
    "lineWidth": 100
  }
}
```

**svelte.config.js:**
```javascript
import adapter from '@sveltejs/adapter-auto';
import { vitePreprocess } from '@sveltejs/vite-plugin-svelte';

export default {
  preprocess: vitePreprocess(),
  kit: {
    adapter: adapter(),
  },
  compilerOptions: {
    runes: true,  // Enforce Svelte 5 runes mode
  },
};
```

**CI Pipeline:**
```bash
# Required CI checks
bunx svelte-check --tsconfig ./tsconfig.json   # Svelte + TypeScript
bunx biome check --max-diagnostics=0 src/      # Zero warnings policy
bunx vitest run                                # Tests
bun run build                                  # Ensure build succeeds
```

```svelte
<!-- GOOD: $inspect for debugging in development -->
<script lang="ts">
  let count = $state(0);

  $inspect(count);  // Logs to console when count changes (dev only)
  $inspect(count).with(console.trace);  // With stack trace
</script>
```

```typescript
// GOOD: Assertion utilities
// lib/utils/assert.ts
export function assertNonNull<T>(
  value: T | null | undefined,
  message: string,
): asserts value is T {
  console.assert(value != null, message);
  if (value == null) {
    throw new Error(message);
  }
}

export function assertValidEmail(email: string): void {
  console.assert(
    /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email),
    `Invalid email: ${email}`,
  );
}
```

**Guidelines:**
- Run `svelte-check` in CI — catches Svelte + TypeScript errors
- Use Biome for linting and formatting (faster than ESLint/Prettier)
- Zero warnings policy in CI
- Use `$inspect()` for debugging reactive values in development
- Enable strict TypeScript checks
- Use `console.assert` for invariants (stripped in production builds)
- No `// @ts-ignore` or `eslint-disable` without justification

---

## Summary: Svelte 5 Runes Adaptation

| # | Original Rule | Svelte 5 Runes Guideline |
|---|---------------|--------------------------|
| 1 | No goto/recursion | No recursive components, guard clauses, simple snippets |
| 2 | Fixed loop bounds | Bounded `{#each}` with `.slice()`, virtualization |
| 3 | No dynamic allocation | Minimize `$state`, maximize `$derived`, avoid duplication |
| 4 | 60-line functions | 60-line `.svelte` components, extract to `.svelte.ts` |
| 5 | 2+ assertions/function | Type safety + Zod validation, `$inspect` for debugging |
| 6 | Minimize scope | Colocate state, use context sparingly, `.svelte.ts` modules |
| 7 | Check returns | Handle loading/error/empty states, typed `$props` |
| 8 | Limit preprocessor | No `{@html}` with user content, explicit component APIs |
| 9 | Restrict pointers | Strict TypeScript, no `any`, typed `$props()` |
| 10 | All warnings enabled | `svelte-check`, Biome, zero warnings, `$inspect` |

---

## References

- [Original Power of 10 Paper](https://spinroot.com/gerard/pdf/P10.pdf) — Gerard Holzmann
- [Svelte 5 Documentation](https://svelte.dev/docs/svelte) — Svelte 5 Runes
- [Svelte 5 Runes Tutorial](https://svelte.dev/tutorial/svelte/introducing-runes) — Interactive Tutorial
- [svelte-check](https://www.npmjs.com/package/svelte-check) — Type checking for Svelte
- [Biome](https://biomejs.dev/) — Fast linter and formatter
- [Zod Documentation](https://zod.dev/) — TypeScript-first validation
- [@tanstack/svelte-virtual](https://tanstack.com/virtual/latest/docs/framework/svelte/svelte-virtual) — Virtualization
