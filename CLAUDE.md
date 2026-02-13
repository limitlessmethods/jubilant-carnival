# CLAUDE.md

This file provides guidance for AI assistants working in this repository.

## Project Overview

**jubilant-carnival** is an automations project. The repository is in its early stages and does not yet contain application code, build tooling, or CI/CD configuration.

## Repository Structure

```
/
├── README.md      # Project description
└── CLAUDE.md      # This file – AI assistant guidance
```

## Current State

- No source code, build system, or dependency manager is configured yet
- No tests or CI/CD pipelines exist
- Single branch history starting from `master`

## Development Guidelines

When adding code to this repository, follow these conventions:

### General

- Keep commits small and focused with clear, descriptive messages
- Do not commit secrets, credentials, or `.env` files
- Prefer simple, readable solutions over clever abstractions

### Adding New Code

Since the project is nascent, the first contributor should:

1. Choose a language/runtime and add the appropriate config (e.g., `package.json`, `pyproject.toml`)
2. Set up a linter and formatter from the start
3. Add a testing framework and write tests alongside new code
4. Document build/run/test commands in this file once established

### Updating This File

Update CLAUDE.md whenever:

- A build system or package manager is added
- New scripts or commands become available (build, test, lint, format)
- Project structure changes significantly
- New conventions or architectural decisions are made
