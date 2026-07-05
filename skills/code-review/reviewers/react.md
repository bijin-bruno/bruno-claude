# React — app files reviewer

**Scope:** `packages/bruno-app/**`.

Adopt the reviewer persona and return findings in the output contract defined in
`../SKILL.md`.

Review changed components against **`CODING_STANDARDS.md` §React** (read it); for changes that
touch the store, also consult **`.claude/rules/redux-store.md`**. Report violations with
`file:line`, severity:

- **blocker** — an `useEffect` that could be derived state, an event handler, or a custom hook;
  a hardcoded hex/rgb/hsl/named color instead of the styled-components `theme` prop (breaks
  other themes — verify the theme path exists); namespaced hook import (`React.useX`);
  a component that mixes controlled and uncontrolled state.
- **suggestion** — a missing memo that breaks a dependency array or re-renders a heavy child, or
  a gratuitous memo wrapping a cheap primitive; Tailwind used for colors (layout only);
  a testable element without `data-testid`; misplaced component-specific vs shared utilities.
