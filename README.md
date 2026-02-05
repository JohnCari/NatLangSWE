# NatLangDev

> A Natural Language Software Developer — Best practices for AI-assisted development to prevent vibe coding

## What is NatLangDev?

**NatLangDev** stands for **Natural Language Software Developer** — a discipline for writing software through structured communication with AI coding assistants.

Natural Language Software Development is the practice of directing AI assistants like Claude Code using explicit, well-defined guidelines rather than ad-hoc prompts. This repository teaches the best practices for this emerging development paradigm.

## Why NatLangDev?

### The Problem: Vibe Coding

"Vibe coding" happens when developers prompt AI assistants without clear constraints or standards, resulting in:

- Inconsistent code quality
- Security vulnerabilities
- Poor architectural decisions
- Code that "works" but violates best practices
- Technical debt that compounds with each generation

### The Solution: Structured Natural Language Development

Instead of vague prompts like *"write me a web server"*, Natural Language Software Development provides explicit rules that AI assistants follow. By adding these guidelines to your AI's context, you get code that adheres to proven standards — whether that's NASA's safety-critical rules, idiomatic language patterns, or framework-specific conventions.

## How to Use

1. **Choose your guidelines** — Select the language, framework, and pattern guidelines relevant to your project
2. **Add to AI context** — Include the guideline files in your AI assistant's context (e.g., via CLAUDE.md, system prompts, or project files)
3. **Develop with structure** — Your AI assistant will now follow these best practices in all generated code

### Example

To use Rust guidelines with Claude Code, add to your `CLAUDE.md`:

```markdown
Follow the guidelines in:
- /path/to/NatLangDev/languages/rust/POWER_OF_10.md
- /path/to/NatLangDev/patterns/MINIMAL_DEPENDENCIES.md
```

## Repository Structure

```
NatLangDev/
├── languages/
│   ├── rust/       # Rust guidelines
│   ├── python/     # Python guidelines
│   └── typescript/ # TypeScript guidelines
├── frameworks/
│   ├── nextjs/     # NextJS (TSX) guidelines
│   ├── react/      # React (TSX) guidelines
│   ├── axum/       # Rust Axum guidelines
│   ├── electron/   # Electron (TypeScript) guidelines
│   ├── fastapi/    # FastAPI (Python) guidelines
│   └── pytorch/    # PyTorch (Python) guidelines
├── patterns/       # Cross-language design patterns
├── workflows/      # Development workflows
├── stacks/         # Recommended tech stacks
└── vscode/         # VS Code configurations
```

---

## Guidelines

### Power of 10 Rules

NASA JPL's safety-critical coding rules adapted for modern development.

| Language | Guidelines |
|----------|------------|
| Rust | [POWER_OF_10.md](languages/rust/POWER_OF_10.md) |
| Python | [POWER_OF_10.md](languages/python/POWER_OF_10.md) |
| TypeScript | [POWER_OF_10.md](languages/typescript/POWER_OF_10.md) |

| Framework | Guidelines |
|-----------|------------|
| React | [POWER_OF_10.md](frameworks/react/POWER_OF_10.md) |
| NextJS | [POWER_OF_10.md](frameworks/nextjs/POWER_OF_10.md) ・ [PREFERENCES.md](frameworks/nextjs/PREFERENCES.md) |
| Electron | [POWER_OF_10.md](frameworks/electron/POWER_OF_10.md) |
| Axum | [POWER_OF_10.md](frameworks/axum/POWER_OF_10.md) |
| FastAPI | [POWER_OF_10.md](frameworks/fastapi/POWER_OF_10.md) |
| PyTorch | [POWER_OF_10.md](frameworks/pytorch/POWER_OF_10.md) |

### Patterns

- [Feature-Sliced Hexagonal](patterns/FEATURE_SLICED_HEXAGONAL.md) — AI agent swarm development in monorepos
- [Minimal Dependencies](patterns/MINIMAL_DEPENDENCIES.md) — When to use vs. avoid libraries
- [Code Comments](patterns/COMMENTS.md) — Inline, concise commenting guidelines

### Workflows

- [Lisa Flow](workflows/LISA_FLOW.md) — Structured AI development with TDD

### Stacks

- [FastMVP](stacks/fastmvp/README.md) — Vercel + NextJS + Convex/Supabase + Clerk + Stripe ・ [Integration Preferences](stacks/fastmvp/INTEGRATION_PREFERENCES.md)

---

## Tooling

### VS Code

- [Review & Orchestration Mode](vscode/configuration/settings.json) — Settings optimized for reviewing AI-generated code

**Recommended Extensions:** [Claude Code](https://marketplace.visualstudio.com/items?itemName=anthropic.claude-code) ・ [Error Lens](https://marketplace.visualstudio.com/items?itemName=usernamehw.errorlens)

---

## Contributing

Contributions welcome! Feel free to submit guidelines for additional languages or improvements to existing ones.

## License

CC0 1.0 Universal — Public Domain
