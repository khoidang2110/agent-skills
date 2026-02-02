# agent-skills

A small collection of **agent skills / playbooks** to keep code consistent across projects.

## Skills included
- **NestJS**: Clean Architecture + Hexagonal (Ports & Adapters) + CQRS-lite, strict Nx module boundaries.
- **React**: UI consistency playbook (design tokens, reusable components, consistent styling patterns).

## How to use in a project (simple copy)
1) Copy the skill you need into your target project:
- NestJS:
cp -r skill/nestjs/SKILL.md <project>/agent-docs/nestjs/SKILL.md

- React:
cp -r skill/reactjs/SKILL.md <project>/agent-docs/react/SKILL.md


2) Add an AGENTS.md file at the root of the target project

Point agents to the playbooks you copied.

AGENTS.md template examples

Backend only (NestJS): agent-docs/nestjs/SKILL.md

Frontend only (React): agent-docs/react/SKILL.md

Then instruct AI agents to read the relevant SKILL.md before coding.