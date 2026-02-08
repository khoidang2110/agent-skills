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

  ---
### 2.1.1 Controller DTOs & API Mappers (LOCKED)
Goal: keep HTTP contract explicit (snake_case) and keep controllers thin.

#### A) Where API DTOs live (MANDATORY)
API DTOs (Swagger + class-validator/class-transformer) MUST live under `libs/api` only.

**Preferred locations (match this repo):**

1) **Feature-level DTOs (shared across roles)**
Use when multiple controllers/roles share the same request/response DTOs.
- `libs/api/controllers/<domain>/<feature>/dtos/`

Examples:
- `libs/api/controllers/catalog/store/dtos/`
- `libs/api/controllers/payouts/payment/dtos/`
- `libs/api/controllers/internal-fulfillment/import-receipt/dtos/`

2) **Role-scoped DTOs (only for one role controller)**
Use when the DTO is specific to one actor/route set.
- `libs/api/controllers/<domain>/<feature>/<role>/dtos/`

Example:
- `libs/api/controllers/support-ticket/backoffice/dtos/`

**API shared reusable DTOs (cross-feature)**
- `libs/api/shared/dtos/`
Example:
- `libs/api/shared/dtos/base-page-query.dto.ts`

Rules:
- API DTO property names MUST be `snake_case` (see 2.3.5a).
- API DTOs MUST NOT be imported by `libs/application/**` or `libs/persistence/**`.

#### B) Where API mappers live (MANDATORY)
API mappers shape the HTTP contract:
- request: API DTO -> Application input
- response: Application/Port DTO -> API response DTO (snake_case + page_meta)

**Preferred locations (match this repo):**

1) **Feature-level mappers (shared across roles)**
- `libs/api/controllers/<domain>/<feature>/mappers/`

2) **Role-scoped mappers (only for one role controller)**
- `libs/api/controllers/<domain>/<feature>/<role>/mappers/`

Naming:
- `*.api-mapper.ts`
- mapping functions: `toXxxApiDto`, `toXxxApiDtos`, `toXxxApiPageResult`

Rules:
- Mappers MUST be pure (no IO).
- Controllers MUST NOT return Port/Application objects directly without mapping.

#### C) Pagination query DTO rule (MANDATORY)
- Base pagination query DTO lives in API shared:
  - `libs/api/shared/dtos/base-page-query.dto.ts`
- Controllers MUST define feature-local query DTOs extending the base (even if empty), located in:
  - `libs/api/controllers/<domain>/<feature>/dtos/` OR
  - `libs/api/controllers/<domain>/<feature>/<role>/dtos/`
- Controllers MUST NOT use the base DTO directly as the endpoint query DTO.

#### D) Pagination response envelope (MANDATORY)
- Paginated endpoints MUST return `{ items, page_meta }` where `page_meta` uses snake_case fields:
  - `page`, `limit`, `total_items`, `total_pages`
- Controllers MUST NOT return shared `PageResult` / `PageMeta` directly.
Note (RECOMMENDED):
- Avoid naming collisions between API DTO classes and type-only DTOs. Prefer naming type-only pagination input as `PageQuery` / `BasePageQuery` to distinguish from API class DTOs.

---

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

### 2.3 Cross-cutting conventions (helpers / mappers / builders) (LOCKED)

Goal: keep feature code discoverable, avoid random `common/` folders, and keep dependency direction clean.

---

#### 2.3.1 General rules
- Prefer colocating helpers inside the feature that owns them.
- Do NOT create new global `common/` folders unless the code is truly reused by ≥2 features.
- Helpers MUST NOT import from persistence adapters or Prisma runtime.
- Feature barrel exports (`libs/application/features/<feature>/index.ts`) MUST export ONLY:
  - QueryServices / UseCases
  - (optional) feature-level DTOs/types that are application-owned
- ❌ Feature barrels MUST NOT re-export from `@tps/persistence/prisma` (no persistence leakage).

#### 2.3.2 Where to put helpers (by layer)

#### A) API layer helpers (libs/api)
Use for:
- request/response mapping specific to controllers
- swagger decorator wrappers
- pipe/validation helpers (DTO-centric)

Location:
- `libs/api/controllers/<feature>/<role>/mappers/`
- `libs/api/controllers/<feature>/<role>/dtos/`
- `libs/api/controllers/<feature>/<role>/helpers/`
- `libs/api/controllers/<domain>/<feature>/dtos|mappers|helpers/`
- `libs/api/shared/...`
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
#### 2.3.3 Standard feature folder layout (recommended)

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

#### 2.3.4 Mapper rules (strict)
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
#### 2.3.5 DTO placement rules (LOCKED)

Goal: prevent DTO sprawl and keep contracts stable.

###### A) API DTOs (HTTP-facing)
Use for:
- request validation (class-validator)
- swagger decorators / OpenAPI schema
- controller response shapes when they are strictly “web contract”

Location:
- `libs/api/controllers/<feature>/<role>/dtos/`

Rules:
- API DTOs may use class-validator + swagger decorators.
- API DTOs must NOT be imported by persistence.

##### B) Application feature DTOs (internal)
Use for:
- internal query/usecase input/output types that are feature-private
- intermediate data shapes (not a public contract)

Location:
- `libs/application/features/<feature>/dtos/` (optional)
- or colocated near the usecase/query that owns it

Rules:
- Feature DTOs are not “public contracts”.
- Do NOT export them from feature index.ts unless explicitly needed.

##### C) Contracts DTOs (port contracts)
Use for:
- port input/output types and shared DTOs used across layers
- stable shapes that application <-> persistence agree on

Location:
- `libs/application/contracts/<feature>/dtos/`

Rules:
- If a type appears in a Port interface, it MUST live in contracts.
- Contracts must not contain logic.
#### 2.3.5a JSON Response Naming & Pagination Envelope (SNAKE_CASE) (LOCKED)

Goal: keep HTTP response contracts consistent across the repo.

##### A) Response JSON naming (MANDATORY)

All HTTP JSON responses MUST use snake_case, including:

nested objects

arrays of objects

pagination metadata fields

Examples:

✅ voided_at, void_reason, created_at

❌ voidedAt, voidReason, createdAt

Notes:

This rule applies to response bodies. (Request/query naming is defined by feature DTOs and should prefer snake_case as well.)

##### B) Pagination envelope (MANDATORY)

Paginated endpoints MUST return pagination metadata under page_meta (snake_case).

Controllers MUST NOT return/ expose shared PageResult / PageMeta (camelCase) directly in HTTP responses.

Controllers MUST map pagination results to API response DTOs (snake_case) via API layer mappers/helpers.

Required page_meta fields (standard):

page_meta.page

page_meta.limit

page_meta.total_items

page_meta.total_pages

Optional fields (if used, MUST be snake_case and consistent):

page_meta.has_next

page_meta.has_prev

Forbidden in HTTP responses:

meta.totalItems, meta.totalPages, pageSize, totalItems, totalPages (camelCase anywhere)

##### C) Naming conversion responsibility (STRICT)

camelCase ↔ snake_case conversion for HTTP responses is the responsibility of the API layer only.

Do NOT convert naming in Application or Persistence layers.

Implementation guidance:

Prefer explicit API mappers (*.api-mapper.ts) and a shared pure helper for pagination envelope mapping.

Do NOT rely on global interceptors that silently transform keys (Swagger/schema drift risk).

#### 2.3.5b DTO Ownership & Validation Location (LOCKED)

Goal: prevent DTO sprawl, keep dependency direction clean, and avoid mixing HTTP concerns into contracts.

##### A) API DTOs are the HTTP contract (MANDATORY)

API DTOs are the only place allowed to contain:

class-validator decorators

class-transformer decorators

Swagger decorators (@ApiProperty, etc.)

Location:

libs/api/controllers/<feature>/<role>/dtos/

Rules:

API DTO property names MUST be snake_case (see 2.3.5a).

API DTOs must NOT be imported by Persistence.

##### B) Contracts DTOs are type-only (MANDATORY)

Contracts DTOs are stable shapes used in Port interfaces and across layers.

Location:

libs/application/contracts/<feature>/dtos/

Rules:

Contracts DTOs MUST be type-only (interfaces/types), or plain classes without:

class-validator

class-transformer

Swagger decorators

If a type appears in a Port interface, it MUST live in contracts.

Contracts MUST NOT contain business logic or helpers.

##### C) Application DTOs are internal shapes (OPTIONAL)

Application may define feature-private DTOs/types for internal orchestration.

Location:

libs/application/features/<feature>/dtos/ (optional) or colocated near the query/usecase.

Rules:

MUST NOT import Swagger or class-validator.

Keep feature DTOs private; do not export from feature index.ts unless explicitly needed.

##### D) Validation split (MANDATORY)

There are two kinds of validation:

Input validation (HTTP shape/range/format)

Lives in API DTOs (controller layer).

Examples: page >= 1, limit <= 100, string length, enum membership.

Business rule validation (domain invariants)

Lives in Application validators (*.validator.ts).

MUST be pure (no IO).

If DB checks are needed, Query/UseCase loads data via ports first, then passes data to validators.

##### E) Import rules (STRICT)

Application layer MUST NOT import:

Swagger decorators

class-validator

class-transformer

API DTOs from libs/api

Persistence layer MUST NOT import API DTOs.
#### 2.3.5c Type Safety at Boundaries (LOCKED)

Goal: prevent shape drift, hidden casts, and unsafe contracts across layers.

##### A) No explicit any at boundaries (MANDATORY)

Explicit any is forbidden in:

API Controllers (request/response types)

Application QueryService / UseCase public methods

Contracts Port interfaces and DTOs

Persistence adapters implementing ports

Use:

concrete DTOs/types in libs/api/.../dtos (HTTP) and libs/application/contracts/.../dtos (ports)

generics (T) when appropriate

##### B) unknown usage policy (MANDATORY)

unknown is allowed only when it is intentional:

catch (error: unknown) and error boundaries

untrusted/external payloads (must be parsed/validated before use)

snapshot / generic JSON payloads that are intentionally schema-agnostic

Forbidden:

returning unknown from Ports or QueryService public methods when the shape is known

Record<string, unknown> in contracts when a stable DTO exists (use a named type instead)

##### C) Cast policy (MANDATORY)

Avoid as unknown as X in Application and Persistence.

If a cast is unavoidable, it MUST be:

justified by the data source,

localized in a mapper/helper,

and not repeated across files.

Special rule for transactions:

Any cast from Tx → Prisma transaction client MUST live in a single persistence helper (see 7.1).

##### D) Definition of done (RECOMMENDED)

A refactor is considered clean when:

rg -n "any" libs/application libs/persistence returns 0 results

rg -n "as unknown as" libs/application libs/persistence returns only whitelist-allowed cases (catch/generic snapshot payload)

nx build api --skip-nx-cache passes

#### 2.3.6 Definitions: Mapper vs Builder vs Helper (LOCKED)

These terms are often mixed. Use this taxonomy to keep code discoverable.

##### Mapper
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

##### Builder
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

##### Helper (utility)
Purpose:
- small reusable pure utilities (formatting, checksum, stable stringify, etc.)

Characteristics:
- pure function
- no IO
- no feature business rules

Naming:
- `*.helper.ts` or `*.util.ts`

Place in feature `helpers/` or `libs/shared/utils/` if reused by ≥2 features.

##### Writer/Service (orchestrator)
Purpose:
- orchestration that involves transactions and persistence calls

Characteristics:
- may do IO (DB/tx)
- belongs in UseCase (application) or Adapter (persistence), depending on responsibility

Naming:
- `*.writer.ts`, `*.service.ts` (avoid calling these “helper”)
- If it opens a Prisma transaction or calls an adapter, it is NOT a helper; name it `*UseCase`, `*Writer`, or `*Service`.

#### 2.3.7 Mapper sample pattern (RECOMMENDED)

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

#### 2.3.8 Snapshot layout (RECOMMENDED)

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

---

#### 2.5.1 Boundary rules (NON-NEGOTIABLE)

libs/persistence ✅ depends on libs/application/contracts + libs/shared

libs/persistence ❌ must NOT import libs/application/features (usecases/queries)

libs/application ❌ must NOT import persistence runtime (PrismaService/PrismaModule/Adapters)

If Application needs DB behavior:
✅ define/extend a Port + Token in contracts, implement in persistence.

#### 2.5.2 Naming rules (LOCKED)

Adapters

Files/classes implementing a Port MUST end with Adapter

✅ ImportReceiptRepositoryAdapter

✅ ImportReceiptReleaseAdapter

✅ MoneyTransactionAdapter

Modules

Modules binding tokens MUST end with PersistenceModule

✅ ImportReceiptPersistenceModule

✅ MoneyTransactionPersistenceModule

Repository Adapter = aggregator

Repository adapter should be a thin entry point that delegates to sub-components.

Heavy logic belongs in focused sub-components: receipt.query, lot.scan, receipt.recompute, release.*.

#### 2.5.3 Token-only DI rule (CRITICAL)

Application injects by token, never concrete adapter classes.

Persistence follows the same rule for cross-feature usage:

❌ do NOT inject another feature’s adapter class

✅ inject the other feature’s port token and import its *PersistenceModule

Goal: agents must NOT “move files” just to make DI work.

#### 2.5.4 Cross-feature persistence dependency (LOCKED)

If feature A needs capability from feature B:

✅ import B’s *PersistenceModule

✅ inject B’s port token

❌ do not copy/move B’s adapter into A’s folder

❌ do not re-provide B’s adapter inside A’s module (avoid duplicate providers / drift)

#### 2.5.5 Transaction participation rule (LOCKED)

Transaction boundary belongs to UseCase (application).

Persistence adapters/sub-components may accept tx? to participate.

Persistence should not “own” business transaction boundaries.

#### 2.5.6 Recommended folder layout (RECOMMENDED)
##### A) Feature persistence (example: import-receipt)
libs/persistence/repositories/import-receipt/
  import-receipt.repository.adapter.ts     # implements ImportReceiptRepositoryPort (delegator only)
  import-receipt.persistence.module.ts     # binds IMPORT_RECEIPT_REPOSITORY_PORT

  receipt/
    receipt.query.ts                       # read: load receipt meta
    receipt.recompute.ts                   # recompute totals + status

  lot/
    lot.scan.ts                            # find/upsert lot helpers for scan flow

  release/
    import-receipt-release.adapter.ts      # money release / balance side-effects


Rule of thumb:

*.repository.adapter.ts = thin delegator

Move heavy SQL/branching into receipt/, lot/, release/.

##### B) Cross-cutting persistence (example: money-transaction)
libs/persistence/repositories/money-transaction/
  money-transaction.adapter.ts
  money-transaction.persistence.module.ts


Do not place cross-cutting adapters inside feature folders.

#### 2.5.7 Module composition rule (MANDATORY)

Feature persistence module should import other persistence modules, not re-provide their adapters.

Example:

ImportReceiptPersistenceModule

imports: MoneyTransactionPersistenceModule (if release needs it)

providers: ImportReceiptRepositoryAdapter + sub-components

exports: only the port token(s) it owns

#### 2.5.8 Split policy (RECOMMENDED)

Split when:

file > ~250–300 lines, or

a file mixes scan + recompute + release + inventory state transitions.

Split by concern:

receipt.query (read)

lot.scan (scan helpers)

receipt.recompute (totals/status)

release/* (money/balance)

optional inventory/* (item state transitions)

#### 2.5.9 “Provide only what you use” rule (MANDATORY)

If a module provides an adapter that is no longer injected, remove it.

Avoid provider bloat that misleads future refactors.
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
### 7.1 Transactions - Unit of Work Port (LOCKED)

Goal: keep transaction boundaries in Application while keeping Prisma details in Persistence.

#### A) Transaction boundary (MANDATORY)

Transaction boundary belongs in UseCase (Application layer).

Application MUST NOT call Prisma $transaction directly.

UseCases MUST open transactions via UNIT_OF_WORK_PORT.

#### B) UnitOfWork contract (MANDATORY)

Define UnitOfWorkPort and token UNIT_OF_WORK_PORT in libs/application/contracts/transaction/.

Define Tx as an opaque type in contracts (no Prisma imports).

Rules:

Contracts/Application MUST NOT import Prisma runtime/types.

Tx is passed through Application but only interpreted in Persistence.

#### C) Persistence implementation (MANDATORY)

Persistence implements UnitOfWorkPort using Prisma $transaction inside a dedicated adapter (e.g. PrismaUnitOfWorkAdapter).

All casts to Prisma transaction client MUST be centralized via a single helper (e.g. tx.helper.ts).
Forbidden: scattered as Prisma.TransactionClient across adapters.

#### D) Ports and tx propagation (MANDATORY)

Port methods that must run inside a transaction MUST accept tx: Tx (required).

Do NOT use tx?: unknown / manager?: unknown in port interfaces.

Read-only/query methods should not require tx unless strictly needed.

#### E) Migration rule (RECOMMENDED)

Migrate per feature/usecase:

Introduce UNIT_OF_WORK_PORT usage in the UseCase

Update affected port signatures to tx: Tx

Update persistence adapters to unwrap Tx via helper

Build must pass after each feature migration

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
For outbox events, business update and outbox insert MUST happen in the **same** transaction.

Correct (Application uses UnitOfWorkPort):
```ts
await unitOfWork.runInTransaction(async (tx) => {
  await updateBusiness(tx);
  await outboxWriter.write(tx, event);
});
```
Notes:

tx is an opaque Tx from contracts (no Prisma types in Application).

Persistence implements UnitOfWorkPort using Prisma $transaction.

Incorrect:

Insert outbox after commit.

Separate transactions.

- “Calling `prisma.$transaction` directly in Application is forbidden (see 7.1).”

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
