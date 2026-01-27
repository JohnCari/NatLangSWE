# NatLangDev

> Coding in "Natural Language" — Prevent vibe coding with structured AI prompts

## What is NatLangDev?

**NatLangDev** (Natural Language Developer Guide) is a collection of curated guidelines that help developers communicate effectively with AI coding assistants like Claude Code.

### The Problem: Vibe Coding

"Vibe coding" happens when developers prompt AI assistants without clear constraints or standards, resulting in:
- Inconsistent code quality
- Security vulnerabilities
- Poor architectural decisions
- Code that "works" but violates best practices

### The Solution: Structured Guidelines

Instead of vague prompts like *"write me a web server"*, NatLangDev provides explicit rules that AI assistants follow. By adding these guidelines to your AI's context, you get code that adheres to proven standards — whether that's NASA's safety-critical rules, idiomatic language patterns, or framework-specific conventions.

## Structure

```
natlangdev/
├── languages/
│   ├── rust/       # Rust guidelines
│   ├── python/     # Python guidelines
│   └── typescript/ # TypeScript guidelines
├── frameworks/
│   ├── nextjs/     # NextJS (TSX) guidelines
│   ├── axum/       # Rust Axum guidelines
│   ├── fastapi/    # FastAPI (Python) guidelines
│   └── pytorch/    # PyTorch (Python) guidelines
├── patterns/       # Cross-language design patterns
├── workflows/      # Development workflows
└── vscode/         # VS Code configurations
```

## Languages

### Rust

- **[Power of 10 Rules for Rust](languages/rust/POWER_OF_10.md)** — NASA JPL's safety-critical coding guidelines adapted for Rust and AI-assisted development

### Python

- **[Power of 10 Rules for Python](languages/python/POWER_OF_10.md)** — NASA JPL's safety-critical coding guidelines adapted for Python

### TypeScript

- **[Power of 10 Rules for TypeScript](languages/typescript/POWER_OF_10.md)** — NASA JPL's safety-critical coding guidelines adapted for TypeScript

## Frameworks

### NextJS (TSX)

*Coming soon*

### Axum (Rust)

*Coming soon*

### FastAPI (Python)

*Coming soon*

### PyTorch (Python)

*Coming soon*

## Patterns

- **[Feature-Sliced Hexagonal Architecture](patterns/FEATURE_SLICED_HEXAGONAL.md)** — The definitive pattern for parallel AI agent swarm development in a monorepo
- **[Minimal Dependencies](patterns/MINIMAL_DEPENDENCIES.md)** — Philosophy on when to use vs. avoid libraries
- **[Code Comments](patterns/COMMENTS.md)** — Inline, concise, and descriptive commenting guidelines

## Workflows

- **[Lisa Flow](workflows/LISA_FLOW.md)** — Structured AI development with automatic TDD and self-healing tests

## VS Code

- **[Review & Orchestration Mode](vscode/configuration/settings.json)** — VS Code settings optimized for reviewing AI-generated code (disables autocomplete, strips UI bloat, enhances diff view)

### Recommended Extensions

- **[Claude Code](https://marketplace.visualstudio.com/items?itemName=anthropic.claude-code)** — Run Claude Code directly in VS Code
- **[Error Lens](https://marketplace.visualstudio.com/items?itemName=usernamehw.errorlens)** — Inline error/warning highlighting for quick review

Other extensions vary by project — ask Claude Code for recommendations based on your stack.

## Contributing

Contributions welcome! Feel free to submit guidelines for additional languages or improvements to existing ones.

## License

CC0 1.0 Universal — Public Domain

