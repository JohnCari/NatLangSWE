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
└── vscode/         # VS Code configurations
```

## Languages

### Rust

- **[Power of 10 Rules for Rust](languages/rust/POWER_OF_10.md)** — NASA JPL's safety-critical coding guidelines adapted for Rust and AI-assisted development

### Python

*Coming soon*

### TypeScript

*Coming soon*

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

*Coming soon — Cross-language best practices for error handling, testing, security*

## VS Code

*Coming soon — Recommended settings, extensions, and keybindings*

## Usage

Copy the relevant guidelines into your AI assistant's context:

### Claude Code
Add to your project's `CLAUDE.md` file or reference directly in prompts.

### Other Tools
Include the guideline content in your system prompt or context window.

## Contributing

Contributions welcome! Feel free to submit guidelines for additional languages or improvements to existing ones.

## License

CC0 1.0 Universal — Public Domain

