# Agent experience spec

Output of Session 4. Measurements are in `docs/decision-log.md` section K. Where this file differs from `architecture.md`, this file and section K win. Items marked **estimate** were not measured.

## 1. Speed targets and the three check tiers

| Tier | Where | Command | Target | Measured (1,001 modules, i7-8665U) |
|---|---|---|---|---|
| 1a Fast loop, no database | Laptop | `pnpm check <feature>`: type check, Biome, `tools/arch`, the feature's domain tests | 10 s | 2.5 s after an edit; 4.0 s median with 3 agents at once |
| 1b Fast loop, with database | Laptop | `pnpm check <feature> --db` adds the feature's database tests on the agent's own database | 30 s | database clone 46 to 78 ms; one command's writes 7.6 ms median. The full `--db` command is not measurable until the runner exists |
| 2 Full check | GitHub Actions | `check`, `db`, `build`, `perf`, `e2e`, `compat` | a few minutes | design only, see section 9 |
| 3 Merge gate | GitHub Actions | The same jobs as required checks | a few minutes | design only |

- Full tests and production builds never run on the laptop. Agents verify against the Vite dev server.
- The test data was synthetic and had no React, Tailwind or UI modules, so real numbers will be higher. **Re-measure when 30 real features exist** (about 1,000 modules). Alarm if tier 1a passes 10 s.
- The 3-agents-at-once number matters: 4 cores are shared, so one agent's check alone is not what agents experience.

## 2. Memory budget and parallelism

Measured: Vite dev server 306 to 404 MB, API server with worker 46 MB, one shared Chromium with 3 isolated contexts 939 MB, Postgres with 3 agent databases 109 MB, a coding agent process 105 to 308 MB (the earlier 415 MB upper figure is kept as the budget), a type-check burst 339 MB at 1,001 modules.

| Mode | Parts | Total |
|---|---|---|
| 3 full agents | 3 x (agent 415 + Vite 400 + API 50 + check burst 163) + Chromium 939 + Postgres 110 + portless about 220 (estimate, from A6) | about 4.4 GB of about 7.4 GB available |
| 5 to 6 light agents | 6 x (agent 415 + check burst 163 to 340) + Postgres 110 (estimate, parts measured, never run together) | 3.6 to 4.6 GB |

Rules:
- One git worktree per agent. `pnpm agent:init <name>` assigns a port, a name URL (portless over plain HTTP) and a database `wendo_<name>` cloned from a template.
- One Postgres, one Chromium. Hard cap: 3 full agents or 6 light agents. A 4th full agent is refused by `agent:init`, which checks free memory first.
- Light mode is the default. An agent starts the dev server and a browser only for a screen change.
- Each agent's browser state is isolated by Playwright contexts (see friction 2). Never `agent-browser connect` to the shared Chromium for anything that sets cookies.
- Collisions: migrations are timestamped per feature (CI fails on a migration older than the newest on `main`; `pnpm new migration --rebase` renames it). Shared types live in `public.ts`. Changes to `core/` go in their own small PR merged first. One file per command means two agents adding commands to one feature do not touch the same file.

## 3. Test policy

- **Heavy:** money, permissions, row security, payments, approvals, the offline queue, the sync trigger chain. These get property tests (`fast-check`), real-database tests and the rule-break suite.
- **Light:** simple screens get one smoke test from the generator. No snapshot tests of markup.
- **Every test answers:** what real behaviour breaks if this is deleted. A test that cannot answer is removed in review.
- **Real database, not mocks.** Each test runs in a transaction that is rolled back. Test data always has two companies and two branches. Tests connect as the narrow app role, never as a superuser, or row security would be silently skipped.
- **Rule-break suite (CI):** for each rule in section 6, a script breaks the protection on purpose (remove the unique index, drop the trigger, skip the revision check) and expects at least one named test to fail. Proven method from Session 2 and again in Session 4: removing the unique key made the duplicate test fail (8 rows for 4 orders); removing the revision check made 4,569 of 5,000 delivery schedules fail.
- **Fault injection for the floor store and outbox** runs in CI with a fixed seed list and a failing seed printed so it can be replayed.

## 4. Environment

- `pnpm dev` starts the API server, Vite, the worker and the database for this agent's worktree, with fake data (two companies, three branches) and a login for every role. External services are fakes. Cloudflare and Let's Encrypt are not involved.
- The clock is controllable: `core/time` reads a clock that tests and `pnpm dev --now 2027-01-10T18:00` can set.
- Postgres on the laptop: one Docker container from the pinned image `postgres:16-alpine`. Docker is not used on droplets or branch servers (J5). Node comes from `mise`.
- Imports use `#core/*` through the `imports` field in `package.json`, with explicit `.ts` extensions. `tsconfig` `paths` do not work under Node's built-in TypeScript stripping.
- Install scripts are off (`ignore-scripts=true`). Packages that need one are listed in `allowBuilds` in the pnpm config, reviewed in the PR.

## 5. How an agent finds code and verifies a change

**Finding.** `pnpm where <word>` prints the path of any command, query, permission, event, table or screen. Without a word it prints the registry (feature, side, tables, permissions, routes). Folder names are glossary words. A glossary lint rejects synonyms.

**File names (settled, was open in J).** One file per thing, with a fixed ending:

| Ending | Holds | Imported by client |
|---|---|---|
| `x.command.ts`, `x.query.ts` | name, permission, side, Valibot input schema, retry key requirement | yes |
| `x.command.server.ts`, `x.query.server.ts` | the implementation | never (`tools/arch` fails the build) |
| `x.screen.tsx` | one lazy screen and its size budget | yes |
| `x.event.ts`, `x.handler.ts` | event type (shared); reaction to another feature's event (server) | type only; never |
| `x.print.ts`, `x.job.ts` | pure print layout; background job | print yes; job no |

This replaces `api/contract.ts` plus `api/handlers.ts` from architecture §4. The reason: a shared file for all commands is edited by every agent adding a command, and it sends the client everything in the file.

**Generator.** `pnpm new feature|command|query|screen|migration <names...>`. Each output passes `pnpm check` immediately, including a starter test that runs.

**Verifying.**
- `pnpm check <feature> [--db]`: tier 1. Exit code 0 means the work is verified at that tier. With no feature name it works out the features changed in the current work (from git) and checks those.
- `pnpm evidence`: prints the result block the PR template requires (what ran, pass or fail, timings), so evidence is one command.
- `pnpm see <route> [--as <role>]`: a text outline of a screen from the dev server, in a fresh isolated browser context, so an agent checks UI without screenshots.
- Error messages name the fix. Examples: `Table loyalty_ledger has no scope label: add it to tables in features/loyalty/feature.ts`; `fetch is not allowed in floor UI (rule 13): use outbox from #core/floor-store`.
- Biome plugin rules (GritQL) were tested and work with a custom message and per-folder scope. They match syntax only and cannot see types.

## 6. Rules and the tool that enforces each

Rule numbers are from `domain-model.md` section 6. "Rule-break" means the CI suite in section 3 breaks it on purpose.

| # | Enforced by | Notes from Session 4 |
|---|---|---|
| 1 Money in whole cents | `Cents` branded type, `core/money` is the only place that does arithmetic; database domains `cents`, `guest_cents`; fast-check properties | A type-aware Biome rule is not possible. The Biome rule is path-scoped: no `*` or `/` in `features/**/domain/**` outside `core/money` calls |
| 2 Sale has tax category, receipt number, amounts | CHECK constraint when `closed_at` is set | |
| 3 Payments idempotent, M-Pesa code once | `retry_key` and `(branch_id, mpesa_code)` unique indexes; rule-break | Proven again in miniature: the duplicate protection is the unique key |
| 4 Payments append-only | Triggers and an app role with INSERT and SELECT only; `lint-sql` | |
| 5 No overpay | Trigger with a row lock; a concurrent database test | |
| 6 Approver and reason recorded | NOT NULL, CHECK and trigger on `approval_decisions` | |
| 7 Branch Manager or Director approves | Permission table, policy, trigger | |
| 8 Walls between branches and companies | Row security enabled and forced; global `pg_class` test; `security_invoker` views; tests run as the app role | Cost measured: about 0.67 ms per write with unindexed tables; add indexes on postings and role_permissions |
| 9 Company agrees with branch | Composite foreign keys | |
| 10 One branch per floor posting | Exclusion constraint | |
| 11 Transfer guard | Clearance table plus trigger on `postings` | Two-machine path stays design only until the sync exists |
| 12 Verification by Accountant or Director | Permission table, policy, trigger | |
| 13 Orders never lost or doubled | `Outbox` as the only client write path; Biome rule bans `fetch` in floor UI; unique ID; CI fault injection | Zero loss and zero duplicates held in a 10-check prototype. Cleared browser storage cannot be recovered by code (section 7) |
| 14 Offline approvals queued | Server-side approval queue; database test | |
| 15 Staff and customer separate | Separate tables, session types and database roles; `tools/arch` | |
| 16 AI never approves | `Actor` union; `approve()` takes `StaffActor`; CHECK `actor_type <> 'ai'` | |
| 17 Every table labelled | `core.register_table`; starter test; `lint-sql` | |
| 18 Floor write records a change | Trigger writes `core.changes`; starter test; fault injection | Trigger chain cost 0.15 ms over row security, bar was 5 ms |
| 19 One writer per branch | Write policy with `core.writable_branches()`; sync refuses stale epoch; cutover tests | Design only until the sync exists |
| 20 Floor reads company data through a copy | `tools/arch`; typed `ctx.read`; app role has SELECT only on company tables | |

## 7. Floor store, offline queue and the phone

**Store contents by role** (settled, was open in J). Nothing in the store is a secret: no password or PIN hash, no payroll, no other branch.

| Role | Held on the phone or tablet |
|---|---|
| Waiter | Open orders and tickets of this branch for today, tables, menu with prices, staff names, notices, own outbox, device token, session token |
| Chef, barista (station tablet) | Tickets for its station, menu names, notices, outbox, tokens. No prices, no bills |
| Branch Manager | Waiter data plus approval requests, cash entries of the branch, suspensions |
| Office roles | No persisted floor store. TanStack Query memory cache only, cleared on logout |

**Logout:** the session token and every cache are cleared. The outbox is kept if it has unsent orders, and logout shows "N orders are not sent yet". The device token stays (it belongs to the device).
**Lost or stolen phone:** the Branch Manager revokes the device and its sessions from the ops dashboard. The next time the phone is online it is refused and wipes its store. Until then the store holds only branch floor data, and the session ends within 12 hours.
**Persistent storage and visible count:** the app calls `navigator.storage.persist()` at first login and records the answer in the device record. The shell always shows "N unsent" when N is above 0, and a red badge if persistence was refused. In a desktop Chromium the answer was `false`, so the Android result is a hardware test (section 10).
**Detecting a lost queue:** a device sends a heartbeat every 30 s while online with its unsent count and its oldest unsent age. An alert fires if the count is above 0 for 10 minutes, and when a device re-enrols after wiped storage the manager sees its last reported count. This is design only.

## 8. Friction list

1. **Dev server memory:** Vite is 306 to 404 MB on a 201-module app, not the 150 MB estimated. It grows with the app. Light mode is the default.
2. **agent-browser sessions on a shared Chromium are not isolated** (open risk 12 confirmed): session B read session A's cookie and localStorage. Playwright contexts on the same Chromium were isolated. `pnpm see` uses a fresh context. Own-Chromium sessions cost about 765 MB each, too much. Per-agent hostnames (portless) separate origins; this is untested.
3. **Biome cannot see types.** Rules are path-scoped. Typed lint rules such as no-floating-promises are what we lose by leaving ESLint; check Biome's own equivalent when the first build exists.
4. **Node type stripping:** no `tsconfig` paths, no enums or parameter properties, explicit `.ts` extensions. Use `#core/*` and `erasableSyntaxOnly`.
5. **pnpm 11 exits with code 1 after a successful install when a script was ignored** (`ERR_PNPM_IGNORED_BUILDS`). An agent can read that as a failed install. Document it and use `allowBuilds`.
6. **Slow or failing downloads:** one tarball timed out three times on a poor network. Agents do not install packages. A shared pnpm store and `fetch-timeout` settings go in `.npmrc`.
7. **Kysely insert types:** `Generated<>` columns are not accepted by `insert().values()`. Use `Insertable<>` helpers; the generator's `queries` stub includes one.
8. **Instructions are not enforcement:** a test agent told to run nothing used a shell command to write its file. Only tools enforce, not prompts: CI, and a block added when a real problem needs one.
9. **The laptop is already busy:** about 8.4 GB used and 4 GB of swap before any agent runs.

## 9. Pipeline summary (detail in the pipeline files)

- Numbers checked on 2026-11-04: Free gives about 2,000 and Team about 3,000 Linux 2-core minutes a month (the real unit is $12 and $18 of compute at $0.006 a minute). Extra minutes cost $0.006.
- Estimate per pull request, all jobs: about 25 minutes. Per push: about 10 minutes (`check`, `db`, `build`). At 3 agents and about 4 pull requests a day this is about 2,200 minutes a month, above Free, inside Team with margin. Recommendation: **Team**, also for required checks on a private repository.
- The repository is public for now (free minutes and free required checks) and goes private after Team is bought; buy Team first, because protection is not enforced on a private repository without a paid plan.
- Savings: cache the pnpm store and Playwright browsers; cancel superseded runs; path filters; `perf` and `e2e` only on pull requests that are ready for review; `compat` only when `data/`, `core/sync` or `core/db` change; `image` on tags only.

## 10. Postponed work, with what it measures and when it runs

| Item | Measures | Runs when |
|---|---|---|
| Friction audit | Tool calls and minutes lost per task, and which friction items above recur | After 3 features exist through the generator |
| Agent trials | Fresh agents on worked tasks W1 to W10 (architecture §11) through the real generator: first-try pass rate, tool calls, tokens, what each tool caught | After the foundation and the generator exist |
| Attack round | Agents told to bypass each rule 1 to 20, leak data across branches, or write bad code; every attempt must be caught by a named tool | After the rule-break suite exists |
| Parallel collision tests | 3 agents at once on migrations, shared types, `core/` edits and merges: conflicts per day, failed merges | After 3 real features exist |
| Re-run of experiments 3 and 4 on real code | Trigger cost (bar 5 ms; P12 30 ms) and offline and floor-store fault injection, as CI jobs | When the command runner, floor store and outbox exist |
| Real phone speed | First load about 2 s, tap at most 100 ms, JS heap at most 60 MB, on a mid-range Android phone | When a phone is available |
| Persistent storage on Android | `persist()` result for a browser tab and an installed app; does clearing site data wipe the outbox | Same phone |
| Printing (I3a) | 100 prints in a row, tap to paper under 2 s, recovery after power or Bluetooth drops, second device behaviour | When printer models are known |
| Same-address DNS | Phone and router follow the local DNS after cutover, Private DNS off | When a branch router is available |
| Cloudflare live streams | SSE with the 20 s heartbeat survives the proxy | When the staging droplet exists |
| Update, rollback, restore, cutover rehearsals | Each completes on staging with no data lost | Before the first branch server |

## 11. Hooks and the Scorecard

**Hooks: none for now** (owner decision, 2026-11-05). A hook is added only when a real, repeated problem needs one, not in advance. What was learned while testing a first set is in decision log K-13, so the next one starts from it. The permission rules in `.claude/settings.json` stay (allow and ask first). There are no deny rules for now (owner decision, 2026-11-05): a block is added only when a real problem shows it is needed.

**Scorecard.** Reviewed after the first 3 features, then monthly. Each review adds a dated row to the decision log. Raw data: `pnpm check` and `pnpm evidence` record their own time and result to a local log (not committed), plus `gh run list` and `gh pr list --json`. A `pnpm metrics` command (foundation build) prints the summary.

| Measure | Starting alarm (proposal, confirm after 3 features) |
|---|---|
| `pnpm check` time (tier 1a) | over 10 s; with `--db` over 30 s |
| `pnpm check` passes on the first run of a task | under two thirds of tasks |
| CI passes on the first push | under two thirds of pull requests |
| Time from task start to pull request | track, no alarm yet |
| Tool calls per task, review rounds per pull request | track, no alarm yet |
| CI minutes used in the month | over 80% of the plan |
| Peak memory with the agents running | over 6 GB |
