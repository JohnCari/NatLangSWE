# The Power of 10 Rules for Safety-Critical NextJS Code

## Background

The **Power of 10 Rules** were created in 2006 by **Gerard J. Holzmann** at NASA's Jet Propulsion Laboratory (JPL) Laboratory for Reliable Software. These rules were designed for writing safety-critical code in C that could be effectively analyzed by static analysis tools.

The rules were incorporated into JPL's institutional coding standard and used for major missions including the **Mars Science Laboratory** (Curiosity Rover, 2012).

> *"If these rules seem draconian at first, bear in mind that they are meant to make it possible to check safety-critical code where human lives can very literally depend on its correctness."* — Gerard Holzmann

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

## The Power of 10 Rules — NextJS Edition

### Rule 1: Simple Control Flow — No Recursion, Guard Clauses

**Original Intent:** Eliminate complex control flow that impedes static analysis and can cause stack overflows.

**NextJS Adaptation:**

```tsx
// BAD: Recursive component
function TreeNode({ node }: { node: Node }) {
  return (
    <div>
      {node.name}
      {node.children.map(child => (
        <TreeNode key={child.id} node={child} />  // Recursion risk
      ))}
    </div>
  );
}

// BAD: Deeply nested conditionals
function UserProfile({ user, settings, permissions }) {
  if (user) {
    if (settings.isActive) {
      if (permissions.canView) {
        return <Profile user={user} />;
      }
    }
  }
  return null;
}

// GOOD: Guard clauses with early returns
function UserProfile({ user, settings, permissions }: Props) {
  if (!user) return null;
  if (!settings.isActive) return <InactiveMessage />;
  if (!permissions.canView) return <AccessDenied />;

  return <Profile user={user} />;
}

// GOOD: Iterative tree rendering with bounded depth
function TreeView({ nodes, maxDepth = 5 }: Props) {
  const renderLevel = (items: Node[], depth: number) => {
    if (depth >= maxDepth) return null;

    return items.map(node => (
      <div key={node.id} style={{ marginLeft: depth * 16 }}>
        {node.name}
        {node.children && renderLevel(node.children, depth + 1)}
      </div>
    ));
  };

  return <div>{renderLevel(nodes, 0)}</div>;
}
```

**Guidelines:**
- No recursive components — use iterative rendering with depth limits
- Use guard clauses for early returns
- Maximum 3-4 levels of nesting
- Prefer flat component structures

---

### Rule 2: Bounded Loops — Paginate Data, Limit Iterations

**Original Intent:** Ensure all loops terminate and can be analyzed statically.

**NextJS Adaptation:**

```tsx
// BAD: Unbounded data fetch
async function getAllUsers() {
  const users = await db.user.findMany();  // Could be millions
  return users;
}

// BAD: Unbounded map
function UserList({ users }: { users: User[] }) {
  return users.map(user => <UserCard key={user.id} user={user} />);
}

// GOOD: Paginated Server Component
const PAGE_SIZE = 50;
const MAX_PAGES = 100;

async function UsersPage({ page }: { page: number }) {
  const safePageNumber = Math.min(Math.max(1, page), MAX_PAGES);

  const users = await db.user.findMany({
    skip: (safePageNumber - 1) * PAGE_SIZE,
    take: PAGE_SIZE,
  });

  return (
    <>
      {users.map(user => <UserCard key={user.id} user={user} />)}
      <Pagination current={safePageNumber} max={MAX_PAGES} />
    </>
  );
}

// GOOD: Bounded iteration with slice
const MAX_ITEMS = 100;

function ItemList({ items }: { items: Item[] }) {
  const boundedItems = items.slice(0, MAX_ITEMS);

  console.assert(
    boundedItems.length <= MAX_ITEMS,
    `Items exceed maximum: ${items.length}`
  );

  return boundedItems.map(item => <ItemCard key={item.id} item={item} />);
}
```

**Guidelines:**
- Always paginate database queries
- Use `.slice(0, MAX)` before mapping arrays
- Define bounds as constants
- Assert collection sizes at boundaries

---

### Rule 3: Controlled Memory — Prefer Server Components

**Original Intent:** Prevent unbounded memory growth and allocation failures.

**NextJS Adaptation:**

```tsx
// BAD: Client-side state for server data
'use client';
function Dashboard() {
  const [users, setUsers] = useState<User[]>([]);
  const [posts, setPosts] = useState<Post[]>([]);
  const [comments, setComments] = useState<Comment[]>([]);

  useEffect(() => {
    // Fetches all data to client memory
    fetchAll().then(data => {
      setUsers(data.users);
      setPosts(data.posts);
      setComments(data.comments);
    });
  }, []);
}

// GOOD: Server Component (no client-side memory)
async function Dashboard() {
  const users = await getRecentUsers(10);
  const posts = await getRecentPosts(10);

  return (
    <div>
      <UserList users={users} />
      <PostList posts={posts} />
    </div>
  );
}

// GOOD: Minimal client state
'use client';
function SearchInput() {
  const [query, setQuery] = useState('');  // Only what's needed

  return (
    <input
      value={query}
      onChange={e => setQuery(e.target.value)}
    />
  );
}

// GOOD: Stream large data
import { Suspense } from 'react';

async function LargeDataPage() {
  return (
    <Suspense fallback={<Loading />}>
      <StreamedContent />
    </Suspense>
  );
}
```

**Guidelines:**
- Default to Server Components (no `'use client'`)
- Minimize `useState` — only for truly interactive state
- Avoid `useEffect` for data fetching — use Server Components
- Stream large responses with Suspense
- Don't store server data in client state

---

### Rule 4: Short Components — 60 Lines Maximum

**Original Intent:** Ensure functions are small enough to understand, test, and verify.

**NextJS Adaptation:**

```tsx
// BAD: Monolithic component
function ProductPage({ id }: { id: string }) {
  // 200 lines of JSX, hooks, handlers, and logic...
}

// GOOD: Decomposed into focused components
async function ProductPage({ params }: { params: { id: string } }) {
  const product = await getProduct(params.id);
  if (!product) return notFound();

  return (
    <main>
      <ProductHeader product={product} />
      <ProductGallery images={product.images} />
      <ProductDetails product={product} />
      <AddToCartSection product={product} />
      <RelatedProducts categoryId={product.categoryId} />
    </main>
  );
}

// GOOD: Extract hooks
function useProductForm(initialProduct: Product) {
  const [product, setProduct] = useState(initialProduct);

  const updateField = (field: keyof Product, value: string) => {
    setProduct(prev => ({ ...prev, [field]: value }));
  };

  return { product, updateField };
}

// GOOD: Extract handlers
function createProductHandlers(product: Product, router: AppRouter) {
  return {
    onSave: async () => { /* ≤20 lines */ },
    onDelete: async () => { /* ≤20 lines */ },
    onCancel: () => router.back(),
  };
}
```

**Guidelines:**
- Maximum 60 lines per component
- Extract custom hooks for reusable logic
- Extract handlers into separate functions
- One component per file
- Split pages into layout + content components

---

### Rule 5: Validation — Zod for All Inputs

**Original Intent:** Defensive programming catches bugs early; assertions document invariants.

**NextJS Adaptation:**

```tsx
import { z } from 'zod';

// Define schemas for all external data
const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  name: z.string().min(1).max(100),
  age: z.number().int().min(0).max(150).optional(),
});

type User = z.infer<typeof UserSchema>;

// GOOD: Validate route params
const ParamsSchema = z.object({
  id: z.string().uuid(),
});

async function UserPage({ params }: { params: unknown }) {
  const result = ParamsSchema.safeParse(params);
  if (!result.success) {
    return notFound();
  }

  const user = await getUser(result.data.id);
  // ...
}

// GOOD: Validate Server Action inputs
'use server';

const CreateUserSchema = z.object({
  email: z.string().email(),
  name: z.string().min(1).max(100),
});

async function createUser(formData: FormData) {
  const rawData = Object.fromEntries(formData);
  const result = CreateUserSchema.safeParse(rawData);

  if (!result.success) {
    return { error: result.error.flatten() };
  }

  // Safe to use result.data
  const user = await db.user.create({ data: result.data });
  return { user };
}

// GOOD: Validate API responses
async function fetchUser(id: string): Promise<User | null> {
  const response = await fetch(`/api/users/${id}`);
  const data = await response.json();

  const result = UserSchema.safeParse(data);
  if (!result.success) {
    console.error('Invalid API response:', result.error);
    return null;
  }

  return result.data;
}
```

**Guidelines:**
- Validate ALL external inputs with Zod/Valibot
- Validate route params (`params`, `searchParams`)
- Validate Server Action inputs
- Validate API responses
- Use `safeParse` — don't throw on invalid data
- Infer TypeScript types from schemas

---

### Rule 6: Minimal Scope — Server First, Colocate State

**Original Intent:** Reduce state complexity and potential for misuse.

**NextJS Adaptation:**

```tsx
// BAD: Global client state for server data
'use client';
// store/userStore.ts
export const useUserStore = create((set) => ({
  users: [],
  fetchUsers: async () => { /* ... */ },
}));

// BAD: Unnecessary 'use client'
'use client';  // Not needed!
function StaticContent() {
  return <div>This could be a Server Component</div>;
}

// GOOD: Server Component (default)
async function UserList() {
  const users = await getUsers();  // Data stays on server
  return users.map(user => <UserCard key={user.id} user={user} />);
}

// GOOD: Colocated client state
'use client';
function ExpandableCard({ title, children }: Props) {
  const [isExpanded, setIsExpanded] = useState(false);  // Local only

  return (
    <div>
      <button onClick={() => setIsExpanded(!isExpanded)}>
        {title}
      </button>
      {isExpanded && children}
    </div>
  );
}

// GOOD: Lift state only when necessary
'use client';
function FilterableList({ items }: { items: Item[] }) {
  const [filter, setFilter] = useState('');

  const filtered = items.filter(item =>
    item.name.toLowerCase().includes(filter.toLowerCase())
  );

  return (
    <>
      <SearchInput value={filter} onChange={setFilter} />
      <ItemList items={filtered} />
    </>
  );
}
```

**Guidelines:**
- Default to Server Components — add `'use client'` only when needed
- Colocate state with the component that uses it
- Avoid global state stores for server data
- Use React Context sparingly and close to usage
- Keep client boundaries as small as possible

---

### Rule 7: Check Returns — Handle All States

**Original Intent:** Never ignore errors; verify inputs at trust boundaries.

**NextJS Adaptation:**

```tsx
// BAD: Unchecked fetch
async function UserPage({ params }: Props) {
  const user = await getUser(params.id);
  return <Profile user={user} />;  // user could be null!
}

// BAD: Ignoring error state
'use client';
function DataDisplay() {
  const { data } = useSWR('/api/data', fetcher);
  return <div>{data.value}</div>;  // What if data is undefined?
}

// GOOD: Handle all return states
async function UserPage({ params }: { params: { id: string } }) {
  const user = await getUser(params.id);

  if (!user) {
    return notFound();
  }

  return <Profile user={user} />;
}

// GOOD: Handle loading, error, and success
'use client';
function DataDisplay() {
  const { data, error, isLoading } = useSWR('/api/data', fetcher);

  if (isLoading) return <Skeleton />;
  if (error) return <ErrorMessage error={error} />;
  if (!data) return <EmptyState />;

  return <div>{data.value}</div>;
}

// GOOD: Server Action with explicit return types
'use server';

type ActionResult<T> =
  | { success: true; data: T }
  | { success: false; error: string };

async function updateUser(formData: FormData): Promise<ActionResult<User>> {
  try {
    const validated = UserSchema.safeParse(Object.fromEntries(formData));
    if (!validated.success) {
      return { success: false, error: 'Invalid input' };
    }

    const user = await db.user.update({ /* ... */ });
    return { success: true, data: user };
  } catch (e) {
    return { success: false, error: 'Update failed' };
  }
}
```

**Guidelines:**
- Always handle `null`/`undefined` returns
- Handle loading, error, and empty states
- Use `notFound()` for missing resources
- Return typed results from Server Actions
- Never ignore Promise rejections

---

### Rule 8: Limit Metaprogramming — No dangerouslySetInnerHTML

**Original Intent:** Avoid constructs that create unmaintainable, unanalyzable code.

**NextJS Adaptation:**

```tsx
// BAD: XSS vulnerability
function Comment({ html }: { html: string }) {
  return <div dangerouslySetInnerHTML={{ __html: html }} />;
}

// BAD: Dynamic code execution
function DynamicComponent({ code }: { code: string }) {
  eval(code);  // Never!
  return <div />;
}

// BAD: Excessive HOCs
const EnhancedComponent = withAuth(
  withLogging(
    withErrorBoundary(
      withTheme(
        withI18n(BaseComponent)
      )
    )
  )
);

// GOOD: Sanitize if HTML is absolutely required
import DOMPurify from 'isomorphic-dompurify';

function SafeHTML({ html }: { html: string }) {
  const sanitized = DOMPurify.sanitize(html, {
    ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'p'],
    ALLOWED_ATTR: [],
  });

  return <div dangerouslySetInnerHTML={{ __html: sanitized }} />;
}

// GOOD: Prefer React components over HTML strings
function FormattedText({ text, bold }: Props) {
  return bold ? <strong>{text}</strong> : <span>{text}</span>;
}

// GOOD: Composition over HOCs
function ProtectedPage({ children }: { children: React.ReactNode }) {
  return (
    <ErrorBoundary>
      <AuthGuard>
        <ThemeProvider>
          {children}
        </ThemeProvider>
      </AuthGuard>
    </ErrorBoundary>
  );
}
```

**Guidelines:**
- **Never use `dangerouslySetInnerHTML` with user content**
- If HTML is required, sanitize with DOMPurify (whitelist only)
- Never use `eval()` or `new Function()`
- Prefer composition over HOCs
- Limit dynamic imports for security-critical paths
- Avoid `Proxy` and `Reflect` in components

---

### Rule 9: Type Safety — Strict TypeScript, Typed Server Actions

**Original Intent:** (C: Restrict pointer usage for safety)

**NextJS Adaptation:**

```tsx
// BAD: Using any
async function getUser(id: any): Promise<any> {
  return await db.user.findUnique({ where: { id } });
}

// BAD: Type assertions without validation
const user = data as User;  // Dangerous assumption

// GOOD: Strict typing with Zod inference
import { z } from 'zod';

const UserSchema = z.object({
  id: z.string().uuid(),
  email: z.string().email(),
  name: z.string(),
  role: z.enum(['admin', 'user']),
});

type User = z.infer<typeof UserSchema>;

async function getUser(id: string): Promise<User | null> {
  const data = await db.user.findUnique({ where: { id } });
  if (!data) return null;

  return UserSchema.parse(data);
}

// GOOD: Typed Server Actions
'use server';

interface CreatePostInput {
  title: string;
  content: string;
}

interface CreatePostResult {
  success: boolean;
  post?: Post;
  error?: string;
}

async function createPost(input: CreatePostInput): Promise<CreatePostResult> {
  // Implementation
}

// GOOD: Typed route params
interface PageProps {
  params: { slug: string };
  searchParams: { page?: string };
}

export default async function BlogPost({ params, searchParams }: PageProps) {
  // params.slug is typed
}

// GOOD: Branded types for IDs
type UserId = string & { readonly brand: unique symbol };
type PostId = string & { readonly brand: unique symbol };

function getPost(userId: UserId, postId: PostId) {
  // Can't accidentally swap userId and postId
}
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
    "noUncheckedIndexedAccess": true
  }
}
```

**Guidelines:**
- Enable all strict TypeScript options
- Never use `any` — use `unknown` and validate
- Infer types from Zod schemas
- Type all Server Actions explicitly
- Use branded types for IDs
- Type route params and search params

---

### Rule 10: Static Analysis — ESLint, next lint, Zero Warnings

**Original Intent:** Catch issues at development time; use every available tool.

**NextJS Adaptation:**

```javascript
// eslint.config.mjs
import { FlatCompat } from '@eslint/eslintrc';

const compat = new FlatCompat();

export default [
  ...compat.extends('next/core-web-vitals', 'next/typescript'),
  {
    rules: {
      // Security
      'react/no-danger': 'error',
      'no-eval': 'error',

      // TypeScript strict
      '@typescript-eslint/no-explicit-any': 'error',
      '@typescript-eslint/no-non-null-assertion': 'error',
      '@typescript-eslint/strict-boolean-expressions': 'error',

      // React best practices
      'react-hooks/exhaustive-deps': 'error',
      'react/jsx-no-leaked-render': 'error',
    },
  },
];
```

```bash
# Required CI pipeline
next lint --strict              # NextJS-specific linting
tsc --noEmit                    # Type checking
eslint --max-warnings 0 .       # Zero warnings
prettier --check .              # Formatting
npm audit                       # Dependency vulnerabilities
```

**Security Headers (next.config.js):**
```javascript
const securityHeaders = [
  { key: 'X-Content-Type-Options', value: 'nosniff' },
  { key: 'X-Frame-Options', value: 'DENY' },
  { key: 'X-XSS-Protection', value: '1; mode=block' },
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },
];

module.exports = {
  async headers() {
    return [{ source: '/:path*', headers: securityHeaders }];
  },
};
```

**Guidelines:**
- Run `next lint` with strict mode
- Enable TypeScript strict checks
- Zero warnings policy
- Configure security headers
- Run `npm audit` regularly
- Use Dependabot or Renovate for updates

---

## Summary: NextJS Adaptation

| # | Original Rule | NextJS Guideline |
|---|---------------|------------------|
| 1 | No goto/recursion | No recursive components, guard clauses |
| 2 | Fixed loop bounds | Paginate data, `.slice()` before map |
| 3 | No dynamic allocation | Prefer Server Components, minimize useState |
| 4 | 60-line functions | 60-line components, extract hooks |
| 5 | 2+ assertions/function | Zod validation for all inputs |
| 6 | Minimize scope | Server-first, colocate state |
| 7 | Check returns | Handle loading/error/empty states |
| 8 | Limit preprocessor | No dangerouslySetInnerHTML, no eval |
| 9 | Restrict pointers | Strict TS, no any, branded types |
| 10 | All warnings enabled | next lint strict, zero warnings |

---

## References

- [Original Power of 10 Paper](https://spinroot.com/gerard/pdf/P10.pdf) — Gerard Holzmann
- [Next.js Security Guide](https://nextjs.org/blog/security-nextjs-server-components-actions) — Server Components & Actions
- [Next.js Data Security](https://nextjs.org/docs/app/guides/data-security) — Data Access Layer
- [Arcjet Security Checklist](https://blog.arcjet.com/next-js-security-checklist/) — Security best practices
- [Next.js TypeScript Config](https://nextjs.org/docs/app/api-reference/config/typescript)
