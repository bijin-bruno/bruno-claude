# bruno-claude

A drop-in [Claude Code](https://docs.claude.com/en/docs/claude-code) configuration for
[Bruno](https://github.com/usebruno/bruno) — the project context, architecture notes, coding
rules, and review/test skills the team has tuned while working on Bruno. Clone it into your
Bruno checkout and Claude picks it up automatically.

## What's inside

| Path | What it is |
|------|------------|
| `CLAUDE.md` | Project overview — commands, monorepo layout, key architecture, coding standards. Auto-loaded every session. |
| `rules/` | Deep, path-scoped guidance read when you touch the relevant area: `architecture`, `redux-store`, `electron-ipc`, `dsl-changes` (on-disk `.bru`/`.yml` format), `cross-platform`, `testing`, `ai-hygiene`. |
| `skills/code-review/` | `/code-review` — reviews the current branch/PR via focused lenses (correctness, security, DSL, React, cross-platform, tests) in parallel. Takes base pointers from `.coderabbit.yaml`, then goes deeper with Bruno-specific checks. |
| `skills/write-e2e-test/` | `/write-e2e-test` — writes a Playwright E2E test following Bruno's fixtures and conventions. |
| `settings.json` | Shared project settings (permissions, env), checked in for the whole team. Empty by default. |
| `settings.local.json` | Your machine-specific overrides. Gitignored — never committed. |

## Requirements

- [Claude Code](https://docs.claude.com/en/docs/claude-code) installed (`npm i -g @anthropic-ai/claude-code`).
- A local checkout of Bruno (or a Bruno fork).

## Install

From the root of your Bruno checkout, clone this repo into a `.claude` folder:

```bash
cd path/to/bruno

git clone https://github.com/bijin-bruno/bruno-claude.git .claude
```

That's it. Start Claude Code (`claude`) from the Bruno root and everything loads on its own:

- **`CLAUDE.md`** and any **rule** without a `paths:` filter load at launch.
- **Path-scoped rules** attach automatically when Claude reads a matching file (e.g. the
  `dsl-changes` rule kicks in when you edit `bruno-filestore`).
- **Skills** are discovered from `.claude/skills/` — type `/code-review` or `/write-e2e-test`.
- **`settings.json`** is read (and hot-reloaded) automatically.

No root `CLAUDE.md` or manual registration is needed — `.claude/CLAUDE.md` is a first-class
config location.

If you want personal, uncommitted instructions on top of this shared config, you *can* add a
root `CLAUDE.md` in your Bruno checkout — Claude loads it alongside `.claude/CLAUDE.md`. Not
recommended (keep shared guidance here so everyone benefits), but useful for one-off local tweaks.

## Keep your Bruno repo clean

`.claude` is its own git repo, so it shouldn't be committed into Bruno. Good news: Bruno's root
`.gitignore` already ignores `.claude`, so cloning here won't touch Bruno's tracked files —
nothing to do. (If you're on a fork that dropped that entry, add it back with
`echo ".claude/" >> .git/info/exclude`.)

Your own machine-specific overrides go in `.claude/settings.local.json`, which is already
gitignored here — put personal permissions or env there, not in `settings.json`.

## Everyday use

```bash
/code-review            # review the current branch against main before pushing
/write-e2e-test         # scaffold a Playwright spec under tests/
```

Outside the skills, just work normally — Claude consults the rules on demand. Ask it to
"review my changes", "add an e2e test for X", or "explain how requests are persisted" and it
already has the project context.

## Updating

Pull the latest guidance any time:

```bash
cd path/to/bruno/.claude
git pull
```

## Contributing

Improvements to the rules and skills are welcome. A few conventions the config follows:

- **Rules are the source of truth; reviewer lenses are thin pointers.** Put the "why/how" in a
  `rules/*.md` file; a reviewer lens carries only its severity mapping and a pointer to the rule.
- **Don't hardcode volatile catalogs.** Describe the category and tell the agent to read the
  codebase for the current set (slices, helpers, handlers) rather than listing items that drift.
- **Verify against real code.** Every rule/example should reflect the actual repo, not assumptions.

## Where things go

- **`rules/*.md`** — engineering rules, scoped by their `paths:` frontmatter (not by file
  location). Cross-cutting rules (`conventions`, `ai-hygiene`, `architecture`, `cross-platform`)
  stay at the top level; domain rules cover **app** (`redux-store`), **electron** (`electron-ipc`),
  **packages / DSL** (`dsl-changes`), and **testing** (`testing`). Add a new rule as a flat file
  with correct `paths:`; only split a domain into a subfolder once it has several rules (and first
  confirm the rule loader recurses into subdirectories).
- **`skills/<name>/`** — a `SKILL.md` orchestrator plus its own support files (e.g. `reviewers/`,
  and future `scripts/`). The shared review persona and output shape live once in
  `skills/code-review/reviewers/_contract.md`; reviewer lenses and `SKILL.md` point to it.
- **`settings.json`** — team-wide permissions/hooks (committed). **`settings.local.json`** —
  personal, machine-specific overrides (gitignored).
- **Source of truth:** coding standards → `CODING_STANDARDS.md`; architecture/behavior →
  `rules/*`; e2e → `docs/playwright-testing-guide.md`. `.coderabbit.yaml` mirrors these for CI.

## License

MIT — see [`LICENSE`](LICENSE).
