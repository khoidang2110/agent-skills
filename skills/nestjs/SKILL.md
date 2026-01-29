---
name: nestjs-clean-hex-cqrs-api-playbook
description: Use this skill FIRST when implementing or modifying ANY NestJS API. Enforces Clean Architecture + Hexagonal (Ports & Adapters) with CQRS-lite and strict Nx module boundaries.
---

# NestJS API Playbook  
**Clean Architecture + Hexagonal (Ports & Adapters) + CQRS-lite**

This skill defines the **mandatory architecture, layering rules, and step-by-step workflow**
for implementing APIs in this repository.

> If you are adding or modifying an API endpoint, you MUST follow this file.

---

## TL;DR – Mandatory Flow

**Controller (libs/api)**  
→ **QueryService / UseCase (libs/application/features)**  
→ **inject Port token (libs/application/contracts)**  
→ **Adapter + PersistenceModule (libs/persistence/repositories)**  
→ **TypeORM entities (libs/persistence/typeorm/entities)**

Architecture style:
- Clean Architecture
- Hexagonal (Ports & Adapters)
- CQRS-lite (QueryService vs UseCase)

---

## 1. Architecture Enforcement (NON-NEGOTIABLE)

### 1.1 Nx boundaries
- ❌ DO NOT relax or bypass `@nx/enforce-module-boundaries`
- ❌ DO NOT add exceptions “just to make it work”
- If a violation occurs:
  - Move the file to the correct layer **or**
  - Reverse the dependency using a Port + Token

### 1.2 Composition root
- `apps/api` (and `libs/api`) are **composition roots**
- They:
  - assemble modules
  - wire dependencies
- They **must not** contain business logic

### 1.3 Dependency direction
api
↓
application
↓
contracts
↑
persistence


No other direction is allowed.

---

## 2. Layers & Allowed Imports

### 2.1 API layer (`libs/api`)
Responsible for:
- routing
- guards & roles
- DTO validation
- swagger decorators
- mapping input → QueryService / UseCase

Rules:
- ❌ Must NOT import:
  - TypeORM repositories
  - entities
  - persistence adapters
- ✅ May import:
  - application queries/usecases
  - shared guards/decorators

---

### 2.2 Application layer (`libs/application`)
Responsible for:
- business flows
- orchestration
- invariants enforcement

Rules:
- ✅ May depend on:
  - contracts (ports + tokens)
  - shared
- ❌ Must NOT import:
  - TypeORM
  - DataSource
  - persistence adapters/entities
- Only **QueryService / UseCase** can throw `DomainError`

---

### 2.3 Contracts (`libs/application/src/contracts`)
Contains:
- Ports (interfaces)
- Tokens
- Shared DTOs (if needed)

Token pattern (canonical):
```ts
export const BALANCE_QUERY_PORT =
  'balance/BalanceQueryPort' as const;
Rules:

Application injects by token

Never inject adapters directly

2.4 Persistence (libs/persistence)
Contains:

TypeORM entities

adapters implementing ports

persistence modules that bind tokens

Rules:

Implements ports defined in contracts

One canonical implementation per port

No parallel implementations for the same port

2.5 Shared (libs/shared)
Contains:

guards

decorators (@CurrentUser, @Roles)

DomainError

error codes

response helpers

No business logic here.

3. Workflow: Adding a New API (COPY THIS)
Step A — Define Port + Token (Contracts)
Location:

libs/application/src/contracts/<feature>/ports/
Files:

<feature>.query.port.ts (read)

<feature>.usecase.port.ts or <feature>.command.port.ts (write)

<feature>.tokens.ts

Rules:

Application depends on ports, not adapters

Token is injected everywhere

Step B — QueryService / UseCase (Application)
Location:

libs/application/src/features/<feature>/queries/
libs/application/src/features/<feature>/usecases/
Naming:

GetXxxQueryService

CreateXxxUseCase

UpdateXxxUseCase

Pattern:

@Inject(TOKEN)
private readonly port: XxxPort;
Error handling:

throw new DomainError(ErrorCode.X, 'message');
Barrel export (REQUIRED):

libs/application/src/features/<feature>/index.ts
Step C — Adapter + PersistenceModule
Adapter:

libs/persistence/src/repositories/<feature>/<feature>.adapter.ts
Module:

libs/persistence/src/repositories/<feature>/<feature>.persistence.module.ts
Binding pattern:

providers: [
  Adapter,
  { provide: XXX_QUERY_PORT, useExisting: Adapter },
  { provide: XXX_USECASE_PORT, useExisting: Adapter },
],
exports: [XXX_QUERY_PORT, XXX_USECASE_PORT],
Rules:

One adapter may implement multiple ports

Use useExisting

Avoid duplicate implementations

Step D — Controller + ApiModule
Controller location:

libs/api/src/controllers/<feature>/<role>/
Role conventions:

user/

admin/

backoffice/

Controller rules:

Guards + roles required

Use @CurrentUser() for identity

Do NOT accept {userId} from params for user APIs

Api module:

libs/api/src/controllers/<feature>/<feature>.module.ts
Imports:

<Feature>PersistenceModule

Providers:

QueryService / UseCase classes

4. Naming & Route Conventions
4.1 Naming
Query: GetXxxQueryService

UseCase: CreateXxxUseCase, AdjustXxxUseCase

Token: <FEATURE>_QUERY_PORT

Module: <Feature>PersistenceModule

4.2 Routes
User APIs:

/api/v2/user/...
Admin:

/api/v2/admin/...
Backoffice:

/api/v2/backoffice/...
Rules:

❌ No /me for multi-actor resources

/me only for identity/profile

5. Definition of Done (MANDATORY)
An API is DONE only when ALL are present:

Controller (guards, roles, swagger)

QueryService / UseCase

Port + Token

Adapter implementation

PersistenceModule binding

ApiModule wiring

Nx boundaries clean (no violations)

6. Error Conventions (DomainError)
6.1 Rules
Application layer throws DomainError ONLY

Application NEVER throws HttpException

Adapter:

expected business error → map to DomainError

unexpected error → throw raw (500)

6.2 Error code naming
UPPER_SNAKE_CASE

Prefixed by feature

Examples:

BALANCE_NOT_FOUND

SUPPORT_TICKET_CLOSED

FILE_TYPE_NOT_ALLOWED

7. Transactions
Use typeorm-transactional if enabled

Avoid manual nested transactions

Transaction boundary belongs in UseCase

8. Events
In-memory (RAM)
Use for non-critical internal flows:

UserRegistered

TestEmail

Outbox (DB-backed)
Use for reliable, cross-service events:

PaymentSucceeded

ItemsReceived

ImportReceiptQuantityRecorded

Rule:

If it must not be lost → Outbox

If best-effort → In-memory

Final Rule
If you are unsure:

Copy the Balance feature structure exactly.
Deviation requires architectural justification.