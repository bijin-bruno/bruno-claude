---
paths:
  - "packages/**/*"
  - "tests/**/*"
  - "scripts/**/*"
---

# Coding Conventions & Readability

`CODING_STANDARDS.md` is the source of truth for Bruno's coding standards — read it. This file is
the readability lens over those standards: the judgment calls a linter can't make. `ai-hygiene.md`
covers comment quality and diff cleanliness, which apply here too.

## Style & formatting

The mechanical style — 2-space indent, single quotes (double for JSX/TSX attributes), semicolons,
no trailing commas, parenthesized arrow params, spacing around arrows, no space before call parens
— is enforced by ESLint and auto-fixed by `npm run lint:fix`. These deviations are real but
low-value to catch by hand: note them briefly rather than dwelling. Naming and casing that ESLint
can't mechanically repair still warrant attention.

## Readability

- **Descriptive names.** Functions and variables carry concise, descriptive names; an unclear or
  misleading name is worth raising even when the code is otherwise correct.
- **Premature abstraction.** Don't extract a shared abstraction until the same code appears in more
  than three places — an abstraction pulled out for one or two call sites adds indirection without
  earning reuse. This targets *reuse* abstraction only: breaking a long, complex function into a
  few well-named local helpers purely for readability is encouraged, not a violation.
- **Single-line indirection.** A one-line function that only forwards to another — adding a stack
  frame without adding meaning — should be inlined.
- **Optional chaining.** `?.` belongs only where the null case is handled right there (a fallback,
  early return, or guard). Used elsewhere it hides whether a value can genuinely be null and works
  against TypeScript's guarantees; fix the type or narrow first.
- **Comments explain the why.** Genuinely complex flow deserves a comment covering the rationale an
  obvious reading can't; self-explanatory code does not. See `ai-hygiene.md` for the full bar.
- **Functional, but readable.** Prefer obvious, linear pipelines over deep functional machinery
  (ADTs, monads) — the code should stay easy for any contributor to follow and extend.
