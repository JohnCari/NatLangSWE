# The Power of 10 Rules for Safety-Critical TypeScript Code

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

## The Power of 10 Rules — TypeScript Edition

### Rule 1: Simple Control Flow — No Recursion

**Original Intent:** Eliminate complex control flow that impedes static analysis and can cause stack overflows.

**TypeScript Adaptation:**

```typescript
// BAD: Recursion (stack overflow risk)
function factorial(n: number): number {
  if (n <= 1) return 1;
  return n * factorial(n - 1);
}

// BAD: Exceptions for flow control
function findUser(id: string): User {
  const user = users.find(u => u.id === id);
  if (!user) throw new NotFoundError();  // Don't use for expected cases
  return user;
}

// GOOD: Iterative implementation
function factorial(n: number): number {
  console.assert(n >= 0, "n must be non-negative");
  let result = 1;
  for (let i = 2; i <= n; i++) {
    result *= i;
  }
  return result;
}

// GOOD: Return types for expected cases
function findUser(id: string): User | undefined {
  return users.find(u => u.id === id);
}
```

**Guidelines:**
- Recursion is forbidden — JavaScript has limited call stack
- Convert recursive algorithms to iterative with explicit stacks
- Don't use exceptions for flow control; use return types
- Catch exceptions immediately around third-party code
- Limit nesting to 3-4 levels maximum

---

### Rule 2: Fixed Loop Bounds — Provable Termination

**Original Intent:** Ensure all loops terminate and can be analyzed statically.

**TypeScript Adaptation:**

```typescript
// BAD: Unbounded loop
while (hasMoreData()) {
  processNext();
}

// BAD: Potentially infinite iteration
for (const item of infiniteGenerator()) {
  process(item);
}

// GOOD: Bounded with explicit limit
const MAX_ITERATIONS = 10_000;

for (let i = 0; i < MAX_ITERATIONS && hasMoreData(); i++) {
  processNext();
}

// GOOD: Bounded array iteration
const items = getItems().slice(0, MAX_ITEMS);
for (const item of items) {
  process(item);
}

// GOOD: Bounded with assertion
function processAll(items: Item[]): void {
  console.assert(items.length <= MAX_ITEMS, `Too many items: ${items.length}`);
  for (const item of items) {
    process(item);
  }
}
```

**Guidelines:**
- All loops must have provable upper bounds
- Use `.slice(0, N)` to enforce maximum iterations
- Define bounds as `const` values
- Prefer `for...of` over `while`
- Add assertions to verify collection sizes

---

### Rule 3: Controlled Memory — Allocate Upfront

**Original Intent:** Prevent unbounded memory growth and allocation failures.

**TypeScript Adaptation:**

```typescript
// BAD: Unbounded array growth
const results: Result[] = [];
for await (const item of stream) {
  results.push(await process(item));  // Memory grows unbounded
}

// BAD: Loading everything into memory
const allUsers = await db.query("SELECT * FROM users");

// GOOD: Paginate data
const PAGE_SIZE = 100;

async function* paginatedQuery<T>(query: string): AsyncGenerator<T> {
  let offset = 0;
  while (true) {
    const page = await db.query<T>(`${query} LIMIT ${PAGE_SIZE} OFFSET ${offset}`);
    if (page.length === 0) break;
    for (const item of page) {
      yield item;
    }
    offset += PAGE_SIZE;
    console.assert(offset < 1_000_000, "Safety limit exceeded");
  }
}

// GOOD: Pre-allocate with fixed size
const buffer = new Array<number>(BUFFER_SIZE).fill(0);

// GOOD: Process in chunks
async function processInChunks<T>(items: T[], chunkSize: number): Promise<void> {
  for (let i = 0; i < items.length; i += chunkSize) {
    const chunk = items.slice(i, i + chunkSize);
    await Promise.all(chunk.map(process));
  }
}
```

**Guidelines:**
- Use pagination for database queries
- Process streams in bounded chunks
- Pre-allocate arrays when size is known
- Avoid unbounded `.push()` in loops
- Set explicit memory limits

---

### Rule 4: Function Length Limit — 60 Lines Maximum

**Original Intent:** Ensure functions are small enough to understand, test, and verify.

**TypeScript Adaptation:**

```typescript
// BAD: Monolithic function
async function processOrder(order: Order): Promise<Result> {
  // 200 lines doing validation, calculation, persistence, notifications...
}

// GOOD: Decomposed functions
async function processOrder(order: Order): Promise<Result> {
  validateOrder(order);
  const total = calculateTotal(order);
  const finalTotal = applyDiscounts(order, total);
  await saveOrder(order, finalTotal);
  return notifyCustomer(order);
}

function validateOrder(order: Order): void {
  // ≤60 lines
  console.assert(order.items.length > 0, "Order must have items");
  console.assert(order.customerId, "Order must have customer");
}

function calculateTotal(order: Order): number {
  // ≤60 lines
  return order.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
}
```

**Guidelines:**
- Maximum 60 lines per function
- Each function does one thing
- Use early returns to reduce nesting
- Extract complex conditionals into named functions
- Keep arrow functions short (prefer named functions for complex logic)

---

### Rule 5: Assertion Density — Minimum 2 Per Function

**Original Intent:** Defensive programming catches bugs early; assertions document invariants.

**TypeScript Adaptation:**

```typescript
// BAD: No defensive checks (relying only on types)
function transfer(from: Account, to: Account, amount: number): void {
  from.balance -= amount;
  to.balance += amount;
}

// GOOD: Runtime assertions beyond type checking
function transfer(from: Account, to: Account, amount: number): void {
  // Preconditions (types can't express these)
  console.assert(amount > 0, "Amount must be positive");
  console.assert(from.balance >= amount, "Insufficient funds");
  console.assert(from.id !== to.id, "Cannot transfer to same account");

  from.balance -= amount;
  to.balance += amount;

  // Postcondition
  console.assert(from.balance >= 0, "Balance went negative");
}

// GOOD: Custom assert function for production
function assert(condition: boolean, message: string): asserts condition {
  if (!condition) {
    throw new Error(`Assertion failed: ${message}`);
  }
}

function divide(a: number, b: number): number {
  assert(b !== 0, "Division by zero");
  const result = a / b;
  assert(Number.isFinite(result), "Result is not finite");
  return result;
}
```

**Guidelines:**
- Minimum 2 assertions per non-trivial function
- Assert preconditions at function entry
- Assert postconditions before returning
- Use `console.assert` or custom assert function
- Types are not enough — assert runtime invariants

---

### Rule 6: Minimize Variable Scope

**Original Intent:** Reduce state complexity and potential for misuse.

**TypeScript Adaptation:**

```typescript
// BAD: Global mutable state
let config: Config;
let currentUser: User | null = null;

function getConfig() {
  return config;
}

// BAD: Variables declared far from use
function process(data: Data): Result {
  let temp: number;
  let result: Result;
  // ... 50 lines ...
  temp = calculate();
  result = transform(temp);
  return result;
}

// GOOD: Encapsulated in classes/modules
class ConfigService {
  private readonly config: Config;

  constructor(config: Config) {
    this.config = Object.freeze(config);
  }

  get(key: keyof Config): string {
    return this.config[key];
  }
}

// GOOD: Variables at point of use, prefer const
function process(data: Data): Result {
  const validated = validate(data);
  const calculated = calculate(validated);
  return transform(calculated);
}

// GOOD: Block scoping for temporary values
function processItems(items: Item[]): Summary {
  let total = 0;
  for (const item of items) {
    const { price, quantity } = item;  // Scoped to loop
    total += price * quantity;
  }
  return { total };
}
```

**Guidelines:**
- Always use `const`; use `let` only when mutation is necessary
- Never use `var`
- Avoid module-level mutable variables
- Declare variables at first use
- Use block scoping `{}` to limit scope
- Use `Object.freeze()` for truly immutable config

---

### Rule 7: Check All Return Values

**Original Intent:** Never ignore errors; verify inputs at trust boundaries.

**TypeScript Adaptation:**

```typescript
// BAD: Ignoring return value
array.map(transform);  // Result unused

// BAD: Non-null assertion
const user = getUser()!;  // Dangerous assumption

// BAD: Swallowing errors
try {
  await riskyOperation();
} catch {
  // Silent failure
}

// GOOD: Handle nullable returns explicitly
const user = getUser();
if (!user) {
  return { error: "User not found" };
}
processUser(user);

// GOOD: Use Result pattern
type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

function divide(a: number, b: number): Result<number, string> {
  if (b === 0) {
    return { ok: false, error: "Division by zero" };
  }
  return { ok: true, value: a / b };
}

// GOOD: Validate at boundaries
function setTemperature(celsius: number): void {
  if (!Number.isFinite(celsius) || celsius < -273.15 || celsius > 1_000_000) {
    throw new RangeError(`Temperature out of range: ${celsius}`);
  }
  internalSetTemp(celsius);
}
```

**Guidelines:**
- Enable `strictNullChecks` in TypeScript
- Never use `!` non-null assertion
- Handle all branches of union types
- Validate inputs at public API boundaries
- Use Result/Either pattern for expected failures
- Let unexpected errors propagate to error boundaries

---

### Rule 8: Limit Metaprogramming

**Original Intent:** Avoid constructs that create unmaintainable, unanalyzable code.

**TypeScript Adaptation:**

```typescript
// BAD: Dynamic code execution
eval(userInput);
new Function("return " + expression)();

// BAD: Excessive decorator stacking
@Log
@Cache
@Retry
@Validate
@Authorize
class Service {}

// BAD: Proxy magic
const obj = new Proxy(target, {
  get(target, prop) { /* magic */ },
  set(target, prop, value) { /* more magic */ }
});

// BAD: Deep property access chains
const value = obj?.foo?.bar?.baz?.qux;

// GOOD: Simple, explicit code
class Service {
  async execute(request: Request): Promise<Response> {
    if (!this.isAuthorized(request)) {
      throw new UnauthorizedError();
    }
    return this.doExecute(request);
  }
}

// GOOD: Limit dereferencing to one level
function getUserCity(user: User): string | undefined {
  const address = user.address;
  if (!address) return undefined;
  return address.city;
}

// GOOD: Explicit over magic
function withLogging<T>(fn: () => T, name: string): T {
  console.log(`Starting ${name}`);
  const result = fn();
  console.log(`Finished ${name}`);
  return result;
}
```

**Guidelines:**
- Never use `eval()` or `new Function()`
- Limit decorators to 1-2 per class/method
- Avoid `Proxy` unless absolutely necessary
- Keep property access to one level of dereferencing
- Prefer explicit code over clever abstractions
- Limit use of `Reflect` API

---

### Rule 9: Type Safety — Strict Mode, No `any`

**Original Intent:** (C: Restrict pointer usage for safety)

**TypeScript Adaptation:**

```typescript
// BAD: Using any
function process(data: any): any {
  return data.foo.bar;
}

// BAD: Type assertions without validation
const user = response.data as User;

// GOOD: Strict typing with unknown
function process(data: unknown): ProcessedData {
  if (!isValidInput(data)) {
    throw new ValidationError("Invalid input");
  }
  return transform(data);
}

// GOOD: Branded types for nominal typing
type UserId = string & { readonly brand: unique symbol };
type OrderId = string & { readonly brand: unique symbol };

function createUserId(id: string): UserId {
  console.assert(id.length > 0, "UserId cannot be empty");
  return id as UserId;
}

// Now these are type errors:
// getUserById(orderId);  // Error!
// getOrderById(userId);  // Error!

// GOOD: Readonly for immutability
interface Config {
  readonly apiUrl: string;
  readonly timeout: number;
}

// GOOD: Discriminated unions over type assertions
type ApiResponse =
  | { status: "success"; data: User }
  | { status: "error"; message: string };

function handleResponse(response: ApiResponse): User | never {
  if (response.status === "error") {
    throw new Error(response.message);
  }
  return response.data;  // TypeScript knows it's User
}
```

**Guidelines:**
- Enable all strict flags in `tsconfig.json`
- Never use `any`; use `unknown` for truly unknown data
- Use branded types for type-safe IDs
- Prefer `readonly` properties
- Use discriminated unions over type assertions
- Validate external data at boundaries with type guards

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

---

### Rule 10: Static Analysis — Zero Warnings

**Original Intent:** Catch issues at development time; use every available tool.

**TypeScript Adaptation:**

```json
// eslint.config.js (flat config)
import tseslint from "typescript-eslint";

export default tseslint.config(
  tseslint.configs.strictTypeChecked,
  {
    rules: {
      "@typescript-eslint/no-explicit-any": "error",
      "@typescript-eslint/no-non-null-assertion": "error",
      "@typescript-eslint/no-unsafe-assignment": "error",
      "@typescript-eslint/no-unsafe-call": "error",
      "@typescript-eslint/no-unsafe-member-access": "error",
      "@typescript-eslint/no-unsafe-return": "error",
      "@typescript-eslint/no-floating-promises": "error",
      "@typescript-eslint/await-thenable": "error",
      "no-console": ["error", { allow: ["assert", "error", "warn"] }]
    }
  }
);
```

```bash
# Required CI pipeline
tsc --noEmit                    # Type checking
eslint --max-warnings 0 src/    # Linting (zero warnings)
prettier --check src/           # Formatting
vitest run                      # Tests
```

**Guidelines:**
- Enable TypeScript strict mode
- Use `typescript-eslint/strict-type-checked` config
- Zero warnings policy — treat warnings as errors
- No `// @ts-ignore` without justification
- No `eslint-disable` without justification
- Run type-aware linting rules

---

## Summary: TypeScript Adaptation

| # | Original Rule | TypeScript Guideline |
|---|---------------|----------------------|
| 1 | No goto/recursion | No recursion, no exceptions for flow control |
| 2 | Fixed loop bounds | Use `.slice()`, assertions, `const` limits |
| 3 | No dynamic allocation | Paginate data, process in chunks |
| 4 | 60-line functions | Single responsibility, early returns |
| 5 | 2+ assertions/function | `console.assert`, runtime checks beyond types |
| 6 | Minimize scope | `const` over `let`, no `var`, no globals |
| 7 | Check returns | strictNullChecks, no `!`, Result pattern |
| 8 | Limit preprocessor | No eval, limit decorators/Proxy |
| 9 | Restrict pointers | Strict mode, no `any`, branded types |
| 10 | All warnings enabled | strict-type-checked, zero warnings |

---

## References

- [Original Power of 10 Paper](https://spinroot.com/gerard/pdf/P10.pdf) — Gerard Holzmann
- [NASA's Power of Ten for JavaScript](https://sambleckley.com/writing/dynamic-powers-of-ten.html) — Sam Bleckley
- [10 Rules for Interpreted Languages](https://dev.to/xowap/10-rules-to-code-like-nasa-applied-to-interpreted-languages-40dd)
- [typescript-eslint Strict Config](https://typescript-eslint.io/users/configs/)
- [TypeScript TSConfig Reference](https://www.typescriptlang.org/tsconfig/)
