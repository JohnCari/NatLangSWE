# The Power of 10 Rules for Safety-Critical PostgreSQL Code

## Background

The **Power of 10 Rules** were created in 2006 by **Gerard J. Holzmann** at NASA's Jet Propulsion Laboratory (JPL) Laboratory for Reliable Software. These rules were designed for writing safety-critical code in C that could be effectively analyzed by static analysis tools.

The rules were incorporated into JPL's institutional coding standard and used for major missions including the **Mars Science Laboratory** (Curiosity Rover, 2012).

> *"If these rules seem draconian at first, bear in mind that they are meant to make it possible to check safety-critical code where human lives can very literally depend on its correctness."* — Gerard Holzmann

**Note:** This adaptation covers PostgreSQL database development, including safe query patterns, schema design, migrations, transaction handling, and operational practices that prevent data corruption, performance degradation, and production incidents.

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

## The Power of 10 Rules — PostgreSQL Edition

### Rule 1: Simple Query Structure — Use CTEs, Avoid Deep Nesting

**Original Intent:** Eliminate complex control flow that impedes static analysis and reasoning.

**PostgreSQL Adaptation:**

```sql
-- BAD: Deeply nested subqueries
SELECT u.name,
       (SELECT COUNT(*)
        FROM orders o
        WHERE o.user_id = u.id
          AND o.status = (SELECT id FROM statuses WHERE name = 'completed')
          AND o.created_at > (SELECT MAX(created_at) - INTERVAL '30 days'
                              FROM user_activity
                              WHERE user_id = u.id)) AS recent_orders
FROM users u
WHERE u.id IN (SELECT user_id
               FROM subscriptions
               WHERE status = 'active'
                 AND tier IN (SELECT id FROM tiers WHERE premium = true));

-- BAD: Cursors (procedural, hard to reason about)
DO $$
DECLARE
    user_cursor CURSOR FOR SELECT id FROM users;
    user_record RECORD;
BEGIN
    OPEN user_cursor;
    LOOP
        FETCH user_cursor INTO user_record;
        EXIT WHEN NOT FOUND;
        -- Procedural processing row by row
    END LOOP;
    CLOSE user_cursor;
END $$;

-- GOOD: Simple CTEs (Common Table Expressions)
WITH active_premium_users AS (
    SELECT u.id, u.name
    FROM users u
    JOIN subscriptions s ON s.user_id = u.id
    JOIN tiers t ON t.id = s.tier
    WHERE s.status = 'active'
      AND t.premium = true
),
completed_status AS (
    SELECT id FROM statuses WHERE name = 'completed'
)
SELECT
    u.name,
    COUNT(o.id) AS recent_orders
FROM active_premium_users u
LEFT JOIN orders o ON o.user_id = u.id
JOIN completed_status cs ON o.status = cs.id
WHERE o.created_at > NOW() - INTERVAL '30 days'
GROUP BY u.id, u.name;

-- GOOD: Simple JOIN-based query
SELECT
    u.name,
    COUNT(o.id) AS order_count
FROM users u
INNER JOIN orders o ON o.user_id = u.id
WHERE o.status = 'completed'
  AND o.created_at > NOW() - INTERVAL '30 days'
GROUP BY u.id, u.name;
```

**Guidelines:**
- Use CTEs (WITH clauses) to break complex queries into readable steps
- Maximum 3 levels of CTE nesting
- Avoid subqueries in SELECT, WHERE, or FROM when possible
- Never use cursors — use set-based operations instead
- Prefer JOINs over correlated subqueries
- Keep each CTE under 15 lines

---

### Rule 2: Fixed Result Bounds — Always Use LIMIT

**Original Intent:** Ensure all loops terminate with provable upper bounds.

**PostgreSQL Adaptation:**

```sql
-- BAD: Unbounded result set
SELECT * FROM orders;

-- BAD: Unbounded aggregation
SELECT user_id, array_agg(order_id)
FROM orders
GROUP BY user_id;  -- Could create huge arrays

-- BAD: No pagination limit
SELECT * FROM users WHERE status = 'active' OFFSET 1000;

-- GOOD: Explicit LIMIT on all queries
SELECT id, name, email
FROM users
WHERE status = 'active'
ORDER BY created_at DESC
LIMIT 100;

-- GOOD: Paginated queries with maximum page size
SELECT id, name, email
FROM users
WHERE status = 'active'
ORDER BY id
LIMIT 20 OFFSET 0;  -- Application enforces LIMIT <= MAX_PAGE_SIZE (100)

-- GOOD: Bounded aggregation with window function
SELECT
    user_id,
    array_agg(order_id ORDER BY created_at DESC) AS recent_orders
FROM (
    SELECT
        user_id,
        order_id,
        created_at,
        ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY created_at DESC) AS rn
    FROM orders
) ranked
WHERE rn <= 100
GROUP BY user_id;

-- GOOD: Enforce limits in functions
CREATE OR REPLACE FUNCTION get_user_orders(
    p_user_id UUID,
    p_limit INTEGER DEFAULT 20
) RETURNS TABLE (order_id UUID, created_at TIMESTAMPTZ) AS $$
BEGIN
    IF p_limit > 100 THEN
        RAISE EXCEPTION 'Limit too high: % (max: 100)', p_limit;
    END IF;

    RETURN QUERY
    SELECT o.id, o.created_at
    FROM orders o
    WHERE o.user_id = p_user_id
    ORDER BY o.created_at DESC
    LIMIT p_limit;
END;
$$ LANGUAGE plpgsql STABLE;

-- GOOD: Keyset pagination (cursor-based, efficient for large tables)
SELECT id, name, email, created_at
FROM users
WHERE created_at > '2024-01-01 00:00:00+00'  -- Last cursor value
  AND status = 'active'
ORDER BY created_at, id
LIMIT 50;
```

**Guidelines:**
- Every SELECT statement must have a LIMIT clause
- Define MAX_PAGE_SIZE constant (typically 100-1000)
- Use keyset (cursor) pagination over OFFSET for large datasets
- Enforce limit bounds in stored functions
- Use ROW_NUMBER() with bounds for window aggregations
- Never allow user input to set LIMIT without validation

---

### Rule 3: Explicit Schema Definition — No Dynamic Types

**Original Intent:** Prevent unbounded memory allocation and type safety issues.

**PostgreSQL Adaptation:**

```sql
-- BAD: SELECT * (unknown columns, schema drift risk)
SELECT * FROM users;

-- BAD: TEXT/VARCHAR without limits
CREATE TABLE users (
    id UUID PRIMARY KEY,
    name TEXT,      -- Unbounded!
    email TEXT,
    bio TEXT
);

-- BAD: Nullable columns by default
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    user_id UUID,     -- NULL allowed implicitly
    total NUMERIC,
    status VARCHAR(50)
);

-- BAD: JSON for structured data
CREATE TABLE products (
    id UUID PRIMARY KEY,
    data JSONB  -- Untyped, schema-less
);

-- GOOD: Explicit column lists
SELECT id, name, email, created_at
FROM users;

INSERT INTO users (id, name, email, created_at)
VALUES (gen_random_uuid(), 'Alice', 'alice@example.com', NOW());

-- GOOD: Constrained types with NOT NULL by default
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL,
    bio VARCHAR(1000),  -- Explicit limit, nullable has meaning (optional)
    age INTEGER CHECK (age >= 0 AND age <= 150),
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- GOOD: Proper relational modeling instead of JSON
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(200) NOT NULL,
    price NUMERIC(10, 2) NOT NULL CHECK (price >= 0),
    category_id UUID NOT NULL REFERENCES categories(id),
    stock_quantity INTEGER NOT NULL DEFAULT 0 CHECK (stock_quantity >= 0),
    is_active BOOLEAN NOT NULL DEFAULT true
);

-- ACCEPTABLE: JSON only for truly dynamic attributes
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(200) NOT NULL,
    price NUMERIC(10, 2) NOT NULL,
    -- Core fields are typed columns
    attributes JSONB,  -- Only for optional, dynamic metadata
    CONSTRAINT chk_attributes_size CHECK (
        pg_column_size(attributes) < 65536  -- 64KB limit
    )
);

-- GOOD: Domain types for reusable constraints
CREATE DOMAIN email_address AS VARCHAR(255)
    CHECK (VALUE ~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');

CREATE DOMAIN positive_amount AS NUMERIC(10, 2)
    CHECK (VALUE >= 0);

CREATE TABLE accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email email_address NOT NULL UNIQUE,
    balance positive_amount NOT NULL DEFAULT 0
);
```

**Guidelines:**
- Always use explicit column lists (never SELECT * or INSERT without columns)
- All columns should be NOT NULL unless NULL has explicit meaning
- Use VARCHAR(N) with explicit limits, not unbounded TEXT
- Define CHECK constraints for value ranges
- Use proper types: NUMERIC for money, TIMESTAMPTZ for timestamps, UUID for IDs
- Use DOMAIN types for reusable validation logic
- Avoid JSONB for structured data — use proper columns
- If using JSON, enforce size limits with CHECK constraints

---

### Rule 4: Short Queries and Migrations — One Concern Per File

**Original Intent:** No function longer than 60 lines.

**PostgreSQL Adaptation:**

```sql
-- BAD: Monolithic migration (multiple concerns)
-- migrations/0001_everything.sql
CREATE TABLE users (...);
CREATE TABLE orders (...);
CREATE TABLE products (...);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_user_id ON orders(user_id);
ALTER TABLE orders ADD CONSTRAINT fk_orders_user
    FOREIGN KEY (user_id) REFERENCES users(id);
-- ... 500 more lines

-- BAD: Complex query (too many responsibilities)
SELECT
    u.id, u.name, u.email,
    COUNT(DISTINCT o.id) AS order_count,
    SUM(o.total) AS total_spent,
    AVG(o.total) AS avg_order_value,
    MAX(o.created_at) AS last_order_date,
    COUNT(DISTINCT p.id) AS unique_products
    -- ... 20 more aggregations
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
LEFT JOIN order_items oi ON oi.order_id = o.id
LEFT JOIN products p ON p.id = oi.product_id
WHERE u.status = 'active'
  AND o.created_at > NOW() - INTERVAL '1 year'
GROUP BY u.id, u.name, u.email;

-- GOOD: One migration per table/concern
-- migrations/0001_create_users_table.sql
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- migrations/0002_create_orders_table.sql
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    total NUMERIC(10, 2) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- migrations/0003_add_foreign_keys.sql
ALTER TABLE orders
ADD CONSTRAINT fk_orders_user
FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE;

-- migrations/0004_add_indexes.sql
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_created_at ON orders(created_at);

-- GOOD: Break complex query into a view
CREATE VIEW user_order_summary AS
SELECT
    u.id AS user_id,
    u.name,
    u.email,
    COUNT(o.id) AS order_count,
    COALESCE(SUM(o.total), 0) AS total_spent
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.status = 'active'
GROUP BY u.id, u.name, u.email;

-- Then query the view
SELECT * FROM user_order_summary
WHERE total_spent > 1000
LIMIT 100;

-- GOOD: Extract into stored function (≤60 lines)
CREATE OR REPLACE FUNCTION get_user_metrics(p_user_id UUID)
RETURNS TABLE (
    order_count BIGINT,
    total_spent NUMERIC,
    avg_order_value NUMERIC,
    last_order_date TIMESTAMPTZ
) AS $$
BEGIN
    RETURN QUERY
    SELECT
        COUNT(o.id)::BIGINT,
        COALESCE(SUM(o.total), 0),
        COALESCE(AVG(o.total), 0),
        MAX(o.created_at)
    FROM orders o
    WHERE o.user_id = p_user_id;
END;
$$ LANGUAGE plpgsql STABLE;
```

**Guidelines:**
- Keep individual queries under 60 lines
- One migration file per logical change (table, index, constraint)
- Break complex queries into views or materialized views
- Extract repeated logic into functions (max 60 lines each)
- Name migrations with timestamp: `YYYYMMDDHHMMSS_description.sql`
- Use migration tools (Flyway, Liquibase, migrate) for ordering
- Each migration must be reversible (provide DOWN migration)

---

### Rule 5: Constraint Density — CHECK, NOT NULL, Foreign Keys

**Original Intent:** Minimum 2 assertions per function to verify invariants.

**PostgreSQL Adaptation:**

```sql
-- BAD: No constraints (trusting application layer)
CREATE TABLE accounts (
    id UUID PRIMARY KEY,
    user_id UUID,
    balance NUMERIC,
    status VARCHAR(50),
    created_at TIMESTAMPTZ
);

-- BAD: Minimal constraints
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    user_id UUID NOT NULL,
    total NUMERIC NOT NULL
);

-- GOOD: Comprehensive constraints (minimum 2 per table beyond PK)
CREATE TABLE accounts (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    balance NUMERIC(12, 2) NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- CHECK constraints (assertions)
    CONSTRAINT chk_balance_non_negative CHECK (balance >= 0),
    CONSTRAINT chk_status_valid CHECK (status IN ('active', 'suspended', 'closed')),
    CONSTRAINT chk_timestamps_valid CHECK (updated_at >= created_at)
);

-- GOOD: Table-level and column-level constraints
CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    subtotal NUMERIC(10, 2) NOT NULL CHECK (subtotal >= 0),
    tax NUMERIC(10, 2) NOT NULL CHECK (tax >= 0),
    total NUMERIC(10, 2) NOT NULL CHECK (total >= 0),
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    shipped_at TIMESTAMPTZ,

    -- Multi-column constraints
    CONSTRAINT chk_total_equals_subtotal_plus_tax
        CHECK (total = subtotal + tax),
    CONSTRAINT chk_status_valid
        CHECK (status IN ('pending', 'paid', 'shipped', 'delivered', 'cancelled')),
    CONSTRAINT chk_shipped_after_created
        CHECK (shipped_at IS NULL OR shipped_at >= created_at)
);

-- GOOD: Use triggers for complex invariants
CREATE OR REPLACE FUNCTION check_account_balance()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.balance < 0 THEN
        RAISE EXCEPTION 'Account balance cannot be negative: %', NEW.balance;
    END IF;

    IF NEW.balance > 1000000 THEN
        RAISE WARNING 'Large balance detected: % for account %', NEW.balance, NEW.id;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_check_account_balance
    BEFORE INSERT OR UPDATE ON accounts
    FOR EACH ROW
    EXECUTE FUNCTION check_account_balance();

-- GOOD: Use exclusion constraints for overlap prevention
CREATE TABLE reservations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    room_id UUID NOT NULL REFERENCES rooms(id),
    start_time TIMESTAMPTZ NOT NULL,
    end_time TIMESTAMPTZ NOT NULL,

    CONSTRAINT chk_end_after_start CHECK (end_time > start_time),
    -- Prevent overlapping reservations for same room
    EXCLUDE USING GIST (
        room_id WITH =,
        tstzrange(start_time, end_time) WITH &&
    )
);
```

**Guidelines:**
- Minimum 2 constraints per table (beyond PRIMARY KEY)
- All columns should be NOT NULL unless NULL has specific meaning
- Use CHECK constraints for value ranges and enums
- Define foreign keys with explicit ON DELETE behavior
- Use multi-column CHECK constraints for cross-field invariants
- Use triggers for constraints that can't be expressed declaratively
- Use EXCLUSION constraints for uniqueness/overlap rules
- Document what each constraint protects against

---

### Rule 6: Least Privilege — Scoped Roles and Row-Level Security

**Original Intent:** Declare data objects at smallest possible scope.

**PostgreSQL Adaptation:**

```sql
-- BAD: Application uses superuser or owner role
-- Connection string: postgres://postgres:password@localhost/mydb

-- BAD: Single role for all operations
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO app_user;

-- BAD: No row-level security (all users see all data)

-- GOOD: Create specific roles with minimal privileges
-- Read-only role
CREATE ROLE app_readonly;
GRANT CONNECT ON DATABASE mydb TO app_readonly;
GRANT USAGE ON SCHEMA public TO app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_readonly;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT ON TABLES TO app_readonly;

-- Read-write role (no DDL)
CREATE ROLE app_readwrite;
GRANT CONNECT ON DATABASE mydb TO app_readwrite;
GRANT USAGE ON SCHEMA public TO app_readwrite;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_readwrite;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public TO app_readwrite;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO app_readwrite;

-- Migration role (DDL only)
CREATE ROLE app_migrator;
GRANT CONNECT ON DATABASE mydb TO app_migrator;
GRANT USAGE, CREATE ON SCHEMA public TO app_migrator;

-- GOOD: Schema isolation
CREATE SCHEMA app;
CREATE SCHEMA audit;
CREATE SCHEMA admin;

-- App role can only access app schema
GRANT USAGE ON SCHEMA app TO app_readwrite;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA app TO app_readwrite;

-- Read-only access to audit
GRANT USAGE ON SCHEMA audit TO app_readonly;
GRANT SELECT ON ALL TABLES IN SCHEMA audit TO app_readonly;

-- GOOD: Row-Level Security (RLS)
CREATE TABLE documents (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    title VARCHAR(200) NOT NULL,
    content TEXT NOT NULL,
    is_public BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Enable RLS
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- Policy: Users can only see their own documents or public ones
CREATE POLICY documents_select_policy ON documents
    FOR SELECT
    USING (
        user_id = current_setting('app.current_user_id')::UUID
        OR is_public = true
    );

-- Policy: Users can only modify their own documents
CREATE POLICY documents_modify_policy ON documents
    FOR UPDATE
    USING (user_id = current_setting('app.current_user_id')::UUID)
    WITH CHECK (user_id = current_setting('app.current_user_id')::UUID);

-- GOOD: Restrict function execution
CREATE OR REPLACE FUNCTION delete_old_logs()
RETURNS VOID AS $$
BEGIN
    DELETE FROM audit.logs WHERE created_at < NOW() - INTERVAL '90 days';
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

REVOKE EXECUTE ON FUNCTION delete_old_logs() FROM PUBLIC;
GRANT EXECUTE ON FUNCTION delete_old_logs() TO admin_role;
```

**Guidelines:**
- Never use superuser or database owner for application connections
- Create separate roles: readonly, readwrite, migrator, admin
- Use schema isolation for different concerns (app, audit, admin)
- Enable Row-Level Security (RLS) for multi-tenant data
- Use SECURITY DEFINER functions sparingly, with explicit privilege grants
- Set session variables for user context in RLS policies
- Regularly audit role permissions with pg_roles

---

### Rule 7: Handle NULL and Check Row Counts

**Original Intent:** Check all return values and validate parameters.

**PostgreSQL Adaptation:**

```sql
-- BAD: Assuming non-NULL values
SELECT name, length(email) FROM users;  -- Fails if email is NULL

-- BAD: Not checking affected rows
UPDATE accounts SET balance = balance - 100 WHERE id = '...';
-- Did it update anything? Was the account found?

-- BAD: Unchecked division by zero
SELECT total / COUNT(*) FROM orders;  -- Fails if no orders

-- BAD: Not validating function parameters
CREATE FUNCTION transfer_funds(from_account UUID, to_account UUID, amount NUMERIC)
RETURNS VOID AS $$
BEGIN
    UPDATE accounts SET balance = balance - amount WHERE id = from_account;
    UPDATE accounts SET balance = balance + amount WHERE id = to_account;
END;
$$ LANGUAGE plpgsql;

-- GOOD: Handle NULL explicitly
SELECT
    name,
    COALESCE(email, 'no-email@example.com') AS email,
    COALESCE(LENGTH(email), 0) AS email_length
FROM users;

-- Use NULLIF to avoid division by zero
SELECT
    SUM(total) / NULLIF(COUNT(*), 0) AS avg_order
FROM orders;

-- GOOD: Check affected row counts
CREATE OR REPLACE FUNCTION withdraw_funds(
    p_account_id UUID,
    p_amount NUMERIC
) RETURNS VOID AS $$
DECLARE
    v_rows_affected INTEGER;
BEGIN
    -- Validate inputs
    IF p_amount <= 0 THEN
        RAISE EXCEPTION 'Amount must be positive: %', p_amount;
    END IF;

    IF p_account_id IS NULL THEN
        RAISE EXCEPTION 'Account ID cannot be NULL';
    END IF;

    -- Perform update
    UPDATE accounts
    SET balance = balance - p_amount
    WHERE id = p_account_id
      AND balance >= p_amount;

    -- Check affected rows
    GET DIAGNOSTICS v_rows_affected = ROW_COUNT;

    IF v_rows_affected = 0 THEN
        RAISE EXCEPTION 'Account not found or insufficient balance: %', p_account_id;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- GOOD: Validate all parameters with preconditions
CREATE OR REPLACE FUNCTION transfer_funds(
    p_from_account UUID,
    p_to_account UUID,
    p_amount NUMERIC
) RETURNS VOID AS $$
DECLARE
    v_from_balance NUMERIC;
BEGIN
    -- Preconditions
    IF p_from_account IS NULL OR p_to_account IS NULL THEN
        RAISE EXCEPTION 'Account IDs cannot be NULL';
    END IF;

    IF p_from_account = p_to_account THEN
        RAISE EXCEPTION 'Cannot transfer to same account';
    END IF;

    IF p_amount <= 0 THEN
        RAISE EXCEPTION 'Amount must be positive: %', p_amount;
    END IF;

    -- Get balance with row locking
    SELECT balance INTO STRICT v_from_balance
    FROM accounts
    WHERE id = p_from_account
    FOR UPDATE;

    -- Check sufficient funds
    IF v_from_balance < p_amount THEN
        RAISE EXCEPTION 'Insufficient funds: % < %', v_from_balance, p_amount;
    END IF;

    -- Perform transfer
    UPDATE accounts SET balance = balance - p_amount WHERE id = p_from_account;
    UPDATE accounts SET balance = balance + p_amount WHERE id = p_to_account;

EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RAISE EXCEPTION 'Account not found';
END;
$$ LANGUAGE plpgsql;

-- GOOD: Use STRICT for required return values
CREATE FUNCTION get_user_email(p_user_id UUID)
RETURNS VARCHAR AS $$
DECLARE
    v_email VARCHAR;
BEGIN
    SELECT email INTO STRICT v_email
    FROM users
    WHERE id = p_user_id;

    RETURN v_email;
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RAISE EXCEPTION 'User not found: %', p_user_id;
    WHEN TOO_MANY_ROWS THEN
        RAISE EXCEPTION 'Multiple users found: %', p_user_id;
END;
$$ LANGUAGE plpgsql STABLE;
```

**Guidelines:**
- Use COALESCE, NULLIF, and CASE to handle NULL values explicitly
- Use STRICT in SELECT INTO to require exactly one row
- Check ROW_COUNT with GET DIAGNOSTICS after INSERT/UPDATE/DELETE
- Validate all function parameters at entry
- Use FOR UPDATE when reading data you intend to modify
- Assert pre-conditions and post-conditions in functions
- Handle exceptions (NO_DATA_FOUND, TOO_MANY_ROWS)
- Never ignore NULL — it's not zero, empty string, or false

---

### Rule 8: No Dynamic SQL — Use Parameterized Queries

**Original Intent:** Limit preprocessor use that obscures code analysis.

**PostgreSQL Adaptation:**

```sql
-- BAD: Dynamic SQL with string concatenation (SQL injection!)
CREATE OR REPLACE FUNCTION search_users(p_name TEXT)
RETURNS TABLE (id UUID, name VARCHAR, email VARCHAR) AS $$
BEGIN
    RETURN QUERY EXECUTE
        'SELECT id, name, email FROM users WHERE name = ''' || p_name || '''';
END;
$$ LANGUAGE plpgsql;
-- Vulnerable to: search_users("'; DROP TABLE users; --")

-- BAD: Building queries with EXECUTE
CREATE OR REPLACE FUNCTION dynamic_query(p_table TEXT, p_column TEXT)
RETURNS SETOF RECORD AS $$
BEGIN
    RETURN QUERY EXECUTE
        'SELECT * FROM ' || p_table || ' WHERE ' || p_column || ' IS NOT NULL';
END;
$$ LANGUAGE plpgsql;

-- BAD: Complex generated SQL
CREATE OR REPLACE FUNCTION build_report(p_filters JSONB)
RETURNS SETOF RECORD AS $$
DECLARE
    v_sql TEXT;
BEGIN
    v_sql := 'SELECT ...';
    -- 100 lines of string concatenation building SQL
    RETURN QUERY EXECUTE v_sql;
END;
$$ LANGUAGE plpgsql;

-- GOOD: Parameterized queries (safe from SQL injection)
CREATE OR REPLACE FUNCTION search_users(p_name TEXT)
RETURNS TABLE (id UUID, name VARCHAR, email VARCHAR) AS $$
BEGIN
    RETURN QUERY
    SELECT u.id, u.name, u.email
    FROM users u
    WHERE u.name = p_name;  -- Direct parameter reference
END;
$$ LANGUAGE plpgsql STABLE;

-- GOOD: Use format() if EXECUTE is truly necessary
CREATE OR REPLACE FUNCTION get_table_count(p_table_name TEXT)
RETURNS BIGINT AS $$
DECLARE
    v_count BIGINT;
BEGIN
    -- Whitelist table names
    IF p_table_name NOT IN ('users', 'orders', 'products') THEN
        RAISE EXCEPTION 'Invalid table name: %', p_table_name;
    END IF;

    -- Use format() with %I for identifiers, %L for literals
    EXECUTE format('SELECT COUNT(*) FROM %I', p_table_name) INTO v_count;
    RETURN v_count;
END;
$$ LANGUAGE plpgsql STABLE;

-- GOOD: Static SQL with conditional logic (replaces dynamic queries)
CREATE OR REPLACE FUNCTION search_users(
    p_name TEXT DEFAULT NULL,
    p_email TEXT DEFAULT NULL,
    p_status TEXT DEFAULT NULL
)
RETURNS TABLE (id UUID, name VARCHAR, email VARCHAR) AS $$
BEGIN
    RETURN QUERY
    SELECT u.id, u.name, u.email
    FROM users u
    WHERE (p_name IS NULL OR u.name ILIKE '%' || p_name || '%')
      AND (p_email IS NULL OR u.email = p_email)
      AND (p_status IS NULL OR u.status = p_status)
    ORDER BY u.created_at DESC
    LIMIT 100;
END;
$$ LANGUAGE plpgsql STABLE;
```

**Application Layer (always use parameterized queries):**
```python
# BAD: String formatting (SQL injection!)
cursor.execute(f"SELECT * FROM users WHERE email = '{email}'")

# GOOD: Parameterized query
cursor.execute("SELECT * FROM users WHERE email = %s", (email,))
```

```typescript
// BAD: Template literal (SQL injection!)
await sql`SELECT * FROM users WHERE email = '${email}'`;

// GOOD: Parameterized query
await sql`SELECT * FROM users WHERE email = ${email}`;
```

**Guidelines:**
- Never concatenate strings to build SQL queries
- Use parameterized queries in application code
- In PL/pgSQL, avoid EXECUTE when possible
- If EXECUTE is necessary, use format() with %I (identifiers) and %L (literals)
- Whitelist and validate any dynamic identifiers (table/column names)
- Prefer static SQL with conditional logic over dynamic SQL
- Use prepared statements in application layer

---

### Rule 9: Type-Safe Schemas — Proper Relations Over JSON

**Original Intent:** (C: Restrict pointer usage to single dereference)

**PostgreSQL Adaptation:**

```sql
-- BAD: Using JSONB for structured, relational data
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    data JSONB  -- Everything in JSON!
);

INSERT INTO orders (id, data) VALUES (
    gen_random_uuid(),
    '{"user_id": "123", "items": [{"product_id": "456", "qty": 2}], "total": 100}'
);
-- Can't enforce referential integrity, no type safety

-- BAD: Overly generic schema
CREATE TABLE entities (
    id UUID PRIMARY KEY,
    entity_type VARCHAR(50),
    entity_data JSONB
);

-- BAD: Multi-level JSON dereferencing
SELECT
    data->'user'->'address'->'city' AS city,
    data->'user'->'address'->'country'->'code' AS country_code
FROM orders;

-- GOOD: Proper relational modeling
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE
);

CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(200) NOT NULL,
    price NUMERIC(10, 2) NOT NULL CHECK (price >= 0)
);

CREATE TABLE orders (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE RESTRICT,
    total NUMERIC(10, 2) NOT NULL CHECK (total >= 0),
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE order_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id UUID NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price NUMERIC(10, 2) NOT NULL CHECK (unit_price >= 0)
);

-- Type-safe and optimizable queries
SELECT
    o.id,
    u.name AS user_name,
    p.name AS product_name,
    oi.quantity,
    oi.unit_price
FROM orders o
INNER JOIN users u ON u.id = o.user_id
INNER JOIN order_items oi ON oi.order_id = o.id
INNER JOIN products p ON p.id = oi.product_id
WHERE o.created_at > NOW() - INTERVAL '30 days'
LIMIT 100;

-- ACCEPTABLE: JSON only for truly dynamic attributes (single-level access)
CREATE TABLE products (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(200) NOT NULL,
    price NUMERIC(10, 2) NOT NULL,
    category_id UUID NOT NULL REFERENCES categories(id),

    -- Only for optional, variable attributes
    specifications JSONB,

    CONSTRAINT chk_specs_size CHECK (
        specifications IS NULL OR
        pg_column_size(specifications) < 65536
    )
);

-- Single-level access only
SELECT
    name,
    specifications->>'color' AS color,
    specifications->>'weight' AS weight
FROM products
WHERE specifications->>'category' = 'electronics'
LIMIT 100;

-- GOOD: Proper types for addresses (not nested JSON)
CREATE TABLE addresses (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    address_type VARCHAR(20) NOT NULL CHECK (address_type IN ('billing', 'shipping')),
    street VARCHAR(200) NOT NULL,
    city VARCHAR(100) NOT NULL,
    state VARCHAR(50),
    postal_code VARCHAR(20) NOT NULL,
    country_code CHAR(2) NOT NULL
);
```

**Guidelines:**
- Use proper relational modeling with typed columns
- Avoid JSONB for structured data that should be relational
- Limit JSON dereferencing to single level (`->>'field'`)
- If using JSON, enforce size constraints with CHECK
- Foreign keys ensure referential integrity (impossible with JSON)
- Proper types enable query optimization and indexing
- Only use JSONB for truly variable/optional metadata

---

### Rule 10: Static Analysis — EXPLAIN, pg_stat_statements, Migration Review

**Original Intent:** Compile with all warnings; use static analyzers daily.

**PostgreSQL Adaptation:**

```sql
-- GOOD: Analyze all queries before deployment
EXPLAIN ANALYZE
SELECT u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.created_at > NOW() - INTERVAL '30 days'
GROUP BY u.id, u.name
LIMIT 100;

-- Check for:
--   Sequential scans on large tables (should use indexes)
--   High cost estimates
--   Actual time vs estimated time discrepancies

-- GOOD: Check index usage with BUFFERS
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders
WHERE user_id = '123e4567-e89b-12d3-a456-426614174000'
LIMIT 10;
-- Should show "Index Scan" not "Seq Scan"

-- GOOD: Enable pg_stat_statements for query analytics
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Find slow queries
SELECT
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    max_exec_time
FROM pg_stat_statements
WHERE mean_exec_time > 100  -- > 100ms average
ORDER BY total_exec_time DESC
LIMIT 20;

-- Find tables missing indexes (high sequential scan count)
SELECT
    schemaname,
    tablename,
    seq_scan,
    seq_tup_read,
    idx_scan,
    seq_tup_read / NULLIF(seq_scan, 0) AS avg_tup_per_scan
FROM pg_stat_user_tables
WHERE seq_scan > 100
  AND seq_tup_read / NULLIF(seq_scan, 0) > 1000
ORDER BY seq_tup_read DESC
LIMIT 20;

-- Find unused indexes (waste space, slow writes)
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY pg_relation_size(indexrelid) DESC;
```

**postgresql.conf — Production Logging:**
```ini
log_min_duration_statement = 100   # Log queries > 100ms
log_checkpoints = on
log_connections = on
log_disconnections = on
log_lock_waits = on
```

**Migration Review Checklist:**
```
Every CREATE TABLE has:
  - PRIMARY KEY
  - NOT NULL on required columns
  - CHECK constraints (minimum 2)
  - Timestamps (created_at, updated_at)

Every foreign key has:
  - Explicit ON DELETE behavior
  - Index on foreign key column

Every VARCHAR has explicit length limit

Migrations are reversible (DOWN script exists)

Large data migrations use batching:
  UPDATE orders SET status = 'completed'
  WHERE id IN (
      SELECT id FROM orders WHERE status = 'old_status' LIMIT 10000
  );

Indexes created CONCURRENTLY (no table lock):
  CREATE INDEX CONCURRENTLY idx_name ON table(column);
```

**CI Pipeline:**
```bash
# Automated checks before deployment
sqlfluff lint migrations/*.sql                    # SQL linting
pg_dump --schema-only test_db > /dev/null         # Verify schema
grep -r "DROP TABLE" migrations/ && exit 1        # Block destructive ops
grep -r "SELECT \*" src/ && echo "WARN: SELECT *" # Warn on SELECT *
```

**Guidelines:**
- Run EXPLAIN ANALYZE on every query before production
- Enable pg_stat_statements in production
- Set log_min_duration_statement to catch slow queries
- Review pg_stat_user_tables regularly for sequential scans
- Identify and remove unused indexes
- Enforce migration checklist in code review
- Use sqlfluff or similar linter for SQL style
- Create indexes CONCURRENTLY to avoid locks
- Test migrations on production-sized datasets
- Batch large data migrations (10,000 rows at a time)

---

## Summary: PostgreSQL Adaptation

| # | Original Rule | PostgreSQL Guideline |
|---|---------------|----------------------|
| 1 | No goto/recursion | Simple CTEs, no cursors, max 3 CTE nesting levels |
| 2 | Fixed loop bounds | LIMIT on all queries, bounded pagination |
| 3 | No dynamic allocation | Explicit columns (no SELECT *), NOT NULL by default, constrained types |
| 4 | 60-line functions | Short queries, one migration per concern, extract views/functions |
| 5 | 2+ assertions/function | CHECK constraints, NOT NULL, foreign keys, triggers |
| 6 | Minimize scope | Least privilege roles, RLS, schema isolation |
| 7 | Check returns | Handle NULL (COALESCE/NULLIF), check ROW_COUNT, STRICT |
| 8 | Limit preprocessor | No dynamic SQL, parameterized queries only |
| 9 | Restrict pointers | Type-safe schemas, proper relations over JSON |
| 10 | All warnings enabled | EXPLAIN all queries, pg_stat_statements, migration review |

---

## References

- [Original Power of 10 Paper](https://spinroot.com/gerard/pdf/P10.pdf) — Gerard Holzmann
- [PostgreSQL Documentation](https://www.postgresql.org/docs/current/)
- [PostgreSQL Row Level Security](https://www.postgresql.org/docs/current/ddl-rowsecurity.html)
- [PostgreSQL Performance Tips](https://wiki.postgresql.org/wiki/Performance_Optimization)
- [Use The Index, Luke](https://use-the-index-luke.com/) — SQL Indexing Guide
- [pg_stat_statements Documentation](https://www.postgresql.org/docs/current/pgstatstatements.html)
- [PostgreSQL Security Best Practices](https://www.postgresql.org/docs/current/ddl-priv.html)
- [SQL Style Guide](https://www.sqlstyle.guide/)
