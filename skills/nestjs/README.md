# skill-nestjs

Agent skills and architectural playbooks for building
production-grade NestJS APIs using Clean Architecture,
Hexagonal (Ports & Adapters), and CQRS-lite.

## Why this repo exists
- Enforce strict architecture boundaries
- Prevent dependency drift in large Nx monorepos
- Make APIs predictable for both humans and AI agents

## What's inside
- API playbook (Clean + Hex + CQRS-lite)
- Error conventions (DomainError)
- Persistence adapters (TypeORM)
- Event patterns (in-memory vs outbox)

## How to use in a project
1. Copy `SKILL.md` into your project: docs/agent/SKILL.md
2. Add `AGENTS.md` to your repo root
3. Instruct agents to read the playbook before coding

---
## AGENTS.md template (RECOMMENDED)

Create a file named `AGENTS.md` at the root of your project.

This file acts as the **entry rule** for all AI coding agents
(Copilot, Claude, Cursor, Codex, etc.).

### Example `AGENTS.md`

```yaml
# Agent Rules – NestJS Clean Architecture

Before implementing or modifying ANY API in this repository:

1. Read the API playbook:
- docs/agent/SKILL.md

2. Follow strictly:
- Clean Architecture
- Hexagonal (Ports & Adapters)
- CQRS-lite (QueryService vs UseCase)
- Nx module boundaries (NO relax, NO bypass)

Hard rules:
- API layer MUST NOT import persistence or adapters
- Application layer injects dependencies via tokens ONLY
- DomainError is the only error thrown from application
- Persistence binds ports using useExisting
- apps/api is composition root only (no business logic)

If unsure:
- Copy an existing feature (e.g. Balance) exactly.
- Do not invent new patterns without architectural justification.
```

## Design principles
- Architecture over convenience
- Explicit dependencies via tokens
- One canonical implementation per port
- Composition root only in API layer

## Who this is for
- Backend engineers using NestJS
- Teams using Nx monorepos
- Developers working with AI coding agents
