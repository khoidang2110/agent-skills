# agent-skills

A small collection of **agent skills / playbooks** to keep code consistent across projects.

## Skills included
- **NestJS**: Clean Architecture + Hexagonal (Ports & Adapters) + CQRS-lite, strict Nx module boundaries.
- **React**: UI consistency playbook (design tokens, reusable components, consistent styling patterns).

## How to use in a project (simple copy)
1) Copy the skill you need into your target project:
- NestJS:
cp -r skill/nestjs/SKILL.md <project>/docs/agent/nestjs/SKILL.md

- React:
cp -r skill/reactjs/SKILL.md <project>/docs/agent/react/SKILL.md


2) Add an `AGENTS.md` file at the **root** of the target project and point agents to the playbooks.

### AGENTS.md template (recommended)
```md
# Agent Rules

Before implementing or modifying ANY backend API:
- Read: docs/agent/nestjs/SKILL.md

Before implementing or modifying ANY React UI/styles:
- Read: docs/agent/react/SKILL.md


## Quick start

Copy only the skill you need into your project:

NestJS:
docs/agent/nestjs/SKILL.md
root/AGENTS.md


React:
docs/agent/react/SKILL.md
root/AGENTS.md


Then instruct AI agents to read the relevant SKILL.md before coding.