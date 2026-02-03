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


---

## Design principles
- Architecture over convenience
- Explicit dependencies via tokens
- One canonical implementation per port
- Composition root only in API layer

## Who this is for
- Backend engineers using NestJS
- Teams using Nx monorepos
- Developers working with AI coding agents
