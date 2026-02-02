# agent-skills

A small collection of **agent skills / playbooks** to keep code consistent across projects.

## Skills included
- **NestJS**: Clean Architecture + Hexagonal (Ports & Adapters) + CQRS-lite, strict Nx module boundaries.
- **React**: UI consistency playbook (design tokens, reusable components, consistent styling patterns).

## Repo structure (quick)
- `skills/<name>/SKILL.md` is the entrypoint (mandatory).
- `skills/<name>/references/` contains optional topic guides used on-demand.
- `skills/<name>/assets/` and `skills/<name>/scripts/` are optional helpers/templates.

## How to use in a project (simple copy)
1) Copy the skill you need into your target project:

- NestJS:
```
cp -r skills/nestjs <project>/agent-docs/nestjs
```

- React:
```
cp -r skills/reactjs <project>/agent-docs/react
```

2) Add an `AGENTS.md` file at the root of the target project.

Point agents to the playbooks you copied (entrypoint `SKILL.md`).

Example references inside `AGENTS.md`:
- Backend only: `agent-docs/nestjs/SKILL.md`
- Frontend only: `agent-docs/react/SKILL.md`

Then instruct AI agents to read the relevant `SKILL.md` before coding.
