# Agent Rules – React UI

Before implementing or modifying ANY UI:
1) Read docs/agent/SKILL.md
2) Reuse existing TailAdmin components and tokens
3) Do not create new components unless reuse is impossible

---
name: react-tailadmin-ui-consistency
description: Use this skill FIRST for any React UI or styling work. Enforces consistent TailAdmin design tokens, Tailwind utility grammar, reusable components, and dark mode parity.
---

# React UI Consistency Playbook – TailAdmin (Tailwind)

This skill enforces **UI consistency** across the React codebase.
Before implementing or modifying ANY UI (layout, styling, components), you MUST follow this file.

The goal is simple:
> Any new UI must look like it already belongs to the project.

---

## Source of Truth (READ FIRST)

Before writing UI code, review these files:

- `src/styles/tailwind.css`  
  (design tokens: colors, typography, spacing, shadows, dark mode)
- `tailwind.config.js`  
  (theme extensions and palette)
- Existing UI primitives (reuse first):
  - `src/components/button.tsx`
  - `src/components/text-field/input-field.tsx`
  - `src/components/text-field/label-field.tsx`
  - `src/components/text-field/checkbox-field.tsx`
- Similar screens under:
  - `src/pages/**` (copy layout and style patterns)

If these files do not exist yet, create minimal versions before implementing UI.

---

## Non-Negotiable Rules

### 1) Reuse existing components first
- Always prefer existing primitives (`Button`, `Input`, `Label`, `CheckboxField`, etc.).
- Do NOT restyle native HTML elements if a shared component already exists.
- If an existing component covers **≥ 80%** of the requirement, reuse or extend it.

---

### 2) Creating new components (ALLOWED WITH CONDITIONS)

Creating a new component is allowed ONLY when **all conditions below are met**:

1) No existing component or example covers at least 80% of the required behavior or layout.
2) The component is reusable (intended for multiple screens) OR represents a new UI primitive
   (e.g. `Badge`, `EmptyState`, `StatusPill`, `StatCard`).
3) The component follows the existing TailAdmin style grammar:
   - uses existing design tokens and utility patterns
   - supports dark mode when applicable
   - avoids hardcoded colors, spacing, or typography
   - uses consistent props naming (`variant`, `size`, `tone`, etc.)

If a similar component exists, **extend or compose it instead of creating a new one**.

One-off components with no reuse intent are discouraged.

---

### 3) Use tokens and established utility patterns
- Prefer existing TailAdmin patterns:
  - spacing: `px-4`, `py-2.5`, `h-11`, `gap-4`
  - radius: `rounded-lg`
  - shadows: `shadow-theme-xs`
  - typography: `text-sm`, `text-title-sm`
- Prefer token-based colors:
  - brand: `bg-brand-500`, `hover:bg-brand-600`
  - neutral: `text-gray-800`, `border-gray-300`
  - dark mode: `dark:bg-gray-900`, `dark:text-white/90`
- Avoid random values such as `mt-[7px]`, `text-[13px]` unless strictly necessary.

---

### 4) Dark mode is mandatory
- Any new UI must include `dark:` equivalents.
- Follow existing conventions:
  - `dark:bg-gray-900`
  - `dark:text-white/90`
  - `dark:border-gray-700`
  - `dark:hover:bg-white/[0.03]`

Missing dark mode styles is considered incomplete work.

---

### 5) Interaction states are required
Any interactive UI must include:
- hover state
- focus state (ring or border)
- disabled state when applicable

Follow existing `Button` and `Input` behavior for consistency.

---

## Workflow (FOLLOW IN ORDER)

1) Find a similar existing UI  
   - Look for a comparable screen in `src/pages/**`.
   - Copy layout and spacing patterns instead of inventing new ones.

2) Compose from primitives  
   - Build UI using shared components first.
   - Only create new components if reuse is not possible (see rules above).

3) Apply styling using TailAdmin grammar  
   Common patterns to reuse:
   - containers: `flex flex-col gap-*`, `max-w-* mx-auto`
   - cards: `rounded-lg bg-white dark:bg-gray-900 shadow-theme-xs`
   - buttons: use `Button` variants instead of custom class strings

4) Ensure dark mode parity  
   - Every background, border, and text color must have a dark equivalent.

5) Run a consistency check  
   - Compare spacing, typography, and colors with similar screens.
   - Verify hover/focus/disabled states.
   - Check responsiveness at common breakpoints.

---

## Definition of Done (UI)

A UI change is DONE only when:
- Existing primitives are reused or extended where possible
- New components (if any) meet the creation conditions
- No hardcoded colors or random spacing values are introduced
- Dark mode styles are present
- Hover, focus, and disabled states are implemented
- UI matches spacing and typography of similar screens
- Code is reusable and not a one-off hack

---

## Common Pitfalls (AVOID)

- Creating new components without searching existing ones
- Styling raw `<button>` or `<input>` instead of using shared primitives
- Missing `dark:` styles
- Mixing multiple styling systems in one component
- Hardcoding hex colors or arbitrary pixel values
- Adding ad-hoc variants instead of extending existing components

---

## Final Rule

If you are unsure:
> Copy the closest existing screen or component and adapt it.  
> Do not invent new UI patterns without clear justification.
