# Feature-Sliced Hexagonal Architecture

> The definitive pattern for parallel AI agent swarm development in a monorepo

---

## Pattern Identity

**Name:** Feature-Sliced Hexagonal Architecture

**Also Known As:**
- Cellular Architecture
- Vertical Hexagonal Slices
- Parallel Agent Architecture

**Combines:**
- **Vertical Slice Architecture** — Feature isolation (each terminal owns a feature)
- **Hexagonal Architecture / Ports & Adapters** — Internal structure within each feature

**Purpose:**
Enable multiple AI agent swarms to work simultaneously without conflicts in a monorepo.

---

## When to Use This Pattern

### Use When:
- Multiple AI agents/terminals working in parallel
- Building a monorepo project
- Features can be developed independently
- Need strict file ownership to prevent conflicts
- Team (human or AI) will grow over time

### Do NOT Use When:
- Single developer/agent project
- Prototype/throwaway code
- Extremely simple CRUD app (< 5 files)

---

## The Complete Workflow

### Phase 1: Ideation

```
YOU: Describe what you want to build to Claude

     "I want to build an e-commerce app with user auth,
      product catalog, shopping cart, and checkout"
```

### Phase 2: Project State Analysis

```
CLAUDE checks:

├── Does a skeleton exist?
│   │
│   ├── NO  → Go to Phase 3 (Create Skeleton)
│   │         ⭐ THIS IS TOP PRIORITY
│   │
│   └── YES → Go to Phase 4 (Feature Breakdown)
```

### Phase 3: Skeleton Creation (New Projects Only)

**This is the most critical phase. Do NOT skip it.**

#### Step 3.1: Create Folder Structure

```bash
mkdir -p src/common/{types,interfaces,constants}
mkdir -p src/features
mkdir -p src/app
```

#### Step 3.2: Create common/ with ALL Shared Types

```
⚠️ THIS IS THE MOST CRITICAL STEP ⚠️

common/
├── types/
│   ├── index.ts           # Re-exports all types
│   ├── user.ts            # User, UserRole, UserProfile, UserSession
│   ├── product.ts         # Product, Category, ProductVariant
│   ├── cart.ts            # Cart, CartItem, CartStatus
│   └── order.ts           # Order, OrderItem, OrderStatus, Payment
├── interfaces/
│   ├── index.ts           # Re-exports all interfaces
│   ├── auth.interface.ts
│   ├── product.interface.ts
│   ├── cart.interface.ts
│   └── order.interface.ts
└── constants/
    ├── index.ts
    └── app.constants.ts
```

**Why common/ Must Be Complete:**

| Without Complete common/ | With Complete common/ |
|--------------------------|----------------------|
| Features define own types | Single source of truth |
| Types become incompatible | All features use same types |
| Merge conflicts guaranteed | No conflicts possible |
| Must stop all agents to fix | Smooth parallel development |

- All features will IMPORT from common/
- common/ becomes **READ-ONLY** after skeleton phase
- If a type is missing, ALL terminals must stop
- Incomplete common/ = guaranteed merge conflicts

#### Step 3.3: Create Feature Folders

```
features/
├── auth/
│   ├── FEATURE.md
│   ├── core/
│   ├── ports/
│   ├── adapters/
│   ├── components/
│   ├── hooks/
│   ├── pages/
│   └── tests/
├── products/
│   └── (same structure)
├── cart/
│   └── (same structure)
└── checkout/
    └── (same structure)
```

#### Step 3.4: Write CLAUDE.md (Global Context)

See [CLAUDE.md Template](#claudemd-template) below.

#### Step 3.5: Write FEATURE.md for Each Feature

See [FEATURE.md Template](#featuremd-template) below.

### Phase 4: Feature Breakdown

Claude outputs feature assignments:

```
"I've broken your idea into the following features:

TERMINAL 1 - Auth Feature
├── Folder: src/features/auth/
├── Imports: User, UserSession from common/
├── Implements: IAuthService
└── Command: 'Implement auth feature per FEATURE.md'

TERMINAL 2 - Products Feature
├── Folder: src/features/products/
├── Imports: Product, Category from common/
├── Implements: IProductService
└── Command: 'Implement products feature per FEATURE.md'

TERMINAL 3 - Cart Feature
├── Folder: src/features/cart/
├── Imports: Cart, CartItem, Product from common/
├── Implements: ICartService
└── Command: 'Implement cart feature per FEATURE.md'

TERMINAL 4 - Checkout Feature
├── Folder: src/features/checkout/
├── Imports: Order, Cart, User from common/
├── Implements: IOrderService
└── Command: 'Implement checkout feature per FEATURE.md'

⚠️ IMPORTANT: common/ is now READ-ONLY
   If you need a new shared type, STOP all terminals
   and add it to common/ first."
```

### Phase 5: Launch Terminal Swarms

```bash
# Terminal 1
cd project && claude "Implement auth feature per FEATURE.md"

# Terminal 2
cd project && claude "Implement products feature per FEATURE.md"

# Terminal 3
cd project && claude "Implement cart feature per FEATURE.md"

# Terminal 4
cd project && claude "Implement checkout feature per FEATURE.md"
```

### Phase 6: Merge & Integration

```
1. Each terminal commits to its feature branch
2. Run tests for each feature independently
3. Assign ONE terminal for integration (see Section 10)
4. Merge feature branches to main
5. Run integration tests
6. Fix any integration issues
```

---

## Architecture Structure

### NextJS/TSX Complete Example

```
my-nextjs-app/
├── CLAUDE.md                          # ⭐ Global AI context - READ THIS FIRST
│
├── src/
│   ├── common/                        # ⭐ Shared code - DEFINE FIRST, READ-ONLY AFTER
│   │   ├── types/
│   │   │   ├── index.ts               # export * from './user'; export * from './product'; ...
│   │   │   ├── user.ts                # User, UserRole, UserProfile, UserSession
│   │   │   ├── product.ts             # Product, Category, ProductVariant, ProductStatus
│   │   │   ├── cart.ts                # Cart, CartItem, CartStatus
│   │   │   └── order.ts               # Order, OrderItem, OrderStatus, PaymentMethod
│   │   │
│   │   ├── interfaces/
│   │   │   ├── index.ts               # export * from './auth.interface'; ...
│   │   │   ├── auth.interface.ts      # IAuthService, IAuthProvider
│   │   │   ├── product.interface.ts   # IProductService, IProductRepository
│   │   │   ├── cart.interface.ts      # ICartService, ICartStorage
│   │   │   └── order.interface.ts     # IOrderService, IPaymentGateway
│   │   │
│   │   └── constants/
│   │       ├── index.ts
│   │       └── app.constants.ts       # API_URL, MAX_CART_ITEMS, etc.
│   │
│   ├── features/
│   │   │
│   │   ├── auth/                      # 🔵 TERMINAL 1's swarm owns this folder
│   │   │   ├── FEATURE.md             # Feature context - agents read this first
│   │   │   │
│   │   │   ├── core/                  # Agent 1: Pure business logic
│   │   │   │   ├── auth.service.ts    # AuthService class implementing IAuthService
│   │   │   │   └── auth.utils.ts      # validateEmail(), hashPassword(), etc.
│   │   │   │
│   │   │   ├── ports/                 # Agent 2: Feature-internal interfaces
│   │   │   │   └── auth.port.ts       # Internal ports NOT in common/
│   │   │   │
│   │   │   ├── adapters/              # Agent 3-4: External implementations
│   │   │   │   ├── auth.api.ts        # Agent 3: API calls to backend
│   │   │   │   └── auth.storage.ts    # Agent 4: LocalStorage, cookies, session
│   │   │   │
│   │   │   ├── components/            # Agent 5: React/TSX UI components
│   │   │   │   ├── LoginForm.tsx
│   │   │   │   ├── SignupForm.tsx
│   │   │   │   ├── ForgotPassword.tsx
│   │   │   │   └── AuthProvider.tsx   # Context provider
│   │   │   │
│   │   │   ├── hooks/                 # Agent 5: React hooks
│   │   │   │   ├── useAuth.ts         # Main auth hook
│   │   │   │   └── useSession.ts      # Session management hook
│   │   │   │
│   │   │   ├── pages/                 # Agent 6: Next.js pages and API routes
│   │   │   │   ├── login.tsx
│   │   │   │   ├── signup.tsx
│   │   │   │   └── api/
│   │   │   │       └── auth/
│   │   │   │           ├── login.ts
│   │   │   │           ├── logout.ts
│   │   │   │           └── [...nextauth].ts
│   │   │   │
│   │   │   └── tests/                 # Agent 7: All tests for this feature
│   │   │       ├── auth.service.test.ts
│   │   │       ├── auth.api.test.ts
│   │   │       └── LoginForm.test.tsx
│   │   │
│   │   ├── products/                  # 🟢 TERMINAL 2's swarm owns this folder
│   │   │   ├── FEATURE.md
│   │   │   ├── core/
│   │   │   ├── ports/
│   │   │   ├── adapters/
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   ├── pages/
│   │   │   └── tests/
│   │   │
│   │   ├── cart/                      # 🟡 TERMINAL 3's swarm owns this folder
│   │   │   ├── FEATURE.md
│   │   │   ├── core/
│   │   │   ├── ports/
│   │   │   ├── adapters/
│   │   │   ├── components/
│   │   │   ├── hooks/
│   │   │   ├── pages/
│   │   │   └── tests/
│   │   │
│   │   └── checkout/                  # 🟣 TERMINAL 4's swarm owns this folder
│   │       ├── FEATURE.md
│   │       ├── core/
│   │       ├── ports/
│   │       ├── adapters/
│   │       ├── components/
│   │       ├── hooks/
│   │       ├── pages/
│   │       └── tests/
│   │
│   └── app/                           # Next.js App Router - INTEGRATION PHASE ONLY
│       ├── layout.tsx                 # Root layout - imports providers from features/
│       ├── page.tsx                   # Home page
│       ├── login/
│       │   └── page.tsx               # Imports from features/auth/components/
│       ├── products/
│       │   ├── page.tsx               # Imports from features/products/components/
│       │   └── [id]/
│       │       └── page.tsx
│       ├── cart/
│       │   └── page.tsx               # Imports from features/cart/components/
│       └── checkout/
│           └── page.tsx               # Imports from features/checkout/components/
│
├── package.json
├── tsconfig.json
├── next.config.js
└── .gitignore
```

---

## File Ownership Rules

### Rule 1: Terminal Ownership

Each terminal/swarm owns ONE feature folder exclusively.

```
✅ Terminal 1 CAN modify:     src/features/auth/**/*
❌ Terminal 1 CANNOT modify:  src/features/products/**/*
❌ Terminal 1 CANNOT modify:  src/features/cart/**/*
❌ Terminal 1 CANNOT modify:  src/common/**/*
❌ Terminal 1 CANNOT modify:  src/app/**/* (integration phase only)
```

### Rule 2: Agent Ownership Within Feature

Within each feature, agents own specific subfolders:

```
Feature: auth/

┌─────────────┬─────────────────────────────────────┐
│ Agent       │ Owns                                │
├─────────────┼─────────────────────────────────────┤
│ Agent 1     │ core/                               │
│ Agent 2     │ ports/                              │
│ Agent 3     │ adapters/auth.api.ts                │
│ Agent 4     │ adapters/auth.storage.ts            │
│ Agent 5     │ components/, hooks/                 │
│ Agent 6     │ pages/                              │
│ Agent 7     │ tests/                              │
└─────────────┴─────────────────────────────────────┘
```

### Rule 3: common/ is READ-ONLY

After skeleton creation:

```
⚠️ common/ is READ-ONLY ⚠️

- NO terminal modifies common/ during implementation
- If new type needed:
  1. STOP all terminals
  2. Add type to common/ with single terminal
  3. Resume all terminals
```

### Rule 4: When Files Must Be Shared

```
If two features need the same type:
  ✅ Put it in common/types/
  ❌ Don't duplicate in both features

If two features need the same interface:
  ✅ Put it in common/interfaces/
  ❌ Don't create feature-specific versions

If two features need the same component:
  ✅ Put it in common/components/ (create if needed)
  ❌ Don't import from another feature's components/
```

---

## Hexagonal Components Explained

Each feature folder uses the Hexagonal Architecture (Ports & Adapters) pattern internally:

| Component | Purpose | Dependencies | Example |
|-----------|---------|--------------|---------|
| **core/** | Pure business logic, no external deps | Only common/types | `auth.service.ts` |
| **ports/** | Feature-internal interfaces | common/interfaces | `auth.port.ts` |
| **adapters/** | External implementations | core/, ports/, libs | `auth.api.ts` |
| **components/** | UI components (TSX) | core/, adapters/, React | `LoginForm.tsx` |
| **hooks/** | React hooks | core/, adapters/ | `useAuth.ts` |
| **pages/** | Route handlers, API endpoints | components/, hooks/ | `login.tsx` |
| **tests/** | All tests for feature | Everything in feature | `*.test.ts` |

### Dependency Direction

```
tests/ ─────────────────────────────────────────┐
                                                │
pages/ ──────────────────────────────────────┐  │
                                             │  │
components/ ──────────────────────────────┐  │  │
                                          │  │  │
hooks/ ────────────────────────────────┐  │  │  │
                                       │  │  │  │
adapters/ ─────────────────────────┐   │  │  │  │
                                   │   │  │  │  │
ports/ ────────────────────────┐   │   │  │  │  │
                               │   │   │  │  │  │
core/ ─────────────────────┐   │   │   │  │  │  │
                           │   │   │   │  │  │  │
                           ▼   ▼   ▼   ▼  ▼  ▼  ▼
                    common/types & common/interfaces


Legend:
  ─────► = "depends on" (arrow points to dependency)

Rules:
  • core/ has NO dependencies except common/
  • adapters/ depends on core/ and ports/
  • components/ can use adapters/ and hooks/
  • tests/ can depend on EVERYTHING in the feature
  • NOTHING depends on tests/
```

---

## Templates

### CLAUDE.md Template

Create this file at the project root:

```markdown
# Project: [PROJECT_NAME]

## Architecture

This project uses **Feature-Sliced Hexagonal Architecture**.

- **Pattern:** Feature-Sliced Hexagonal Architecture
- **Framework:** Next.js with TypeScript
- **Styling:** [Tailwind/CSS Modules/etc.]

## Golden Rules

1. Each feature folder is owned by ONE terminal/swarm
2. `common/` is **READ-ONLY** after skeleton creation
3. **NEVER** modify files outside your assigned feature folder
4. Import ALL shared types from `common/`
5. Read `FEATURE.md` before starting work on any feature

## Project Structure

\`\`\`
src/
├── common/              # Shared types - READ ONLY
│   ├── types/           # TypeScript types
│   ├── interfaces/      # Service interfaces
│   └── constants/       # App constants
├── features/
│   ├── [feature]/       # One terminal per feature
│   │   ├── FEATURE.md   # READ THIS FIRST
│   │   ├── core/        # Business logic
│   │   ├── ports/       # Internal interfaces
│   │   ├── adapters/    # Implementations
│   │   ├── components/  # React components
│   │   ├── hooks/       # React hooks
│   │   ├── pages/       # Next.js pages/API
│   │   └── tests/       # Tests
└── app/                 # Next.js App Router (integration only)
\`\`\`

## Features Overview

| Feature | Description | Terminal |
|---------|-------------|----------|
| auth | User authentication and sessions | 1 |
| products | Product catalog and search | 2 |
| cart | Shopping cart management | 3 |
| checkout | Order processing and payment | 4 |

## Shared Types (common/)

### Types
- `User`, `UserSession`, `UserRole`, `UserProfile`
- `Product`, `Category`, `ProductVariant`
- `Cart`, `CartItem`, `CartStatus`
- `Order`, `OrderItem`, `OrderStatus`, `PaymentMethod`

### Interfaces
- `IAuthService`: login, logout, register, getSession
- `IProductService`: getProducts, getProduct, searchProducts
- `ICartService`: getCart, addItem, removeItem, clearCart
- `IOrderService`: createOrder, getOrder, processPayment

## Adding a New Feature

1. **STOP** all terminals
2. Create feature folder structure
3. Add any new shared types to `common/`
4. Write `FEATURE.md` for the new feature
5. Update this file's Features Overview table
6. Resume terminals with new assignment

## Conventions

- File naming: `kebab-case.ts` for files, `PascalCase.tsx` for components
- Interface naming: `I[Name]` prefix (e.g., `IAuthService`)
- Type naming: `PascalCase` (e.g., `UserSession`)
- Test files: `[name].test.ts` or `[name].test.tsx`
```

### FEATURE.md Template

Create this file in each feature folder:

```markdown
# Feature: [FEATURE_NAME]

## Description

[What this feature does in 2-3 sentences. Be specific about the functionality.]

## Terminal Assignment

This feature is assigned to: **Terminal [N]**

⚠️ Only Terminal [N] may modify files in this folder.

## Imports from common/

### Types
- `common/types/user.ts`: `User`, `UserSession`
- `common/types/[other].ts`: `[Type1]`, `[Type2]`

### Interfaces
- `common/interfaces/[feature].interface.ts`: `I[Feature]Service`

## Agent Assignments

| Folder | Agent | Files to Create | Responsibility |
|--------|-------|-----------------|----------------|
| core/ | 1 | `[feature].service.ts`, `[feature].utils.ts` | Business logic |
| ports/ | 2 | `[feature].port.ts` | Internal interfaces |
| adapters/ | 3 | `[feature].api.ts` | API calls |
| adapters/ | 4 | `[feature].storage.ts` | Local storage |
| components/ | 5 | `[Component1].tsx`, `[Component2].tsx` | UI components |
| hooks/ | 5 | `use[Feature].ts` | React hooks |
| pages/ | 6 | `[page].tsx`, `api/[route].ts` | Routes and API |
| tests/ | 7 | `*.test.ts`, `*.test.tsx` | All tests |

## Interfaces to Implement

From `common/interfaces/[feature].interface.ts`:

\`\`\`typescript
interface I[Feature]Service {
  method1(param: Type): Promise<ReturnType>;
  method2(param: Type): Promise<ReturnType>;
  method3(): Promise<ReturnType>;
}
\`\`\`

## Dependencies on Other Features

- **None** — This feature is independent
- OR: Depends on `common/` types only (no feature-to-feature deps)

## Acceptance Criteria

- [ ] [Criterion 1 - specific and testable]
- [ ] [Criterion 2 - specific and testable]
- [ ] [Criterion 3 - specific and testable]
- [ ] All tests pass
- [ ] No TypeScript errors

## API Endpoints (if applicable)

| Method | Endpoint | Request Body | Response | Description |
|--------|----------|--------------|----------|-------------|
| POST | `/api/[feature]/[action]` | `{ field: type }` | `{ result: type }` | [Description] |
| GET | `/api/[feature]/[id]` | - | `{ data: type }` | [Description] |

## Component List

| Component | Props | Description |
|-----------|-------|-------------|
| `[Component1]` | `{ prop1: type }` | [What it renders] |
| `[Component2]` | `{ prop1: type }` | [What it renders] |

## Notes

[Any special considerations, gotchas, or implementation details]
```

### Skeleton Creation Script

Save as `create-skeleton.sh`:

```bash
#!/bin/bash
# create-skeleton.sh - Creates Feature-Sliced Hexagonal Architecture skeleton
#
# Usage: ./create-skeleton.sh [project-name] [feature1] [feature2] ...
# Example: ./create-skeleton.sh my-app auth products cart checkout

set -e

PROJECT_NAME=${1:-"my-project"}
shift
FEATURES=${@:-"auth products cart checkout"}

echo "Creating skeleton for: $PROJECT_NAME"
echo "Features: $FEATURES"
echo ""

# Create common/
echo "Creating common/ folder..."
mkdir -p src/common/{types,interfaces,constants}
touch src/common/types/index.ts
touch src/common/interfaces/index.ts
touch src/common/constants/index.ts

# Create feature folders
for feature in $FEATURES; do
  echo "Creating feature: $feature"
  mkdir -p src/features/$feature/{core,ports,adapters,components,hooks,pages,tests}
  touch src/features/$feature/FEATURE.md
  touch src/features/$feature/core/.gitkeep
  touch src/features/$feature/ports/.gitkeep
  touch src/features/$feature/adapters/.gitkeep
  touch src/features/$feature/components/.gitkeep
  touch src/features/$feature/hooks/.gitkeep
  touch src/features/$feature/pages/.gitkeep
  touch src/features/$feature/tests/.gitkeep
done

# Create app folder
echo "Creating app/ folder..."
mkdir -p src/app
touch src/app/layout.tsx
touch src/app/page.tsx

# Create root files
touch CLAUDE.md

echo ""
echo "✅ Skeleton created for $PROJECT_NAME"
echo ""
echo "Next steps:"
echo "1. Define types in src/common/types/"
echo "2. Define interfaces in src/common/interfaces/"
echo "3. Write CLAUDE.md (global context)"
echo "4. Write FEATURE.md for each feature"
echo "5. Launch terminal swarms"
```

---

## Anti-Patterns

**What NOT to do:**

| Anti-Pattern | Why It's Bad | Correct Approach |
|--------------|--------------|------------------|
| Start coding without skeleton | Agents create conflicting structures | Always create skeleton first |
| Two agents edit same file | Merge conflicts, lost work | Strict folder ownership |
| Define types in feature folder | Other features can't use them | Put shared types in `common/` |
| Modify `common/` during implementation | Breaks other terminals | Stop all, modify, resume all |
| Skip FEATURE.md | Agents don't know boundaries | Always write FEATURE.md first |
| Feature imports from another feature | Tight coupling, conflicts | Depend only on `common/` |
| Put business logic in components | Hard to test, mixed concerns | Put in `core/`, import in components |
| No tests folder | Can't verify feature works | Always include tests/ |
| Modify `app/` during feature work | Integration conflicts | Only modify during integration phase |
| Incomplete `common/` types | Agents define conflicting types | Define ALL shared types upfront |

---

## Handling Edge Cases

### Edge Case 1: Need New Shared Type Mid-Development

```
Situation: During implementation, you realize a shared type is missing.

Solution:
1. STOP all terminals (save work, don't commit yet)
2. One terminal adds type to common/types/[file].ts
3. Update common/types/index.ts exports
4. All terminals pull latest changes
5. Resume all terminals
```

### Edge Case 2: Feature Needs Another Feature's Component

```
Situation: Cart feature needs a ProductCard component from Products feature.

❌ WRONG:
import { ProductCard } from '../products/components/ProductCard';

✅ RIGHT:
Option A: Move ProductCard to common/components/
Option B: Create a shared 'ui' feature for common components
Option C: Duplicate the component (if truly feature-specific)
```

### Edge Case 3: Merge Conflict Despite Following Rules

```
Situation: Merge conflict occurs even though rules were followed.

Most likely cause: Two terminals modified app/ folder or package.json

Solution:
1. app/ integration happens AFTER all features complete
2. Only ONE terminal handles app/ integration
3. package.json changes coordinated through single terminal
```

### Edge Case 4: Feature Scope Creep

```
Situation: During implementation, feature grows larger than expected.

Solution:
1. STOP the terminal
2. Evaluate if feature should be split
3. Create new feature folder(s) if needed
4. Update FEATURE.md files
5. Assign additional terminal(s) if needed
6. Resume work
```

### Edge Case 5: Agent Needs to Modify Another Agent's File

```
Situation: Agent 5 (components) needs to fix something in Agent 1's (core) file.

Solution:
1. Agent 5 notes the issue in FEATURE.md or creates a TODO
2. Agent 1 fixes the issue in core/
3. Agent 5 continues with updated core/
4. OR: Single agent handles both if swarm is small
```

---

## Integration Phase

After all features are complete, perform integration:

### Integration Workflow

```
1. All feature terminals commit to their feature branches
   git checkout -b feature/auth
   git add .
   git commit -m "Complete auth feature"

2. Run tests for each feature independently
   npm test -- --testPathPattern=features/auth
   npm test -- --testPathPattern=features/products
   # etc.

3. Assign ONE terminal for integration:

   Integration Terminal Tasks:
   ├── Create integration branch
   │   git checkout main
   │   git checkout -b integration
   │
   ├── Merge all feature branches
   │   git merge feature/auth
   │   git merge feature/products
   │   git merge feature/cart
   │   git merge feature/checkout
   │
   ├── Wire up app/ routes
   │   └── Import components from features/ into app/ pages
   │
   ├── Set up providers in layout.tsx
   │   └── Wrap app with AuthProvider, CartProvider, etc.
   │
   ├── Update package.json if needed
   │
   ├── Run full test suite
   │   npm test
   │
   └── Fix any integration issues

4. Other terminals remain available for feature-specific fixes

5. Final merge to main
   git checkout main
   git merge integration
```

### Integration Checklist

```
[ ] All feature branches merged without conflicts
[ ] All providers added to layout.tsx
[ ] All routes properly connected in app/
[ ] All feature tests pass
[ ] Integration tests pass
[ ] No TypeScript errors
[ ] App runs successfully (npm run dev)
[ ] Manual smoke test of all features
```

---

## Quick Reference

### Folder Ownership at a Glance

```
src/
├── common/          → READ-ONLY (skeleton phase only)
├── features/
│   ├── auth/        → Terminal 1 ONLY
│   ├── products/    → Terminal 2 ONLY
│   ├── cart/        → Terminal 3 ONLY
│   └── checkout/    → Terminal 4 ONLY
└── app/             → Integration Terminal ONLY
```

### Agent Assignments at a Glance

```
Within each feature:
├── core/            → Agent 1
├── ports/           → Agent 2
├── adapters/        → Agents 3-4
├── components/      → Agent 5
├── hooks/           → Agent 5
├── pages/           → Agent 6
└── tests/           → Agent 7
```

### The Golden Rules

1. **Skeleton First** — Always create skeleton before implementation
2. **common/ is Sacred** — Read-only after skeleton phase
3. **Own Your Folder** — Never modify files outside your assignment
4. **Import from common/** — All shared types come from common/
5. **Read FEATURE.md** — Always read before starting work

---

## References

- [Vertical Slice Architecture](https://www.jimmybogard.com/vertical-slice-architecture/)
- [Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/)
- [Feature-Sliced Design](https://feature-sliced.design/)
- [Ports and Adapters Pattern](https://herbertograca.com/2017/09/14/ports-adapters-architecture/)
