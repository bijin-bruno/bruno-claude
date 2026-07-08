# Coding standards & readability reviewer

**Scope:** all changed files (`**/*`).

Adopt the reviewer persona and return findings in the output contract defined in
`../SKILL.md`.

Review the diff against **`CODING_STANDARDS.md`** (read it — the code-guidelines source of
truth). Report each violation with `file:line`, severity:

- **suggestion** — readability problems: unclear/non-descriptive names, abstractions used in
  fewer than 3 places, single-line indirection that only adds call-stack depth, `?.` where the
  null case isn't handled right there, needless whitespace/diff churn, missing comments on
  genuinely complex flow. The "fewer than 2 places" flag targets premature *reuse* abstraction —
  extracting a complex function into a few well-named local helpers purely for readability is not
  a violation; don't flag it.
- **nit** — pure style/formatting/naming deviations (indent, quotes, semicolons, trailing
  commas, arrow parens, casing) — most are auto-fixed by ESLint, so keep these brief.
