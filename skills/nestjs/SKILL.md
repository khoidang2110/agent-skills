---
name: nestjs-clean-hex-cqrs-api-playbook
description: Use this skill FIRST when implementing or modifying ANY NestJS API. Enforces Clean Architecture + Hexagonal (Ports & Adapters) with CQRS-lite, Prisma 6.x persistence, and strict Nx module boundaries.
---

# NestJS API Playbook  
**Clean Architecture + Hexagonal (Ports & Adapters) + CQRS-lite + Prisma 6.x**

This skill defines the **mandatory architecture, layering rules, and step-by-step workflow**
for implementing APIs in this repository.

> If you are adding or modifying an API endpoint, you MUST follow this file.

---

## TL;DR – Mandatory Flow

**Controller (libs/api)**  
→ **QueryService / UseCase (libs/application/features)**  
→ **inject Port token (libs/application/contracts)**  
→ **Adapter + PersistenceModule (libs/persistence/repositories)**  
→ **Prisma schema/models (libs/persistence/prisma/schema.prisma)**

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
  - PrismaClient
  - Prisma models/types
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
  - PrismaClient
  - Prisma types
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

Prisma schema + migrations

Prisma client module/provider

adapters implementing ports

persistence modules that bind tokens

Rules:

Implements ports defined in contracts

One canonical implementation per port

No parallel implementations for the same port

Adapters use PrismaClient (injected) and map DB errors to DomainError

Use Prisma 6.x only (no TypeORM)

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
Use Prisma transactions (`$transaction`)

Avoid manual nested transactions

Transaction boundary belongs in UseCase

8. Events - Outbox Pattern – Simplified Implementation 
Purpose

Implement reliable side-effects (email, notifications, external calls) using Outbox Pattern, without over-engineering.

This setup is optimized for:

Prisma 6.x

Clean Architecture + Hexagonal

CQRS-lite

≤ 10 events

Dynamic PDF preview reuse

Event Classification (MANDATORY)
In-Memory Events (best-effort)

Use when delivery is not critical.

Example: UserRegistered

Implementation:

In-memory event bus

No DB durability

No retry guarantee

Outbox Events (durable)

Use when event must not be lost.

Example:

ImportReceiptFinished

TestEmailRequested

Implementation:

DB-backed outbox

Async processing

Retry with backoff

Rule:

If losing the event is unacceptable → Outbox
If best-effort is fine → In-memory

Layer Responsibilities (STRICT)
Domain (libs/domain)

Event definitions only

EventNames

*.event.ts payload types

❌ No Prisma
❌ No queues
❌ No handlers

Application (libs/application)

UseCases orchestrate business flow

Emits domain events

Writes outbox inside transaction for durable events

Allowed:

Ports + tokens

DomainError

Forbidden:

PrismaClient

BullMQ

Email providers

Naming rule:

*.write-outbox.handler.ts


Handlers here only insert outbox rows, nothing else.

Contracts (libs/application/src/contracts)

Define ports + tokens

Example:

OUTBOX_WRITER_PORT

Application injects by token, never adapters.

Infrastructure / Persistence

Prisma adapters

Outbox processor

Queue enqueue

Workers (email, PDF, etc.)

Allowed:

PrismaClient

BullMQ

External services

Transaction Rule (NON-NEGOTIABLE)

For outbox events:

Business update and

Outbox insert

MUST happen in the same Prisma transaction.

✅ Correct:

prisma.$transaction(tx => {
  updateBusiness(tx)
  outboxWriter.write(tx, event)
})


❌ Incorrect:

Insert outbox after commit

Separate transactions

Outbox Processing (Minimal)
Required Components

OutboxWriter (insert row)

OutboxProcessor (poll + retry)

Single worker (≤ 10 events)

No routers, no per-event queues (yet).

Processing Rules

Pick rows where:

status = PENDING

next_run_at <= now()

On success:

mark SENT

On failure:

increment attempts

set next_run_at (backoff)

Use DB index (status, next_run_at)

Worker Strategy (Simple & Safe)
PDF + Email Handling

Do NOT call HTTP preview endpoints from workers

Reuse the same PDF renderer/service

Flow:

Preview API:

Query DB

Render PDF

Return response

Worker:

Query DB

Render PDF using same renderer

Send email

This avoids:

HTTP dependency

Auth issues

Circular calls

Snapshot Policy (Current Decision)

finish-scan locks data

Worker renders PDF from DB at processing time

No snapshot payload for now

This is acceptable as long as finished data is immutable.

Future upgrade (optional):

Add JSON snapshot (no binary)

Keep flow unchanged

Folder Structure (Minimal)
libs/
  domain/
    events/event-names.ts
    <feature>/<event>.event.ts

  application/
    eventing/
      in-memory-event-bus.ts
      subscriptions.ts
    features/
      **/*.usecase.ts
      **/*.write-outbox.handler.ts

  application/contracts/
    outbox/
      outbox-writer.port.ts
      outbox.tokens.ts

  infrastructure/
    outbox/
      outbox-writer.adapter.ts
      outbox-processor.service.ts
    messaging/bullmq/
      email.worker.ts

Idempotency (Recommended)

When enqueueing jobs:

jobId = outbox_event.id

Prevents duplicate processing.