# Feature File Layout (Builders / Mappers / DTOs)

## Principles
- Business mapping helpers (Builder/Mapper) live in application feature code:
  `libs/application/features/<feature>/...`
- Persistence contains only adapters, persistence modules, and Prisma DB access/error mapping.
- Contracts contain only ports, tokens, and shared DTO/types if needed.

## Naming conventions
- Pure mapping: `*.mapper.ts` or `*.builder.ts`
- Payload contract/types: `*.types.ts`
- Transactional/IO orchestration: `*.writer.ts` or `*.service.ts`

Avoid ambiguous names like `snapshot.helper.ts` when the file does DB/tx work.

## Canonical layout (snapshot-based feature example)
```
libs/application/features/sales-invoice/
  usecases/
    save-sales-invoice.usecase.ts
    confirm-sales-invoice.usecase.ts
    void-sales-invoice.usecase.ts
    revise-sales-invoice.usecase.ts
  queries/
    get-sales-invoice.query.ts
    list-sales-invoices.query.ts
  snapshot/
    sales-invoice.snapshot.builder.ts
    sales-invoice.snapshot.types.ts
    sales-invoice.snapshot.checksum.ts
  mappers/
    company.snapshot.mapper.ts
    warehouse.snapshot.mapper.ts
    client.snapshot.mapper.ts
  index.ts

libs/application/contracts/sales-invoice/
  ports/
    sales-invoice.query.port.ts
    sales-invoice.command.port.ts
    sales-invoice.snapshot.port.ts
  sales-invoice.tokens.ts
  dtos/

libs/persistence/repositories/sales-invoice/
  sales-invoice.adapter.ts
  snapshot/
    sales-invoice.snapshot.adapter.ts
    sales-invoice.snapshot.persistence.module.ts
  sales-invoice.persistence.module.ts

libs/shared/utils/checksum/
  stable-stringify.ts
  sha256.ts
```

Rules:
- `snapshot.builder.ts` must be pure: no Prisma imports, DB calls, or transactions.
- UseCases load data via ports, build `{ payload, checksum }`, then call snapshot port to persist.
- Persistence adapters only write to DB with provided payload/checksum/version/type.
- No snapshot payload-building logic inside persistence.

## DTO placement
- `libs/api/controllers/...`: Request/Response DTOs (validation + Swagger), map request -> usecase input.
- `libs/application/features/...`: internal usecase input types (optional).
- `libs/application/contracts/.../dtos`: only when multiple layers/features must share a stable type.

## Migration rule (helpers in persistence)
If snapshot builders/helpers exist in persistence:
- Move payload/checksum building to `libs/application/features/<feature>/snapshot/*.builder.ts`.
- Keep persistence as writer/adapter only.
- UseCases call builder, then snapshot port.
