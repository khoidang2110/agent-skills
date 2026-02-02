# Architecture & Layering (Non-negotiable)

## Style
- Clean Architecture
- Hexagonal (Ports & Adapters)
- CQRS-lite (QueryService vs UseCase)

## Nx boundaries
- Do NOT relax or bypass `@nx/enforce-module-boundaries`.
- If a violation occurs, move the file to the correct layer or reverse dependency via port + token.

## Composition root
- `apps/api` and `libs/api` assemble modules and wire dependencies.
- They must NOT contain business logic.

## Dependency direction (strict)
- `libs/api` -> `libs/application` + `libs/shared`
- `libs/application` -> `libs/application/contracts` (+ `libs/shared` + `libs/domain` if exists)
- `libs/persistence` -> `libs/application/contracts`
- `libs/application/contracts` -> nothing (or shared types only)

No other direction is allowed.

## Layer rules

### API (`libs/api`)
Responsible for routing, guards/roles, DTO validation, swagger, mapping input -> query/usecase.
- Must NOT import PrismaClient, Prisma types, or persistence adapters.
- May import application queries/usecases and shared guards/decorators.

### Application (`libs/application`)
Responsible for business flows, orchestration, invariants enforcement.
- May depend on contracts and shared.
- Must NOT import PrismaClient, Prisma types, persistence adapters/entities.
- Only QueryService / UseCase can throw DomainError.
- Must NOT re-export anything from `@tps/persistence/prisma` in feature barrels.

### Contracts (`libs/application/contracts`)
Contains ports (interfaces), tokens, shared DTOs (only if needed).
- Application injects by token, never adapters directly.

### Persistence (`libs/persistence`)
Contains Prisma schema + migrations, client module/provider, adapters implementing ports, persistence modules that bind tokens.
- Implements ports defined in contracts.
- One canonical implementation per port.
- Adapters map DB errors to DomainError.
- Prisma 6.x only (no TypeORM).

### Shared (`libs/shared`)
Contains guards, decorators, DomainError, error codes, response helpers.
- No business logic.
