# Workflow: Adding a New API (Step-by-step)

## Step A — Define Port + Token (Contracts)
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

## Step B — QueryService / UseCase (Application)
Location:
- `libs/application/features/<feature>/queries/`
- `libs/application/features/<feature>/usecases/`

Naming:
- Query: `GetXxxQueryService`, `ListXxxQueryService`
- UseCase: `CreateXxxUseCase`, `UpdateXxxUseCase`, `AdjustXxxUseCase`

Pattern:
- `@Inject(TOKEN) private readonly port: XxxPort;`
- Error handling: `throw new DomainError(ErrorCode.X, 'message');`
- Required barrel export: `libs/application/features/<feature>/index.ts`

## Step C — Adapter + PersistenceModule (Persistence)
Adapter:
- `libs/persistence/repositories/<feature>/<feature>.adapter.ts`

Module:
- `libs/persistence/repositories/<feature>/<feature>.persistence.module.ts`

Binding pattern:
```
providers: [
  Adapter,
  { provide: XXX_QUERY_PORT, useExisting: Adapter },
  { provide: XXX_USECASE_PORT, useExisting: Adapter },
],
exports: [XXX_QUERY_PORT, XXX_USECASE_PORT],
```

Rules:
- One adapter may implement multiple ports (use `useExisting`).
- One port token has exactly one canonical implementation.

## Step D — Controller + ApiModule (API)
Controller location:
- `libs/api/controllers/<feature>/<role>/`

Role conventions:
- `user/`, `admin/`, `backoffice/`

Controller rules:
- Guards + roles required.
- Use `@CurrentUser()` for identity.
- Do NOT accept `{userId}` from params for user APIs.

Api module:
- `libs/api/controllers/<feature>/<feature>.module.ts`

Imports:
- `<Feature>PersistenceModule`

Providers:
- QueryService / UseCase classes (provided directly in ApiModule)

## Naming & Route Conventions
Naming:
- Query: `GetXxxQueryService`
- UseCase: `CreateXxxUseCase`, `AdjustXxxUseCase`
- Token: `<FEATURE>_QUERY_PORT`
- Module: `<Feature>PersistenceModule`

Routes:
- User: `/api/v2/user/...`
- Admin: `/api/v2/admin/...`
- Backoffice: `/api/v2/backoffice/...`

Rules:
- No `/me` for multi-actor resources; `/me` only for identity/profile.
