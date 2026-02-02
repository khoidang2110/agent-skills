---
name: nestjs-clean-hex-cqrs-api-playbook
description: Use this skill FIRST when implementing or modifying ANY NestJS API. Enforces Clean Architecture + Hexagonal (Ports & Adapters) with CQRS-lite, Prisma 6.x persistence, and strict Nx module boundaries.
---

# NestJS API Playbook (Entry)
**Clean Architecture + Hexagonal (Ports & Adapters) + CQRS-lite + Prisma 6.x**

This file is the **entrypoint**. Keep it short and enforce the rules below. For details, open the referenced files.

If you are adding or modifying an API endpoint, you MUST follow this file.

## Non-negotiables
- Respect `@nx/enforce-module-boundaries` (no exceptions).
- Composition roots: `apps/api` and `libs/api` only wire modules; no business logic.
- Dependency direction (strict):
  - `libs/api` -> `libs/application` + `libs/shared`
  - `libs/application` -> `libs/application/contracts` (+ `libs/shared` + `libs/domain` if exists)
  - `libs/persistence` -> `libs/application/contracts`
  - `libs/application/contracts` -> nothing (or shared types only)
- No direct adapter injection; always inject by token.

## Mandatory flow (summary)
Controller (`libs/api`)
-> QueryService / UseCase (`libs/application/features`)
-> Port + token (`libs/application/contracts`)
-> Adapter + PersistenceModule (`libs/persistence/repositories`)
-> Prisma schema folder mode (`libs/persistence/prisma/schema`)

## Workflow (summary)
- Define ports + tokens in contracts.
- Implement QueryService / UseCase in application (DomainError only).
- Implement adapter + persistence module in persistence (bind tokens via `useExisting`).
- Implement controller + api module in api layer (guards/roles + swagger).

## Definition of Done (DoD)
An API is DONE only when all are present:
- Controller (guards, roles, swagger)
- QueryService / UseCase
- Port + token
- Adapter implementation
- PersistenceModule binding
- ApiModule wiring
- Nx boundaries clean (no violations)

## Read next (topic guides)
When needed, open these files (the runtime does NOT auto-read references):
- Architecture & layering rules: `references/01-architecture.md`
- New API workflow (step-by-step): `references/02-workflow-new-api.md`
- Feature layout (builders/mappers/DTOs): `references/03-feature-layout.md`
- DomainError conventions: `references/04-errors-domainerror.md`
- Transactions: `references/05-transactions.md`
- Outbox pattern: `references/06-outbox.md`
- Prisma schema folder mode: `references/07-prisma-folder-mode.md`
