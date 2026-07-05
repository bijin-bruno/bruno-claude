---
paths:
  - "packages/**/*"
  - "tests/**/*"
  - "scripts/**/*"
---

# AI Code Hygiene

Code and comments must read as a natural, permanent part of the project — never as artifacts of the
task or session that produced them. Applies to every change. See also `CODING_STANDARDS.md`
§Readability.

## Comments

- **No situational or prompt-driven comments.** A comment must not reference the change, the task, or
  the moment it was written. Drop `// added to fix ...`, `// as requested`, `// new logic for X`,
  `// updated to handle ...`, `// per review`. If the reason genuinely matters, state it as a
  timeless fact about the code (or link the issue/PR) — not as "what I just did".
- **No obvious comments.** Don't restate what the code already says. `// loop over items` above a
  loop, `// set the name` above `obj.name = name`, `// return the result` — these add nothing. If the
  code is self-explanatory, leave it uncommented.
- **Comment the why, not the what.** Reserve comments for what the code can't show: non-obvious
  rationale, invariants, edge cases, a workaround and the constraint forcing it, units, or a pointer
  to a spec/issue. These stay useful long after the change lands.
- **No scaffolding or narration.** No `// ... existing code ...`, no placeholder or TODO-for-me
  notes, no changelog or step-by-step narration in comments, no commented-out code left behind.

## Beyond comments

- Don't add speculative abstractions, options, or configuration "for later" — build only what the
  change needs (the 3-use rule in `CODING_STANDARDS.md`).
- Match the surrounding code's style and naming so a change is indistinguishable from the existing
  codebase, not visibly bolted on.
- Keep diffs minimal — no unrelated reformatting or whitespace churn.
