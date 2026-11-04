# Feature shape

Updated in Session 3 to match `docs/decision-log.md` section J (the chosen architecture). The folder rules and starter tests below are decided. Items marked **Session 4** are not.

## What a feature is

A feature is one self-contained folder. Every feature has the same rooms, so an agent finds code from the path and the file ending, without reading documents. No central list is ever edited: features are found by a glob on `features/*/*/feature.ts`.

## Where it lives

- `features/floor/<name>/` runs on the branch's writer (the cloud or the branch server). Its data is branch-scoped.
- `features/company/<name>/` runs in the cloud only.
- `core/` holds shared services: Money, Actor, postings and permissions, approvals, audit, events, the command runner, sync, serving (served-by and epoch), print, the floor store, UI primitives and testing helpers.
- A feature may import only `core/*` and other features' `public.ts`. Floor code cannot import company features and has a type that cannot see company tables.

## The rooms

- `feature.ts`: name (must equal the folder), tables with their scope labels, permissions and grants, copied company data (`copies`), emitted and handled events, settings.
- `public.ts`: the only file other features may import (types, event types, copy readers, slot names).
- `domain/`: pure logic with no database, no screens, no network. Fast tests next to it.
- `data/`: Kysely types, queries, and `migrations/<UTC timestamp>_<name>.sql` (add-only; a drop or rename waits in a separate contract step).
- `api/`: the commands and queries, each naming its permission, side, input schema and, for floor commands, a device-made retry key.
- `events/`: events this feature emits and handlers for events it consumes.
- `jobs/`: optional background jobs.
- `print/`: optional ticket and receipt layouts (pure, with golden tests).
- `ui/`: routes and screens, one lazy chunk per screen, each with its own size budget.
- `tests/`: `domain/` (fast), `db/` (real database, rolled back), and a generated starter test.

**File names (settled in Session 4, decision log K-2).** One file per thing, with a fixed ending: `x.command.ts` and `x.query.ts` (shared contract: name, permission, side, input schema, retry key), `x.command.server.ts` and `x.query.server.ts` (implementation, server only, never imported by the client), `x.screen.tsx`, `x.event.ts` (event type), `x.handler.ts` (reaction to an event, server only), `x.print.ts`, `x.job.ts`. This replaces `api/contract.ts` plus `handlers.ts`. Imports use `#core/*` with explicit `.ts` extensions. Validation is Valibot.

## The generator

`pnpm new feature loyalty --side floor --tables loyalty_ledger:branch --perms read,write` creates the folder, `feature.ts`, a first migration through the table helper, `public.ts`, one sample command and screen, and a starter test that already passes. `pnpm new command|query|screen|migration <feature> <name>` adds one inside an existing feature. `pnpm where <word>` prints the path of any command, permission, event, table or screen; with no word it lists the registry. `pnpm agent:init <name>` sets up a worktree, port, URL and database for an agent (spec section 2).

## Table helper

Every table is created through one SQL helper that:
- enables and forces row security and adds the standard policies;
- adds the composite key tying the company to its branch;
- records the scope label (branch-scoped or company-wide);
- for floor tables, attaches one trigger chain that checks the writer epoch, stamps the actor, writes the audit entry and writes the change log in the same transaction;
- for append-only tables, blocks update and delete;
- makes sales write-once.

A migration that creates a table any other way fails the build, and the message names the fix.

## Starter tests (every feature, generated)

- A user without the permission gets nothing.
- Every table has a company ID, forced row security, a policy and a scope label.
- Every view is `security_invoker`.
- Money columns use the cents or guest-cents domains.
- A floor write also appears in the change log in the same transaction.
- A floor feature reads company data only through its read-only copy.
- Test data always includes two companies.
- A CI suite breaks each rule on purpose and expects the tests to fail.

## Check commands

- `pnpm check <feature>`: type check, lint and the feature's fast tests. Target about 10 s.
- `pnpm check <feature> --db`: adds the feature's database tests on the agent's own database. Target about 30 s.
- `pnpm see <route>`: a text outline of a screen, so agents can check the UI without screenshots.
- Full tests and production builds run only in CI.

## Why this shape

- An agent reads one folder, and the path says where it runs.
- Two agents on two features almost never edit the same file.
- Rules in `domain-model.md` section 6 fail loudly when broken, in the type check, the lint, a test or the database.
