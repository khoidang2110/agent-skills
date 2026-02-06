---
name: nestjs-clean-hex-cqrs-api-playbook
description: Use this skill FIRST when implementing or modifying ANY NestJS API. Enforces Clean Architecture + Hexagonal (Ports & Adapters) with CQRS-lite, Prisma 6.x persistence, and strict Nx module boundaries.
---

# NestJS API Playbook
**Clean Architecture + Hexagonal (Ports & Adapters) + CQRS-lite + Prisma 6.x**

This skill defines the mandatory architecture, layering rules, and step-by-step workflow
for implementing APIs in this repository.

> If you are adding or modifying an API endpoint, you MUST follow this file.

---

## TL;DR - Mandatory Flow

**Controller (libs/api)**
-> **QueryService / UseCase (libs/application/features)**
-> **Inject Port token (libs/application/contracts)**
-> **Adapter + PersistenceModule (libs/persistence/repositories)**
-> **Prisma single schema file (libs/persistence/prisma/schema.prisma)**

Architecture style:
- Clean Architecture
- Hexagonal (Ports & Adapters)
- CQRS-lite (QueryService vs UseCase)

---

## 1. Architecture Enforcement (NON-NEGOTIABLE)

### 1.1 Nx boundaries
- Do not relax or bypass `@nx/enforce-module-boundaries`.
- Do not add exceptions just to make it work.
- If a violation occurs:
  - Move the file to the correct layer, or
  - Reverse the dependency using a Port + Token.

### 1.2 Composition root
- `apps/api` (and `libs/api`) are composition roots.
- They assemble modules and wire dependencies.
- They must not contain business logic.

### 1.3 Dependency direction (strict)
- `libs/api` depends on `libs/application` + `libs/shared`.
- `libs/application` depends on `libs/application/contracts` (+ `libs/shared` + `libs/domain` if exists).
- `libs/persistence` depends on `libs/application/contracts` and implements ports.
- `libs/application/contracts` depends on nothing (or shared types).

No other direction is allowed.

---

## 2. Layers & Allowed Imports

### 2.1 API layer (`libs/api`)
Responsible for:
- routing
- guards & roles
- DTO validation
- swagger decorators
- mapping input -> QueryService / UseCase

Rules:
- Must NOT import:
  - PrismaClient / PrismaService / PrismaModule
  - persistence adapters/repos
  - any persistence prisma package (even type-only)
  - `@prisma/client` (even type-only)
- If API needs enums/types, they MUST come from:
  - cross-feature -> `libs/shared/types/`
  - feature-scoped contract -> `libs/application/contracts/<feature>/dtos/`
- Shared enums/types:
  - cross-feature -> `libs/shared/types/`
  - feature-scoped -> `libs/application/contracts/<feature>/dtos/`
### 2.2 Application layer (`libs/application`)
Responsible for:
- business flows
- orchestration
- invariants enforcement

Rules:
- May depend on:
  - contracts (ports + tokens)
  - shared

- Must NOT import:
  - PrismaClient / PrismaService / PrismaModule
  - persistence adapters/entities
  - `@prisma/client` (even type-only)
  - any persistence prisma package (even type-only)
- Application uses contract/shared enums/types only.
- Use contracts/shared types instead of persistence types.
- Only QueryService / UseCase can throw `DomainError`.

## 2.3 Cross-cutting conventions (helpers / mappers / builders) (LOCKED)

Goal: keep feature code discoverable, avoid random `common/` folders, and keep dependency direction clean.

### 2.3.1 General rules
- Prefer colocating helpers inside the feature that owns them.
- Do NOT create new global `common/` folders unless the code is truly reused by ≥2 features.
- Helpers MUST NOT import from persistence adapters or Prisma runtime.
- Feature barrel exports (`libs/application/features/<feature>/index.ts`) MUST export ONLY:
  - QueryServices / UseCases
  - (optional) feature-level DTOs/types that are application-owned
- ❌ Feature barrels MUST NOT re-export from `@tps/persistence/prisma` (no persistence leakage).

### 2.3.2 Where to put helpers (by layer)

#### A) API layer helpers (libs/api)
Use for:
- request/response mapping specific to controllers
- swagger decorator wrappers
- pipe/validation helpers (DTO-centric)

Location:
- `libs/api/controllers/<feature>/<role>/mappers/`
- `libs/api/controllers/<feature>/<role>/dtos/`
- `libs/api/controllers/<feature>/<role>/helpers/`

Rules:
- Must be controller-scoped.
- Must NOT be reused across roles unless duplicated intentionally or moved to shared.
- Do NOT export these folders outside the controller/module.

Naming:
- `*.controller.ts`
- `*.dto.ts`
- `*.api-mapper.ts`
- `*.api.helper.ts`

#### B) Application layer helpers (libs/application)
Use for:
- mapping port return data (plain objects) -> application response DTO
- business rule validators
- builders for application objects (e.g., snapshot payload)
- pure calculation utilities for this feature

Location (recommended):

Query-specific:
- `libs/application/features/<feature>/queries/mappers/`
- `libs/application/features/<feature>/queries/validators/`

UseCase-specific:
- `libs/application/features/<feature>/usecases/mappers/`
- `libs/application/features/<feature>/usecases/validators/`
- `libs/application/features/<feature>/usecases/builders/`

Feature-shared (only if reused by multiple queries/usecases in the same feature):
- `libs/application/features/<feature>/helpers/`
- `libs/application/features/<feature>/builders/`
- `libs/application/features/<feature>/mappers/`

Rules:
- Keep helpers PRIVATE to the feature by default.
- Do NOT export helpers/, mappers/, builders/, validators/ from the feature index.ts.
- Do NOT create ad-hoc common/ helper folders.

Naming conventions:
- Mapper: `*.mapper.ts` (pure functions)
- Builder: `*.builder.ts` (pure constructors/composers)
- Validator: `*.validator.ts` (may throw DomainError)
- Helper: `*.helper.ts` / `*.util.ts` (pure utilities)

Validator rules:

Validators MUST be pure (no IO).

If a validator needs DB checks, the Query/UseCase must load data via ports first, then pass data into the validator.

Examples:

balance/queries/mappers/balance-response.mapper.ts

import-receipt/usecases/validators/import-receipt-status.validator.ts

file/helpers/file-size.helper.ts

#### C) Contracts layer (libs/application/contracts)
Use for:
- ports + tokens
- DTO/types referenced by Port interfaces (stable contracts)

Rules:
- No helpers here.
- DTOs here must be stable and meant as a contract.
- No business logic.

#### D) Persistence layer helpers (libs/persistence)
Use for:
- mapping Prisma records -> adapter return models (plain objects)
- Prisma query argument builders
- DB error mapping utilities

Location:
- `libs/persistence/repositories/<feature>/mappers/`
- `libs/persistence/repositories/<feature>/builders/` (Prisma args builders)
- `libs/persistence/repositories/<feature>/errors/`

Naming:
- Prisma args builder: `*.prisma-args.builder.ts`
- Mapper: `*.persistence.mapper.ts`
- Error mapping: `*.persistence-error.mapper.ts`

Rules:
- Persistence helpers may import Prisma types and PrismaService.
- Must NOT import application QueryService/UseCase.
- Adapters MUST return plain objects/DTOs, not Prisma model instances.
### 2.3.3 Standard feature folder layout (recommended)

Folder names MUST NOT start with '_' in this repo.

Application:
```text
libs/application/features/<feature>/
  index.ts
  queries/
    get-*.query.ts
    mappers/
      *.mapper.ts
  usecases/
    *.usecase.ts
    validators/
      *.validator.ts
    builders/
      *.builder.ts
  helpers/
    *.helper.ts
```

Persistence:
```text
libs/persistence/repositories/<feature>/
  <feature>.adapter.ts
  <feature>.persistence.module.ts
  mappers/
    *.persistence.mapper.ts
  builders/
    *.prisma-args.builder.ts
  errors/
    *.persistence-error.mapper.ts
```

API:
```text
libs/api/controllers/<feature>/<role>/
  *.controller.ts
  dtos/
    *.dto.ts
  mappers/
    *.api-mapper.ts
  helpers/
    *.api.helper.ts
```


---

### 2.3.4 Mapper rules (strict)
- API mappers map: HTTP DTO <-> Application request/response DTO.
- Application mappers map: Port return data <-> Application response DTO.
- Persistence mappers map: Prisma records <-> Adapter return data.

Rules:
- Mappers MUST be pure functions (no IO).
- Builders/mappers/helpers must not call DB, HTTP, queue, filesystem.
- Adapters MUST NOT return Prisma model instances. Return plain objects/DTOs only.
- Validators may throw DomainError.
- Builders must be deterministic (no Date.now/random unless injected explicitly).

---
### 2.3.5 DTO placement rules (LOCKED)

Goal: prevent DTO sprawl and keep contracts stable.

#### A) API DTOs (HTTP-facing)
Use for:
- request validation (class-validator)
- swagger decorators / OpenAPI schema
- controller response shapes when they are strictly “web contract”

Location:
- `libs/api/controllers/<feature>/<role>/dtos/`

Rules:
- API DTOs may use class-validator + swagger decorators.
- API DTOs must NOT be imported by persistence.

#### B) Application feature DTOs (internal)
Use for:
- internal query/usecase input/output types that are feature-private
- intermediate data shapes (not a public contract)

Location:
- `libs/application/features/<feature>/dtos/` (optional)
- or colocated near the usecase/query that owns it

Rules:
- Feature DTOs are not “public contracts”.
- Do NOT export them from feature index.ts unless explicitly needed.

#### C) Contracts DTOs (port contracts)
Use for:
- port input/output types and shared DTOs used across layers
- stable shapes that application <-> persistence agree on

Location:
- `libs/application/contracts/<feature>/dtos/`

Rules:
- If a type appears in a Port interface, it MUST live in contracts.
- Contracts must not contain logic.
### 2.3.5a JSON Naming Convention (SNAKE_CASE) (LOCKED)

Goal: keep API contract consistent. This repo’s HTTP response contract is snake_case.

#### A) Response contract (MANDATORY)

All JSON responses from API endpoints MUST be snake_case, including:

nested objects

arrays of objects

pagination metadata fields

Examples:

✅ voided_at, void_reason, created_at

❌ voidedAt, voidReason, createdAt

#### B) API DTO naming (MANDATORY)

All DTOs in libs/api/controllers/**/dtos/ MUST use snake_case property names.

Swagger/OpenAPI schemas MUST reflect snake_case (i.e., do not rely on runtime transforms that hide naming differences).

#### C) Mapping responsibility (STRICT)

Conversion between naming conventions is the responsibility of API layer mappers only:

Application/Contracts DTO (camelCase) → API DTO (snake_case)

Application layer MUST NOT “shape” output for HTTP naming.

Persistence layer MUST NOT depend on API DTOs.

#### D) Pagination response (MANDATORY) (UPDATED)

Pagination metadata MUST be snake_case and returned under page_meta.

API layer MUST NOT return/ expose shared PageResult / PageMeta directly in HTTP responses (because they are camelCase contracts).

Controllers MUST map pagination results to API DTOs (snake_case) via API mappers.

Required fields (standard):

page_meta.page

page_meta.limit

page_meta.total_items

page_meta.total_pages

Optional fields (if used, still snake_case):

page_meta.has_next

page_meta.has_prev

Notes:

Do NOT use totalItems, totalPages, pageSize in HTTP responses.

If existing endpoints return camelCase pagination meta, follow the Legacy / drift policy.
#### E) Legacy / drift policy (MANDATORY)

New endpoints MUST NOT introduce camelCase response fields.

If an endpoint already returns camelCase:

mark as deprecated in Swagger (description + tag if available)

add a new version/route returning snake_case

migrate consumers, then remove old contract in a later release

#### F) Implementation guidance (RECOMMENDED)

Use explicit API mappers (*.api-mapper.ts) for conversion instead of global interceptors.

Keep mappers pure (no IO) and controller-scoped.

### 2.3.6 Definitions: Mapper vs Builder vs Helper (LOCKED)

These terms are often mixed. Use this taxonomy to keep code discoverable.

#### Mapper
Purpose:
- transform shape A -> shape B (usually 1-to-1)

Characteristics:
- pure function (no IO)
- minimal/no business rules
- typical use: Entity/Port result -> Response DTO

Naming:
- `*.mapper.ts`

Functions:
- `toXxxDto`, `toXxxDtos`

#### Builder
Purpose:
- construct a new object by composing multiple sources and/or applying rules
- typical use: snapshot payload, PDF model, export model, schema-versioned payload

Characteristics:
- pure & deterministic
- may include normalization and field selection
- MUST NOT do DB/HTTP/queue/file IO

Naming:
- `*.builder.ts`

Functions:
- `buildXxxPayloadV1`, `buildXxxModel`

#### Helper (utility)
Purpose:
- small reusable pure utilities (formatting, checksum, stable stringify, etc.)

Characteristics:
- pure function
- no IO
- no feature business rules

Naming:
- `*.helper.ts` or `*.util.ts`

Place in feature `helpers/` or `libs/shared/utils/` if reused by ≥2 features.

#### Writer/Service (orchestrator)
Purpose:
- orchestration that involves transactions and persistence calls

Characteristics:
- may do IO (DB/tx)
- belongs in UseCase (application) or Adapter (persistence), depending on responsibility

Naming:
- `*.writer.ts`, `*.service.ts` (avoid calling these “helper”)
- If it opens a Prisma transaction or calls an adapter, it is NOT a helper; name it `*UseCase`, `*Writer`, or `*Service`.

### 2.3.7 Mapper sample pattern (RECOMMENDED)

Use this when the response contract must stay stable even if storage changes.

Location:
- `libs/application/features/<feature>/queries/mappers/`

Example (Company):
- `company-response.mapper.ts` exports pure functions:
  - `toCompanyResponseDto(item)`
  - `toCompanyResponseDtos(items)`

Rules:
- Do NOT import Prisma runtime.
- Prefer minimal input “shape” types if you want to avoid leaking Prisma types.

### 2.3.8 Snapshot layout (RECOMMENDED)

For features with snapshot/versioning (e.g., Sales Invoice):

Application:
```text
libs/application/features/<feature>/
  usecases/
    *.usecase.ts
  snapshot/
    <feature>.snapshot.types.ts
    <feature>.snapshot.builder.ts     # builds payload (pure)
  helpers/
    checksum.helper.ts                # optional if feature-only
```

Contracts:
```text
libs/application/contracts/<feature>/
  ports/
    <feature>.snapshot.port.ts
  dtos/
    snapshot.dtos.ts                  # only if referenced by ports
```

Persistence:
```text
libs/persistence/repositories/<feature>/
  snapshot/
    <feature>.snapshot.adapter.ts     # persist snapshot records only
```

Rules:
- Snapshot payload building MUST be in application (builder).
- Persistence only persists {payload, checksum, version, type}.
- Builders MUST be pure and must not load data. UseCases load aggregates via ports, then pass to builders.
### 2.4 Contracts (`libs/application/contracts`)
Contains:
- ports (interfaces)
- tokens
- shared DTOs (if needed)

Token pattern (canonical):
```ts
export const BALANCE_QUERY_PORT =
  'balance/BalanceQueryPort' as const;
```

Rules:
- Application injects by token.
- Never inject adapters directly.

### 2.5 Persistence (`libs/persistence`)
Contains:
- Prisma schema + migrations
- Prisma client module/provider
- adapters implementing ports
- persistence modules that bind tokens

Rules:
- Implements ports defined in contracts.
- One canonical implementation per port.
- No parallel implementations for the same port.
- Adapters use PrismaService (injected) and may use Prisma-generated types.
- Use Prisma 6.x only (no TypeORM).

### 2.6 Shared (`libs/shared`)
Contains:
- guards
- decorators (`@CurrentUser`, `@Roles`)
- `DomainError`
- error codes
- response helpers

Rules:
- No business logic here.

---

## 3. Workflow: Adding a New API (COPY THIS)

### Step A - Define Port + Token (Contracts)
Location:
- `libs/application/contracts/<feature>/ports/`
- `libs/application/contracts/<feature>/<feature>.tokens.ts`

Files:
- `<feature>.query.port.ts` (read)
- `<feature>.usecase.port.ts` or `<feature>.command.port.ts` (write)
- `<feature>.tokens.ts`

Rules:
- Application depends on ports, not adapters.
- Token is injected everywhere (no direct adapter injection).

### Step B - QueryService / UseCase (Application)
Location:
- `libs/application/features/<feature>/queries/`
- `libs/application/features/<feature>/usecases/`

Naming:
- Query: `GetXxxQueryService`, `ListXxxQueryService`
- UseCase: `CreateXxxUseCase`, `UpdateXxxUseCase`, `AdjustXxxUseCase`

Pattern:
```ts
@Inject(TOKEN)
private readonly port: XxxPort;
```

Error handling:
```ts
throw new DomainError(ErrorCode.X, 'message');
```

Barrel export (required):
- `libs/application/features/<feature>/index.ts`

### Step C - Adapter + PersistenceModule
Adapter:
- `libs/persistence/repositories/<feature>/<feature>.adapter.ts`

Module:
- `libs/persistence/repositories/<feature>/<feature>.persistence.module.ts`

Binding pattern:
```ts
providers: [
  Adapter,
  { provide: XXX_QUERY_PORT, useExisting: Adapter },
  { provide: XXX_USECASE_PORT, useExisting: Adapter },
],
exports: [XXX_QUERY_PORT, XXX_USECASE_PORT],
```

Rules:
- One adapter may implement multiple ports (use `useExisting`).
- One port token has exactly one canonical implementation (avoid drift).

### Step D - Controller + ApiModule
Controller location:
- `libs/api/controllers/<feature>/<role>/`

Role conventions:
- `user/`
- `admin/`
- `backoffice/`

Controller rules:
- Guards + roles required.
- Use `@CurrentUser()` for identity.
- Do not accept `{userId}` from params for user APIs.

Api module:
- `libs/api/controllers/<feature>/<feature>.module.ts`

Imports:
- `<Feature>PersistenceModule`

Providers:
- QueryService / UseCase classes (repo currently provides directly in ApiModule)

---

## 4. Naming & Route Conventions

### 4.1 Naming
- Query: `GetXxxQueryService`
- UseCase: `CreateXxxUseCase`, `AdjustXxxUseCase`
- Token: `<FEATURE>_QUERY_PORT`
- Module: `<Feature>PersistenceModule`

### 4.2 Routes
User APIs:
- `/api/v2/user/...`

Admin:
- `/api/v2/admin/...`

Backoffice:
- `/api/v2/backoffice/...`

Rules:
- No `/me` for multi-actor resources.
- `/me` only for identity/profile.

---

## 5. Definition of Done (MANDATORY)

An API is DONE only when ALL are present:
- Controller (guards, roles, swagger)
- QueryService / UseCase
- Port + Token
- Adapter implementation
- PersistenceModule binding
- ApiModule wiring
- Nx boundaries clean (no violations)

---

## 6. Error Conventions (`DomainError`)

### 6.1 Rules
- Application layer throws `DomainError` only.
- Application never throws `HttpException`.
- Adapter:
  - expected business error -> map to `DomainError`
  - unexpected error -> throw raw (500)

### 6.2 Error code naming
- `UPPER_SNAKE_CASE`
- Prefixed by feature

Examples:
- `BALANCE_NOT_FOUND`
- `SUPPORT_TICKET_CLOSED`
- `FILE_TYPE_NOT_ALLOWED`

---

## 7. Transactions
- Use Prisma transactions (`$transaction`).
- Avoid manual nested transactions.
- Transaction boundary belongs in UseCase.

---

## 8. Events - Outbox Pattern (Simplified)

Purpose:
- Implement reliable side-effects (email, notifications, external calls) using Outbox Pattern, without over-engineering.

This setup is optimized for:
- Prisma 6.x
- Clean Architecture + Hexagonal
- CQRS-lite
- 10 or fewer events
- Dynamic PDF preview reuse

### Event Classification (MANDATORY)
In-memory events (best-effort):
- Use when delivery is not critical.
- Example: `UserRegistered`
- Implementation: In-memory event bus.
- No DB durability, no retry guarantee.

Outbox events (durable):
- Use when event must not be lost.
- Examples: `ImportReceiptFinished`, `TestEmailRequested`
- Implementation: DB-backed outbox, async processing, retry with backoff.

Rule:
- If losing the event is unacceptable -> Outbox.
- If best-effort is fine -> In-memory.

### Layer Responsibilities (STRICT)
Domain (`libs/domain`):
- Event definitions only
- `EventNames`
- `*.event.ts` payload types
- No Prisma
- No queues
- No handlers

Application (`libs/application`):
- UseCases orchestrate business flow
- Emits domain events
- Writes outbox inside transaction for durable events
- Allowed: ports + tokens, `DomainError`
- Forbidden: `PrismaClient`, BullMQ, email providers
- Naming rule: `*.write-outbox.handler.ts`

Contracts (`libs/application/contracts`):
- Define ports + tokens
- Example: `OUTBOX_WRITER_PORT`
- Application injects by token, never adapters

Infrastructure / Persistence:
- Prisma adapters
- Outbox processor
- Queue enqueue
- Workers (email, PDF, etc.)
- Allowed: `PrismaClient`, BullMQ, external services

### Transaction Rule (NON-NEGOTIABLE)
For outbox events, business update and outbox insert MUST happen in the same Prisma transaction.

Correct:
```ts
await prisma.$transaction(async (tx) => {
  await updateBusiness(tx);
  await outboxWriter.write(tx, event);
});
```

Incorrect:
- Insert outbox after commit.
- Separate transactions.

### Outbox Processing (Minimal)
Required components:
- OutboxWriter (insert row)
- OutboxProcessor (poll + retry)
- Single worker (10 or fewer events)
- No routers, no per-event queues (yet)

Processing rules:
- Pick rows where `status = PENDING` and `next_run_at <= now()`.
- On success: mark SENT.
- On failure: increment attempts, set `next_run_at` (backoff).
- Use DB index on `(status, next_run_at)`.

### Worker Strategy (Simple & Safe)
PDF + Email handling:
- Do not call HTTP preview endpoints from workers.
- Reuse the same PDF renderer/service.

Flow:
- Preview API:
  - Query DB
  - Render PDF
  - Return response
- Worker:
  - Query DB
  - Render PDF using same renderer
  - Send email

This avoids:
- HTTP dependency
- Auth issues
- Circular calls

### Snapshot Policy (Current Decision)
- `finish-scan` locks data.
- Worker renders PDF from DB at processing time.
- No snapshot payload for now.

This is acceptable as long as finished data is immutable.

Future upgrade (optional):
- Add JSON snapshot (no binary)
- Keep flow unchanged

### Folder Structure (Minimal)
```
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
```

### Idempotency (Recommended)
When enqueueing jobs:
- `jobId = outbox_event.id`

Prevents duplicate processing.

---

## 9. Prisma Schema Architecture (LOCKED)

This repo uses a SINGLE Prisma schema file.

### 9.1 Entry & source of truth
Prisma CLI runs with:
- `--schema=libs/persistence/prisma/schema.prisma`

Source of truth:
- `libs/persistence/prisma/schema.prisma`

Rules:
- `schema.prisma` contains datasource + generator + ALL models + ALL enums.
- Do NOT use Prisma schema folder mode in this repo.

Forbidden paths:
- `libs/persistence/prisma/schema/`
- `libs/persistence/prisma/schema/models/`
- `libs/persistence/prisma/schema/enums/`

### 9.2 Schema layout (recommended)
Inside `schema.prisma`, use this ordering:

1) datasource
2) generator
3) `// ===== ENUMS (PERSISTENCE) =====`
4) `// ===== MODELS (by domain blocks) =====`

Notes:
- Keep models grouped by domain using comment headers.
- Keep naming consistent across relations.

### 9.3 Change workflow (mandatory)
When modifying schema:
- update `schema.prisma`
- run:
  - `prisma format`
  - `prisma validate`
  - `prisma generate`
- run migration only if schema semantics changed.

---

## 10. Enum Rules (LOCKED)

Goal:
- Prevent persistence leakage.
- Keep API/Application stable even if DB enums change.
- Keep Nx boundaries clean.

### 10.1 Never import Prisma enums outside persistence

API (libs/api) and Application (libs/application) MUST NOT import:
- `@prisma/client` (even type-only)
- any persistence prisma package (even type-only)

Persistence (libs/persistence) may use Prisma enums/types freely.

### 10.2 Enum placement by scope (canonical)

#### A) Contract enums (used in API/Application/Ports)

Cross-feature enums -> libs/shared/types/

Feature-scoped enums that appear in Port interfaces -> libs/application/contracts/<feature>/dtos/

These are the ONLY enums allowed to appear in:

Controller DTOs (validation/swagger)

QueryService/UseCase inputs/outputs

Port input/output types

Optional:
- Prefer const object + union type for contract enums to support validation/swagger easily.

#### B) Persistence enums (DB-facing only)

If an enum is truly DB-internal and never leaves persistence, it MAY exist only in schema.prisma.
Persistence enums MUST NOT appear in Port DTOs or API DTOs.

### 10.3 Naming conventions

Enum type names: PascalCase (e.g. InvoiceStatus, UserRole)

Values: SCREAMING_SNAKE_CASE (e.g. PENDING, APPROVED)

Forbidden: generic names like Status, Type (ambiguous)

### 10.4 Mapping rule (mandatory)

Adapters MUST map persistence enum -> contract enum.

If values match 1:1, mapping can be a simple cast/assign.

If values differ, mapping MUST be explicit (switch/map) and tested.

Adapters MUST NOT return Prisma model instances or Prisma enum types.
Adapters return plain objects using contract enums/types.

### 10.5 Enum change policy (migration-safe)

Do NOT rename an enum value that already exists in production data in a single step.

Safe workflow:

Add new enum value

Backfill data (SQL/script)

Update application to use new value

Remove old value in a later migration
