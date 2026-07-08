---
paths:
  - "packages/bruno-app/**"
  - "packages/bruno-electron/**"
  - "packages/bruno-cli/**"
  - "packages/bruno-lang/**"
  - "packages/bruno-filestore/**"
  - "packages/bruno-schema/**"
  - "packages/bruno-schema-types/**"
  - "packages/bruno-converters/**"
  - "packages/bruno-toml/**"
---

# On-Disk DSL & Serialization Changes

Bruno persists everything users create — collections, requests, environments, folders, and
config — as plain-text files on the user's disk and in their Git repos. Two on-disk formats
exist:

- **`.bru`** — the Bru DSL, parsed/serialized by the grammars in `packages/bruno-lang`
  (`v1` and `v2`) via the `packages/bruno-filestore/src/formats/bru` layer.
- **`.yml`** — OpenCollection YAML (the current `DEFAULT_COLLECTION_FORMAT`), handled by
  `packages/bruno-filestore/src/formats/yml`.
- Plus `bruno.json` (collection config), `preferences.json`, `*.env`, and TOML
  (`packages/bruno-toml`).

These files outlive the version that wrote them: an older Bruno reads files written by a
newer one, teammates on different versions share the same repo, and users hand-commit these
files. **A change to the on-disk shape is a change to a public contract.** Get it wrong and
you break parsing, corrupt collections, or silently drop user data — on collections you
can't reach to fix.

The in-memory objects that become these files are often assembled far from the serialization
layer — in `bruno-electron` (IPC handlers) or `bruno-app` (Redux) — and handed to
`bruno-filestore`, which can't tell that a field's shape changed upstream. So a DSL-affecting
change can originate in *any* package on the data path, not just the format packages. Treat
this rule as in scope for changes anywhere the collection/request/environment/config object
is built, mutated, or serialized — the serialization packages plus `bruno-app`,
`bruno-electron`, and `bruno-cli`, where these objects are assembled or read/written.

## Rules

**Avoid unnecessary DSL changes.** First ask whether the need can be met in memory, in the
UI, or as a derived value without touching the serialized shape at all — most enhancements
can. Only change the on-disk format when the data genuinely must persist.

**Additive and optional only.** A new field must be optional with a safe default, so files
without it still parse and behave. Never rename, remove, repurpose, or retype an existing
field, and never change its semantics — old files and old app versions depend on the
current meaning.

**Round-trip must be lossless.** `parse(stringify(x))` must equal `x`, and `stringify` must
never drop fields it doesn't recognize. A file written by a newer version, opened and
re-saved by an older one, must not lose the newer fields.

**Prove escaping on the written bytes, not in memory.** When a value is embedded in a
delimited block (`.bru` multiline `'''…'''`, annotation args), an in-memory `escape`/`unescape`
round-trip can pass while the on-disk file still corrupts — the parser terminates at the first
re-introduced delimiter. Test `parse(read(serialize(x)))` on the actual bytes, and fuzz across
quote/backslash runs. Escape at single-character granularity: escaping a multi-character
delimiter by splitting on the whole delimiter (`split("'''")`) leaves fragments that recombine
on runs of length ≥ the delimiter; escape the backslash first, then every single delimiter char.

**New syntax breaks older parsers — not just older fields.** The additive-and-optional rule
protects field-level compat, but adding new `.bru` *syntax* (a new escape form, block delimiter,
or annotation grammar) makes files written by the new version fail to *parse* in older versions,
which have no handling for it. This is a forward-compat break beyond field loss; it needs an
explicit decision and a compatibility path before shipping.

**Descriptions are string-only internally.** Every `description` field (headers, params,
assertions, variables, body entries, ...) is stored as a plain `string`. Parsers accept a bare
string *or* a legacy `{ content }` object on read and normalize to the string; writers emit a
string. This read-accepts-object / write-emits-string asymmetry is an established convention
across all description fields — it is not a lossy round-trip. New description-like fields should
follow the same string-only shape.

**Keep both formats in lockstep.** Any field added or changed must be handled in *both*
`bru` and `yml` — parse and stringify on each side — or the same collection behaves
differently depending on its format. Pay attention to the *unset* case: a field that
defaults or persists one way in `.bru` and another in `.yml` (e.g. one path returns `''`,
the other `'1'`) is a divergence bug. The default app-data workspace and custom-filesystem
workspaces are a second such twin — a value that persists in one but is silently dropped in
the other is the same class of bug. Update every layer the field touches: `bruno-schema-types`
(types), `bruno-schema` (Yup validation), `bruno-filestore` (both formats), `bruno-converters`
(import/export), and the grammar in `bruno-lang` if `.bru` syntax itself changes.

**When a shape must change, migrate on read — never make users edit files.** Add a
read-time compatibility shim that upgrades the old shape as it is parsed, following the
existing patterns: `ensureAuthV3Rc1BackwardsCompatibility` in `formats/yml/parseItem.ts`,
and the pre-v3 status/statusText swap in `formats/bru/index.ts`. Gate any eventual removal
behind a major version bump and leave a dated `TODO(remove after vN)`.

**Design for scale and consistency.** New keys follow the naming and nesting of the existing
`meta`/section blocks (`meta.name`, `meta.seq`, `meta.type`, `meta.tags`, ...). Prefer
structures that extend cleanly — a keyed list over a positional one, an object over a
widening union — and match how neighbouring fields are already modelled.

**Name properties like they're permanent — because they are.** A DSL key is written once and
effectively forever: renaming it breaks every existing file and forces a compat shim. These
names are also user-facing — people read and hand-edit `.bru`/`.yml` — and are read by AI
agents reasoning over collections. Choose clear, descriptive, fully spelled-out names, with
no cryptic abbreviations or ambiguity, consistent with existing keys, and get it right the
first time. A property name deserves stricter scrutiny than an ordinary variable.

**Test the contract.** Add round-trip tests (parse → stringify → parse) for the new field in
*both* formats, plus a golden fixture of the *old* format that must still parse unchanged.
The `*.spec.ts` files next to each format serializer in `bruno-filestore` are the pattern.

## Before changing the DSL — checklist

- [ ] Change is genuinely necessary (can't be solved in memory / UI)
- [ ] New field is optional with a safe default; nothing renamed, removed, or retyped
- [ ] Handled in both `bru` and `yml`, parse **and** stringify
- [ ] Types (`bruno-schema-types`), Yup schema (`bruno-schema`), and converters updated
- [ ] Old files still parse — read-time compat shim added if the shape changed
- [ ] No new `.bru` syntax that older parsers can't read (or a forward-compat path decided)
- [ ] Any new escaping proven via on-disk reparse + fuzz, escaping single chars not the delimiter
- [ ] `bru`/`yml` (and default vs custom workspace) behave identically, including the unset case
- [ ] Round-trip + old-format fixture tests added
- [ ] Naming/structure consistent with existing blocks and scalable
