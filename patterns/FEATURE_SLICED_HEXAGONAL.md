# Feature-Sliced Hexagonal Architecture

**Pattern for parallel AI agent swarm development in a monorepo.**

Combines Vertical Slice Architecture (feature isolation) + Hexagonal Architecture (ports & adapters).

---

## Structure

```
src/
├── common/                    # Shared types - READ ONLY after skeleton
│   ├── types/
│   ├── interfaces/
│   └── constants/
├── features/
│   ├── [feature]/             # One terminal/swarm per feature
│   │   ├── core/              # Business logic (no deps except common/)
│   │   ├── ports/             # Internal interfaces
│   │   ├── adapters/          # Implementations (API, storage)
│   │   ├── components/        # UI (TSX)
│   │   ├── hooks/             # React hooks
│   │   ├── pages/             # Routes/API endpoints
│   │   └── tests/
└── app/                       # Integration only (after features complete)
```

---

## Golden Rules

1. **Skeleton first** — Create folder structure + define `common/` types before implementation
2. **common/ is read-only** — After skeleton phase, no terminal modifies `common/`
3. **One terminal per feature** — Each terminal/swarm owns exactly one feature folder
4. **No cross-feature imports** — Features only import from `common/`, never from other features
5. **Agent ownership** — Within a feature, each agent owns specific subfolders

---

## Workflow

```
1. IDEATION     → You describe idea to Claude
2. ANALYSIS     → Claude checks: does skeleton exist?
                  - NO  → Claude creates skeleton FIRST (folders + common/ types)
                  - YES → Claude uses existing structure
3. BREAKDOWN    → Claude splits idea into features, assigns to terminals
4. PARALLEL     → Each terminal implements its feature (swarm inside)
5. INTEGRATION  → One terminal wires up app/ routes, merges branches
```

---

## File Ownership

| Owner | Can Modify |
|-------|------------|
| Terminal 1 | `features/auth/**` only |
| Terminal 2 | `features/products/**` only |
| Terminal N | `features/[assigned]/**` only |
| Integration | `app/**` only (after features done) |
| Nobody | `common/**` (read-only after skeleton) |

---

## Agent Roles Within Feature

| Folder | Agent | Purpose |
|--------|-------|---------|
| core/ | 1 | Pure business logic |
| ports/ | 2 | Internal interfaces |
| adapters/ | 3-4 | API calls, storage |
| components/ | 5 | React/TSX UI |
| hooks/ | 5 | React hooks |
| pages/ | 6 | Routes, API endpoints |
| tests/ | 7 | All tests |

---

## Anti-Patterns

- Starting implementation without skeleton
- Two agents/terminals modifying same file
- Defining shared types in feature folder (put in `common/`)
- Importing from another feature's folder
- Modifying `common/` during implementation
