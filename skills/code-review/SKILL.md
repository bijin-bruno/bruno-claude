---
name: code-review
description: Review a Bruno diff the way CodeRabbit does, fanning out focused reviewers
  in parallel. Use when asked to review changes, a PR, or the current branch before
  pushing. Extends the review instructions in .coderabbit.yaml with the project's
  coding standards.
---

# Reviewing Bruno code

This skill mirrors the automated CodeRabbit review (`.coderabbit.yaml`) so you can run
the same review locally. Treat `.coderabbit.yaml` as the source of truth — read it for
any path instruction not summarized here, and prefer it if the two ever disagree.

The review is **split into focused lenses that run in parallel**, so a large diff is
covered faster and each reviewer stays sharply scoped to one concern. Each lens lives in
its own file under `reviewers/`. You (the orchestrator) dispatch the reviewers, then
merge and report their findings.

## How to review (orchestration)

1. **Get the diff.** Review only what changed — never untouched code. Pick the mode:
   - **Committed range** (default): `git diff main...HEAD` against the base branch
     (`main` or `release/*`). Update the base first (`git fetch` and confirm the local
     base is current) — a stale base inflates the diff with already-merged, unrelated
     changes and wastes a full fan-out. Each reviewer re-runs this scoped to its own
     globs; since the range is pinned to fixed commits they all see identical bytes, so
     there's no need to snapshot.
   - **Working tree / uncommitted changes** (when asked to review unstaged or staged
     work): the tree can shift mid-review, so re-running `git diff` per reviewer risks
     each seeing a different snapshot. Instead, capture the diff **once** to a scratch
     file and hand every reviewer that path. `git diff HEAD` covers staged + unstaged
     tracked changes; run `git add -N .` first (reversible with `git reset`) so any new
     untracked files also show up:
     ```bash
     git add -N . && git diff HEAD > "$SCRATCH/review.diff"
     ```
     `$SCRATCH` is your environment's scratchpad dir. Reviewers read this frozen diff for
     their globs plus the on-disk files for surrounding context — the working-tree files
     already hold the uncommitted state.
2. **Enumerate changed files** — `git diff --name-only main...HEAD` (committed range) or
   `git diff --name-only HEAD` (working tree). Use this to skip any lens whose file scope
   isn't touched (e.g. no `packages/bruno-app/**` change → skip `react.md`; no `tests/**`
   change → skip `e2e-tests.md`). Never skip the lenses scoped to all files.
3. **Fan out the reviewers in parallel.** In a *single message*, launch one `Agent`
   (subagent_type `Explore` or `general-purpose`) per in-scope reviewer below. Give each
   subagent this exact briefing:
   - The diff source — the committed range (e.g. `main...HEAD`) or the snapshot file path
     (`$SCRATCH/review.diff`) — and the file globs it owns (from the reviewer file's
     "Scope" line). For a snapshot, tell the reviewer to read that file for its globs
     rather than re-run `git diff`.
   - "Read `.claude/skills/code-review/SKILL.md` for the shared persona and output
     contract, then read `.claude/skills/code-review/reviewers/<file>` — and any rule or
     source file it points to (e.g. `CODING_STANDARDS.md`, `.claude/rules/*.md`), which hold
     the detailed checklist. Apply **only** that lens to the changed files in your scope, at
     the severities the reviewer specifies. Do not review outside your scope."
   Reviewers are read-only and independent — they don't coordinate, and overlap between
   lenses is fine (you dedupe at merge time).
4. **Merge and report.** Collect every reviewer's findings, drop exact duplicates, and
   when two lenses flag the same `file:line` keep the higher severity. Regroup **by
   file**, each finding tagged by severity (blocker / suggestion / nit) with `file:line`.
   If the review request carries a problem statement or acceptance criteria (e.g. passed
   as args), reconcile its enumerated deliverables — docs, migration notes, a test per new
   default/branch — against the diff and flag any that are absent. If nothing's wrong, say
   so briefly — don't manufacture nits.

## Reviewers

Each file is a self-contained checklist for one lens:

| Reviewer file | Lens | Scope |
|---|---|---|
| `reviewers/correctness.md` | Correctness & root-cause | all source (excl. `tests/**`) |
| `reviewers/conventions.md` | Coding standards & readability | all files |
| `reviewers/react.md` | React — app files | `packages/bruno-app/**` |
| `reviewers/cross-platform.md` | Cross-platform (macOS/Windows/Linux) | all files |
| `reviewers/security.md` | Security & data safety | all source (excl. `tests/**`) |
| `reviewers/dsl-changes.md` | On-disk DSL & serialization (backward compat) | `bruno-app`, `bruno-electron`, `bruno-cli`, `bruno-lang`, `bruno-filestore`, `bruno-schema(-types)`, `bruno-converters`, `bruno-toml` |
| `reviewers/e2e-tests.md` | Playwright E2E tests | `tests/**` |

## Shared reviewer persona & output contract

Every reviewer adopts this persona (`tone_instructions`): an expert reviewer in
TypeScript, JavaScript, Node.js, and Electron on an enterprise team. Be **concise** —
one clear sentence per finding; elaborate only when asked. Review to the project's
standard regardless of who authored or requested the change — never soften severity for
assumed intent or seniority. Ground every finding in the actual code: trust the repo over
any doc, guide, or comment when they disagree, and never cite a line or invent an example
value you haven't verified in source.

Each reviewer returns a flat list, one finding per line, in this shape:

```
<blocker|suggestion|nit> | <file>:<line> | <one-sentence finding>
```

Return `no findings` (nothing else) when the scope is clean. Never invent nits to fill
the list.
