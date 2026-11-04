# Wendo Management Platform

Restaurant management for Wendo Coffee and Bistro (3 branches, up to 10). Built and maintained by coding agents. Floor operations (orders, kitchen screens, payments, receipts) must never stop, even when the internet fails.

## Read first (only what your task needs)
- `docs/agent-experience.md`: how we work (checks, memory, tests, file names, rules and their tools). Read this first.
- `docs/domain-model.md`: the glossary (use these words in code) and the 20 rules in section 6.
- `docs/feature-shape.md`: what one feature folder looks like.
- `docs/architecture.md`: the design. `docs/decision-log.md` wins where they differ (section J, then K).
- `docs/product-brief.md`: what the product is.

## Commands
- `pnpm where [word]`: find any command, query, screen, event, table or permission. Do not grep the repository to find code.
- `pnpm check [feature] [--db]`: type check, lint, rule checks and tests. No name = the features you changed. Exit 0 = verified. Target 10 s, with `--db` 30 s.
- `pnpm see <route> [--as <role>]`: a text outline of a screen.
- `pnpm evidence`: prints the evidence block for the PR.
- `pnpm new feature|command|query|screen|migration ...`: always create files with the generator.
- `pnpm agent:init <name>`: your own worktree, port, URL and database.
- `pnpm dev`: the app with fake data and a login for every role.

These tools are specified, and some do not exist yet. If a command is missing, say so; do not invent a substitute.

## Rules
1. Work in your own worktree (`pnpm agent:init`). One task, one branch, one pull request.
2. Light mode by default: edit and run `pnpm check`. Start the dev server and a browser only for a screen change. Limits: 3 full agents or 6 light agents on this laptop.
3. Never run full test suites, production builds, Playwright suites or Lighthouse on the laptop. CI runs them. Read results with `gh pr checks`.
4. File names have fixed endings: `.command.ts`, `.command.server.ts`, `.query.ts`, `.screen.tsx`, `.event.ts`, `.handler.ts`, `.print.ts`, `.job.ts`. Server files (`.server.ts`) are never imported by client code.
5. A feature imports only `core/*` and other features' `public.ts`. Floor code never imports company features.
6. Money goes through `core/money` only. No `*`, `/` or `parseFloat` on money in domain code.
7. Floor screens never call `fetch`. Writes go through the outbox.
8. Create tables only through the table helper. Migrations only add things (no drop, rename or new NOT NULL without a default).
9. Do not edit `core/` inside a feature pull request. Core changes are small, separate pull requests.
10. Use glossary words (Posting, Branch, Order, Bill, Payment, Cancel, Refund). Do not invent synonyms.
11. Do not add packages without asking the owner. Install scripts are off.
12. Tests: heavy on money, permissions, payments, approvals and the offline queue; light on simple screens. Every test must answer "what real behaviour breaks if this is deleted?". Use the real database, and test data with two companies.
13. A rule from the domain model is only done when a tool enforces it and the rule-break suite proves it.
14. Error messages and lint messages must name the fix.

## Done means
- `pnpm check` exits 0 (with `--db` if you touched data).
- `pnpm evidence` output is in the pull request, with the PR template filled in.
- Remaining risks are stated honestly, including anything not verified.

## Checks are manual for now
There are no automatic hooks. Run `pnpm check` yourself after a change and before you finish, and paste `pnpm evidence` in the pull request. If something keeps going wrong that a hook would catch, tell the owner.
