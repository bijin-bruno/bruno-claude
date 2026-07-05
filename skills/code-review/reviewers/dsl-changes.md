# On-disk DSL & serialization reviewer

**Scope:** `packages/bruno-app/**`, `packages/bruno-electron/**`, `packages/bruno-cli/**`,
`packages/bruno-lang/**`, `packages/bruno-filestore/**`, `packages/bruno-schema/**`,
`packages/bruno-schema-types/**`, `packages/bruno-converters/**`, `packages/bruno-toml/**`.
The format packages own serialization, but DSL objects are assembled in `bruno-app` (Redux)
and `bruno-electron` (IPC), and `bruno-cli` reads/writes these files directly — a shape change
made there won't be caught by the serialization layer, so they're all in scope.

Adopt the reviewer persona and return findings in the output contract defined in
`../SKILL.md`.

Review the diff against **`.claude/rules/dsl-changes.md`** (read it) — Bruno persists
collections/requests/environments/config as `.bru` and `.yml` files read across app versions, so
backward compatibility is the priority. Report violations with `file:line`, severity:

- **blocker** — a breaking field change (rename, remove, retype, repurpose, or changed
  default/semantics); a new field that isn't optional with a safe default; format drift between
  `bru` and `yml` (or between parse and stringify); a lossy round-trip (`stringify` drops
  unknown fields, or `parse(stringify(x)) !== x`); a shape change with no read-time compat shim;
  a cryptic, abbreviated, or inconsistent property name (names are permanent and user-facing).
- **suggestion** — missing round-trip test in both formats or missing old-format golden fixture;
  a name/structure that won't scale (positional vs keyed, widening union).
- Also push back on any **unnecessary** DSL change that could be solved in memory / UI without
  altering the serialized shape.
