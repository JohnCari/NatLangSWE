# Power of 10 Rules for Python

> NASA JPL's safety-critical coding guidelines adapted for Python

Based on Gerard J. Holzmann's original rules (2006) for the Mars rover and other mission-critical software.

---

## Rule 1: Simple Control Flow

**No recursion. Limit `break`/`continue`. Avoid deep nesting.**

```python
# BAD: Recursion
def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)

# GOOD: Iteration
def factorial(n: int) -> int:
    result = 1
    for i in range(2, n + 1):
        result *= i
    return result
```

---

## Rule 2: Bounded Loops

**All loops must have provable upper limits.**

```python
from itertools import islice

# BAD: Unbounded
for item in infinite_generator():
    process(item)

# GOOD: Bounded with islice
MAX_ITEMS = 10_000
for item in islice(infinite_generator(), MAX_ITEMS):
    process(item)

# GOOD: Bounded with assertion
def iter_bounded(iterable, max_iter: int):
    for count, item in enumerate(iterable):
        assert count < max_iter, f"Exceeded {max_iter} iterations"
        yield item
```

---

## Rule 3: Controlled Memory

**Use pagination and chunking. Avoid unbounded data structures.**

```python
# BAD: Load everything into memory
all_users = db.query("SELECT * FROM users")

# GOOD: Paginate
PAGE_SIZE = 100
offset = 0
while True:
    users = db.query(f"SELECT * FROM users LIMIT {PAGE_SIZE} OFFSET {offset}")
    if not users:
        break
    for user in users:
        process(user)
    offset += PAGE_SIZE
```

---

## Rule 4: Short Functions

**Maximum ~60 lines. Single responsibility.**

```python
# BAD: Does too much
def process_order(order):
    # validate (20 lines)
    # calculate totals (20 lines)
    # apply discounts (20 lines)
    # save to db (20 lines)
    # send email (20 lines)
    pass

# GOOD: Split into focused functions
def validate_order(order: Order) -> None: ...
def calculate_totals(order: Order) -> Decimal: ...
def apply_discounts(order: Order, total: Decimal) -> Decimal: ...
def save_order(order: Order) -> None: ...
def notify_customer(order: Order) -> None: ...
```

---

## Rule 5: Assertions

**Minimum 2 assertions per function. Check pre/post conditions. Fail fast.**

```python
def transfer_funds(from_account: Account, to_account: Account, amount: Decimal) -> None:
    # Preconditions
    assert amount > 0, "Amount must be positive"
    assert from_account.balance >= amount, "Insufficient funds"

    from_account.balance -= amount
    to_account.balance += amount

    # Postcondition
    assert from_account.balance >= 0, "Balance went negative"
```

---

## Rule 6: Minimal Scope

**No globals. Encapsulate in classes/modules. Prefix private with `_`.**

```python
# BAD: Global state
config = {}
def get_setting(key):
    return config[key]

# GOOD: Encapsulated
class Config:
    def __init__(self):
        self._settings: dict[str, Any] = {}

    def get(self, key: str) -> Any:
        return self._settings[key]

    def _load(self, path: str) -> None:
        # Private method
        ...
```

---

## Rule 7: Check Returns

**Handle all return values. Let exceptions bubble up to proper handlers.**

```python
# BAD: Ignoring return value
open("file.txt")

# BAD: Swallowing exceptions
try:
    risky_operation()
except Exception:
    pass

# GOOD: Handle or propagate
try:
    result = risky_operation()
    assert result is not None, "Operation returned None"
except SpecificError as e:
    logger.error(f"Operation failed: {e}")
    raise
```

---

## Rule 8: Limit Metaprogramming

**Avoid `exec`/`eval`. Limit decorators and metaclasses.**

```python
# BAD: Dynamic code execution
eval(user_input)
exec(f"result = {expression}")

# BAD: Excessive decorator stacking
@log
@cache
@retry
@validate
@authorize
def my_function(): ...

# GOOD: Simple, readable code
def my_function():
    if not authorized():
        raise PermissionError()
    return compute_result()
```

---

## Rule 9: Type Safety

**Use type hints. Prefer immutable types. Don't mutate arguments.**

```python
from typing import NamedTuple
from dataclasses import dataclass

# BAD: Mutable, untyped
def process(data):
    data["processed"] = True  # Mutates argument
    return data

# GOOD: Immutable, typed
class ProcessedData(NamedTuple):
    value: str
    processed: bool

def process(data: dict[str, Any]) -> ProcessedData:
    return ProcessedData(
        value=data["value"],
        processed=True
    )
```

---

## Rule 10: Static Analysis

**Use mypy, ruff/pylint. Zero warnings policy.**

```bash
# pyproject.toml
[tool.mypy]
strict = true
warn_return_any = true

[tool.ruff]
select = ["ALL"]

# Run before every commit
mypy src/
ruff check src/
```

```python
# All code must pass with zero warnings
# Type hints are mandatory
# No # type: ignore without justification
```

---

## Summary

| Rule | Python Guideline |
|------|------------------|
| 1 | No recursion, limit break/continue |
| 2 | Bounded loops with `islice` or assertions |
| 3 | Paginate data, avoid unbounded collections |
| 4 | Functions under 60 lines |
| 5 | Min 2 assertions per function |
| 6 | No globals, use `_` prefix for private |
| 7 | Handle returns, let exceptions bubble |
| 8 | No `exec`/`eval`, limit metaprogramming |
| 9 | Type hints, `NamedTuple`, don't mutate args |
| 10 | mypy strict, ruff, zero warnings |

---

## Sources

- [Original Paper (PDF)](https://spinroot.com/gerard/pdf/P10.pdf) — Gerard J. Holzmann, NASA JPL
- [10 Rules Applied to Interpreted Languages](https://dev.to/xowap/10-rules-to-code-like-nasa-applied-to-interpreted-languages-40dd)
- [Powerof10-NASA GitHub](https://github.com/Vhivi/Powerof10-NASA)
