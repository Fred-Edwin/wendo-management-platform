## What changed
<!-- One or two sentences. Link the issue: Closes #123 -->

## Evidence
<!-- Paste the output of `pnpm evidence`. A PR without it will not be reviewed. -->

```
```

## Screens (UI changes only)
<!-- Paste the `pnpm see` outline, or before and after screenshots. -->

## Rules touched
<!-- Rule numbers from docs/domain-model.md section 6, and the test that guards each. "none" if none. -->

## Remaining risks and what was not verified
<!-- Be specific. "None" is rarely true. -->

## Checklist
- [ ] `pnpm check` exits 0 (`--db` if data changed)
- [ ] Migrations only add things
- [ ] No edits to `core/` mixed into a feature change
- [ ] New or changed behaviour has a test that fails without the change
- [ ] No new packages (or the owner approved them)
