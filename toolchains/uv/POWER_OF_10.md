# The Power of 10 Rules for Safety-Critical UV (Python Toolchain) Configuration

## Background

The **Power of 10 Rules** were created in 2006 by **Gerard J. Holzmann** at NASA's Jet Propulsion Laboratory (JPL) Laboratory for Reliable Software. These rules were designed for writing safety-critical code in C that could be effectively analyzed by static analysis tools.

The rules were incorporated into JPL's institutional coding standard and used for major missions including the **Mars Science Laboratory** (Curiosity Rover, 2012).

> *"If these rules seem draconian at first, bear in mind that they are meant to make it possible to check safety-critical code where human lives can very literally depend on its correctness."* — Gerard Holzmann

**Note:** UV by Astral replaces pip, pip-tools, pipx, poetry, pyenv, and virtualenv with a single, deterministic tool written in Rust. These rules treat `pyproject.toml` as the single source of truth and `uv.lock` as the integrity contract for every environment. For Python language-level rules, see the [Python Power of 10](../../languages/python/POWER_OF_10.md).

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

## The Power of 10 Rules — UV Edition

### Rule 1: Simple Control Flow — Declarative Dependencies, No Shell Script Chains

**Original Intent:** Eliminate complex control flow that impedes static analysis and can cause stack overflows.

**UV Adaptation:**

```bash
# BAD: Multi-tool shell script chain
pip install pip-tools
pip-compile requirements.in -o requirements.txt
pip-sync requirements.txt
pip install -e .
source .venv/bin/activate
python -c "import pkg_resources; pkg_resources.require(open('requirements.txt').readlines())"

# BAD: Mixing package managers
conda install numpy
pip install flask
poetry add sqlalchemy
pip freeze > requirements.txt  # Inconsistent state

# BAD: Nested Makefile targets for dependency management
install:
	python -m venv .venv
	. .venv/bin/activate && pip install --upgrade pip
	. .venv/bin/activate && pip install -r requirements.txt
	. .venv/bin/activate && pip install -r requirements-dev.txt
	. .venv/bin/activate && pip install -e .

# GOOD: One command to resolve and install
uv sync

# GOOD: Run anything through uv (no activation needed)
uv run pytest
uv run python main.py
uv run mypy src/

# GOOD: Simple Makefile with single-command targets
test:
	uv run pytest

lint:
	uv run ruff check src/

typecheck:
	uv run mypy src/
```

**Guidelines:**
- All dependency management must be declarative via `pyproject.toml` and UV commands
- No chaining multiple package managers (pip + conda + poetry)
- Use `uv run <command>` instead of `source .venv/bin/activate && command`
- Maximum one step to reproduce an environment: `uv sync`
- No recursive Makefile includes for dependency management

---

### Rule 2: Fixed Bounds — Pin Versions, Lock Everything

**Original Intent:** Ensure all loops terminate and can be analyzed statically.

**UV Adaptation:**

```toml
# BAD: No version constraints
[project]
dependencies = [
    "requests",           # No version — could resolve to anything
    "flask>=2.0",         # No upper bound — major version breaks possible
    "numpy>=1.0,<999",   # Absurdly wide bound — meaningless constraint
]
# uv.lock not committed to version control

# BAD: No Python version constraint
[project]
name = "myapp"
# requires-python is missing — any Python version could be used

# GOOD: Bounded version specifiers
[project]
name = "myapp"
requires-python = ">=3.12,<3.13"
dependencies = [
    "requests~=2.31",     # Compatible release: >=2.31, <3.0
    "flask>=3.0,<4",      # Explicit upper bound at next major
    "numpy>=1.26,<2",     # Pinned to 1.x series
    "pydantic>=2.6,<3",   # Pinned to 2.x series
]
# uv.lock committed and verified in CI

# GOOD: Pin Python version for local development
```

```bash
# GOOD: Pin Python version locally
uv python pin 3.12

# GOOD: Verify lockfile is in sync
uv lock --check
```

**Guidelines:**
- Always commit `uv.lock` to version control
- Use compatible release (`~=`) or explicit upper bounds (`>=X,<Y`)
- Never use `*` or unbounded `>=` without an upper bound
- Run `uv lock --check` in CI to verify the lockfile is up to date
- Pin Python version with `requires-python` in `pyproject.toml`
- Use `uv python pin` for explicit local version pinning

---

### Rule 3: No Dynamic Allocation After Init — Deterministic Environments, No Runtime Installs

**Original Intent:** Prevent unbounded memory growth and allocation failures.

**UV Adaptation:**

```python
# BAD: Runtime package installation
import subprocess

def setup_analytics():
    subprocess.run(["pip", "install", "pandas"])  # Installing at runtime!
    import pandas as pd
    return pd.DataFrame()

# BAD: Conditional imports with fallback installs
try:
    import redis
except ImportError:
    import subprocess
    subprocess.run(["pip", "install", "redis"])
    import redis
```

```dockerfile
# BAD: Dockerfile without lockfile
FROM python:3.12
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]

# GOOD: Deterministic Docker build with UV
FROM python:3.12-slim AS base
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-install-project --no-dev

COPY . .
RUN uv sync --frozen --no-dev

CMD ["uv", "run", "python", "-m", "myapp"]
```

```bash
# BAD: Installing without a lockfile
uv pip install flask  # Ad-hoc install, not tracked

# GOOD: Frozen install (fails if lockfile is stale)
uv sync --frozen

# GOOD: Dependencies-only install (for linting/CI stages)
uv sync --frozen --no-install-project
```

**Guidelines:**
- The environment is fully resolved at build time (`uv sync`) — never at runtime
- Use `uv sync --frozen` in CI and production (fails if lockfile is stale)
- Never call `pip install` or `uv pip install` from application code
- Docker: resolve once with `uv sync --frozen`, copy `.venv` to final stage
- Use `--no-install-project` when you only need dependencies (e.g., linting stage)

---

### Rule 4: Keep It Small — Lean pyproject.toml, Organized Dependency Groups

**Original Intent:** Ensure functions are small enough to understand, test, and verify.

**UV Adaptation:**

```toml
# BAD: Monolithic dependency list mixing all concerns
[project]
dependencies = [
    "fastapi", "uvicorn", "sqlalchemy", "alembic", "celery",
    "redis", "httpx", "pydantic", "python-jose", "passlib",
    "pytest", "pytest-asyncio", "pytest-cov", "factory-boy",
    "mypy", "ruff", "black", "isort", "pre-commit",
    "sphinx", "sphinx-rtd-theme", "myst-parser",
    # ... 20 more entries mixing production, test, and dev deps
]

# BAD: Commented-out or duplicate dependencies
[project]
dependencies = [
    "requests~=2.31",
    # "httpx~=0.27",  # Switched from httpx to requests
    "requests~=2.31",  # Duplicate!
]

# GOOD: Minimal core dependencies (under 15)
[project]
dependencies = [
    "fastapi>=0.110,<1",
    "sqlalchemy>=2.0,<3",
    "httpx>=0.27,<1",
    "pydantic>=2.6,<3",
    "alembic>=1.13,<2",
]

# GOOD: Separated dependency groups by concern
[dependency-groups]
dev = [
    "ruff>=0.4",
    "mypy>=1.10",
    "pre-commit>=3.7",
]
test = [
    "pytest>=8.0",
    "pytest-asyncio>=0.23",
    "coverage>=7.5",
]
docs = [
    "sphinx>=7.3",
    "sphinx-rtd-theme>=2.0",
]
```

```bash
# GOOD: Add to a specific group
uv add --group dev ruff
uv add --group test pytest

# GOOD: Install only what you need
uv sync --group dev --group test  # Dev + test, no docs
uv sync --no-dev                  # Production only
```

**Guidelines:**
- Keep `[project.dependencies]` under 15 entries — these are your production dependencies
- Use `[dependency-groups]` for dev/test/lint/docs separation
- One concern per dependency group
- No commented-out dependencies — remove them entirely
- Use `uv add --group <name>` instead of manual editing
- Install only needed groups in each environment

---

### Rule 5: Validate the Environment — Assert Correctness with `uv run`

**Original Intent:** Defensive programming catches bugs early; assertions document invariants.

**UV Adaptation:**

```bash
# BAD: Running directly (which Python? which venv?)
python script.py
pytest tests/
mypy src/

# BAD: No environment verification in CI
- name: Run tests
  run: pytest tests/  # Deps might not be installed, wrong Python version

# GOOD: Always execute through uv run
uv run python script.py
uv run pytest tests/
uv run mypy src/

# GOOD: Pin and verify Python version
uv python pin 3.12
uv run python --version  # Verify it's actually 3.12
```

```yaml
# GOOD: Full environment validation in CI (GitHub Actions)
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v3

      - name: Verify lockfile integrity
        run: uv lock --check

      - name: Install dependencies
        run: uv sync --frozen

      - name: Verify Python version
        run: uv run python -c "import sys; assert sys.version_info >= (3, 12), f'Expected 3.12+, got {sys.version}'"

      - name: Run tests
        run: uv run pytest tests/
```

**Guidelines:**
- Always use `uv run` to execute project commands — never bare `python` or `pytest`
- Verify `uv lock --check` passes in CI (lockfile integrity)
- Use `requires-python` in `pyproject.toml` to enforce Python version
- Use `uv python pin` for explicit local version pinning
- Add environment validation as the first CI step before running tests
- Assert Python version programmatically in CI

---

### Rule 6: Minimal Scope — Isolated Virtual Environments, No Global Installs

**Original Intent:** Reduce state complexity and potential for misuse.

**UV Adaptation:**

```bash
# BAD: System-level installs
sudo pip install flask
pip install --user requests

# BAD: Sharing a venv across projects
source ~/shared-venv/bin/activate
pip install flask  # Project A
pip install django  # Project B — now conflicts

# BAD: Global tool installs polluting the system
pip install ruff
pip install mypy
pip install pre-commit  # All in system Python

# GOOD: Per-project virtual environment (automatic with uv sync)
cd my-project/
uv sync  # Creates .venv/ scoped to this project

# GOOD: Isolated global CLI tools
uv tool install ruff        # Gets its own isolated environment
uv tool install pre-commit  # Each tool is sandboxed
uv tool install mypy        # No conflicts between tools

# GOOD: List and manage installed tools
uv tool list
uv tool upgrade ruff
```

```toml
# GOOD: UV workspaces for monorepos (shared lockfile, isolated packages)
# root pyproject.toml
[tool.uv.workspace]
members = ["packages/*"]

# packages/api/pyproject.toml
[project]
name = "myapp-api"
dependencies = ["fastapi>=0.110,<1"]

# packages/worker/pyproject.toml
[project]
name = "myapp-worker"
dependencies = ["celery>=5.3,<6"]
```

```gitignore
# GOOD: Always gitignore the virtual environment
.venv/
```

**Guidelines:**
- Never use `pip install` at the system or user level
- One `.venv` per project, managed by UV (created automatically by `uv sync`)
- Use `uv tool install` for global CLI tools (each gets its own isolated env)
- Use UV workspaces for monorepo projects with shared lockfile
- Add `.venv/` to `.gitignore`
- Never activate another project's virtual environment

---

### Rule 7: Check Return Values — Verify Locks, Hashes, and Resolution

**Original Intent:** Never ignore errors; verify inputs at trust boundaries.

**UV Adaptation:**

```bash
# BAD: Ignoring failures
uv lock || true                  # Swallowing resolution errors
uv sync 2>/dev/null              # Suppressing all output
uv run pytest; echo "done"       # Ignoring test exit code

# BAD: No integrity verification
uv sync                          # In CI without --frozen (may resolve differently)
uv pip install flask             # Not tracked in lockfile

# GOOD: Every step checked
set -euo pipefail  # In scripts: fail on any error

uv lock --check                  # Fails if lockfile is stale
uv sync --frozen                 # Fails if lock doesn't match pyproject.toml
uv run pytest                    # Exit code propagated

# GOOD: Hash verification for production
uv export --format requirements-txt --no-hashes > requirements.txt  # For compatibility
uv export --format requirements-txt > requirements.txt              # With hashes (default)
```

```yaml
# GOOD: CI pipeline with strict verification
- name: Check lockfile integrity
  run: uv lock --check

- name: Install with frozen lockfile
  run: uv sync --frozen

- name: Audit for vulnerabilities
  run: uv run pip-audit
```

**Guidelines:**
- Never ignore `uv lock` / `uv sync` exit codes
- Use `uv lock --check` in CI to detect lockfile drift
- Use `--frozen` in CI and production to fail on stale lockfiles
- Use `set -euo pipefail` in shell scripts
- Enable hash verification for production deployments
- Review `uv lock` output for yanked or deprecated packages

---

### Rule 8: Minimize Build Complexity — Simple Build Backend, No Custom Build Scripts

**Original Intent:** Avoid constructs that create unmaintainable, unanalyzable code.

**UV Adaptation:**

```python
# BAD: Complex setup.py with runtime logic
# setup.py
import subprocess
import os

version = subprocess.check_output(["git", "describe", "--tags"]).strip().decode()

if os.environ.get("USE_CYTHON"):
    ext_modules = [Extension("mylib.fast", ["src/mylib/fast.pyx"])]
else:
    ext_modules = []

setup(
    name="mypackage",
    version=version,
    install_requires=open("requirements.txt").readlines(),
    ext_modules=ext_modules,
)
```

```toml
# BAD: Dynamic metadata from external commands
[project]
dynamic = ["version", "dependencies"]  # Where do these come from?

[tool.setuptools.dynamic]
version = {attr = "mypackage._version.VERSION"}
dependencies = {file = "requirements.txt"}  # Defeats the purpose of pyproject.toml

# GOOD: Standard build backend with static config
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "mypackage"
version = "1.2.0"

# GOOD: Version from source file (declarative, not subprocess)
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.version]
path = "src/mypackage/__init__.py"  # Reads __version__ = "1.2.0"
```

```bash
# GOOD: Build with uv
uv build                         # Creates sdist and wheel
uv publish                       # Publish to PyPI
```

**Guidelines:**
- Use standard build backends: `hatchling`, `setuptools>=68`, `flit-core`, or `maturin` (for Rust)
- Keep all build config in `pyproject.toml` — no `setup.py`, no `setup.cfg`
- No dynamic version computation via shell commands or subprocess
- No conditional dependencies based on environment sniffing
- Use `uv build` for building packages
- If you need `setup.py`, keep it under 10 lines

---

### Rule 9: Typed Dependencies — No Wildcards, No Uncontrolled Sources

**Original Intent:** Restrict pointers: single dereference only, no function pointers.

**UV Adaptation:**

```toml
# BAD: Unversioned and unpinned dependencies
[project]
dependencies = [
    "requests",                                              # No version at all
    "mylib @ git+https://github.com/org/repo",              # Unpinned branch
    "sketchy-pkg @ https://random-server.example.com/pkg.whl",  # Untrusted source
]

# BAD: Multiple unrestricted index URLs
[[tool.uv.index]]
url = "https://random-mirror.example.com/simple"  # Who controls this?

[[tool.uv.index]]
url = "https://another-mirror.example.com/simple"  # Dependency confusion risk

# GOOD: Every dependency has a version constraint
[project]
dependencies = [
    "requests~=2.31",
    "flask>=3.0,<4",
    "pydantic>=2.6,<3",
]

# GOOD: Git dependencies pinned to specific tags or commits
[tool.uv.sources]
mylib = { git = "https://github.com/org/repo", tag = "v1.2.3" }
internal-utils = { git = "https://github.com/org/utils", rev = "a1b2c3d" }

# GOOD: Explicit index configuration
[[tool.uv.index]]
name = "PyPI"
url = "https://pypi.org/simple"

# GOOD: Private index for internal packages only
[[tool.uv.index]]
name = "internal"
url = "https://pypi.internal.company.com/simple"
explicit = true  # Only used for packages that explicitly reference it
```

**Guidelines:**
- Every dependency must have a version constraint
- Pin git dependencies to specific commits or tags — never branches
- Configure package indexes explicitly in `pyproject.toml`
- Use `[tool.uv.sources]` for non-PyPI dependencies
- Use `explicit = true` on private indexes to prevent dependency confusion
- Restrict to a single trusted package index when possible
- No `--no-deps` in production installs

---

### Rule 10: Static Analysis — Audit Dependencies, Automate CI

**Original Intent:** Catch issues at development time; use every available tool.

**UV Adaptation:**

```yaml
# GOOD: Complete CI pipeline (GitHub Actions)
name: UV Safety Checks
on: [push, pull_request]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install uv
        uses: astral-sh/setup-uv@v3

      - name: Verify lockfile
        run: uv lock --check

      - name: Install dependencies
        run: uv sync --frozen --group dev --group test

      - name: Lint
        run: uv run ruff check src/

      - name: Format check
        run: uv run ruff format --check src/

      - name: Type check
        run: uv run mypy src/

      - name: Tests
        run: uv run pytest --cov=src tests/

      - name: Security audit
        run: uv run pip-audit
```

```bash
# BAD: No CI verification
# "It works on my machine" — lockfile never checked, deps never audited

# BAD: Manual lockfile updates
uv lock  # Developer remembers to run this... sometimes

# GOOD: Pre-commit hook for lockfile consistency
# .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      - id: uv-lock-check
        name: Check uv lockfile
        entry: uv lock --check
        language: system
        pass_filenames: false
        files: pyproject.toml

# GOOD: Automated dependency updates
# Use Renovate or Dependabot to create PRs for dependency updates
# renovate.json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["config:recommended"],
  "packageRules": [
    {
      "matchManagers": ["uv"],
      "groupName": "python dependencies"
    }
  ]
}
```

**Guidelines:**
- CI must include: `uv lock --check`, `uv sync --frozen`, `uv run pytest`
- Run `pip-audit` against the lockfile for vulnerability scanning
- Use `uv run ruff check` and `uv run mypy` for code quality
- Automate dependency updates with Renovate or Dependabot
- Enforce all checks as required status checks on PRs
- Use pre-commit hooks for lockfile consistency
- Zero warnings policy — fix all warnings before merging

---

## Summary: UV Adaptation

| # | Original Rule | UV Guideline |
|---|---------------|--------------|
| 1 | No goto/recursion | Declarative deps via `pyproject.toml`, no shell script chains, `uv sync` only |
| 2 | Fixed loop bounds | Pin all versions with upper bounds, commit `uv.lock`, bounded specifiers |
| 3 | No dynamic allocation | `uv sync --frozen` at build time, no runtime installs, deterministic Docker |
| 4 | 60-line functions | Lean `pyproject.toml` (under 15 core deps), organized `[dependency-groups]` |
| 5 | 2+ assertions/function | `uv run` for all execution, validate env and Python version in CI |
| 6 | Minimize scope | Per-project `.venv`, `uv tool` for global CLIs, no system installs |
| 7 | Check returns | Verify lockfile integrity, `--frozen` in CI, never ignore exit codes |
| 8 | Limit preprocessor | Standard build backends, no custom `setup.py`, pure `pyproject.toml` config |
| 9 | Restrict pointers | No wildcards, pinned git deps, explicit trusted indexes |
| 10 | All warnings enabled | Full CI pipeline: `ruff` + `mypy` + `pytest` + `pip-audit`, automated updates |

---

## References

- [Original Power of 10 Paper](https://spinroot.com/gerard/pdf/P10.pdf) — Gerard Holzmann
- [UV Documentation](https://docs.astral.sh/uv/) — Astral
- [UV GitHub Repository](https://github.com/astral-sh/uv) — Astral
- [UV: An Extremely Fast Python Package Manager](https://astral.sh/blog/uv) — Astral Blog
- [pyproject.toml Specification](https://packaging.python.org/en/latest/specifications/pyproject-toml/) — Python Packaging Authority
- [Ruff Linter](https://docs.astral.sh/ruff/) — Astral
- [pip-audit](https://github.com/pypa/pip-audit) — Python Packaging Authority
- [Dependency Groups (PEP 735)](https://peps.python.org/pep-0735/) — Python Enhancement Proposal
