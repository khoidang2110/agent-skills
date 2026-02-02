# Prisma Schema Folder Mode (LOCKED)

This repo uses Prisma schema folder mode (Prisma 6).

## Entry & source of truth
- Prisma CLI runs with: `--schema=libs/persistence/prisma/schema`
- Source of truth: `libs/persistence/prisma/schema/schema.prisma`
- `schema.prisma` contains generator + datasource ONLY.
- Do NOT define models/enums in `schema.prisma`.

## Model structure (strict)
Folder: `libs/persistence/prisma/schema/models/`
Rules:
- 1 file = 1 model.
- A model file may include exactly one model and optional enums used only by that model.

Forbidden:
- Multiple models in one file.
- A generic `models.prisma`.

## Enum strategy
- Shared enums (used by >=2 models): `libs/persistence/prisma/schema/enums/shared.prisma`
- Single-model enums: colocate inside the model file under `models/<model>.prisma`

Forbidden:
- Reintroducing a giant `enums.prisma`.
- Duplicating enums across files.
- Duplicating generator/datasource outside `schema.prisma`.

## Change workflow (mandatory)
When modifying schema:
1) Decide enum scope (shared vs colocate).
2) Update schema files.
3) Run:
   - `prisma format`
   - `prisma validate`
   - `prisma generate`
4) Run migration only if schema semantics changed.

Status:
- Schema split completed.
- Structure locked; do not restructure again unless a major business decision.
