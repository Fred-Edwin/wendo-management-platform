# Architecture: chosen base design (fast-light)

This is the base design chosen in Session 3. Where `docs/decision-log.md` section J differs, **section J wins**. The differences:
- Folders are `features/floor/<name>` and `features/company/<name>`, with fixed file endings (J6), not a flat `features/<name>` with a `side` field.
- The cloud supports branch versions N, N-1 and N-2, not only N-1.
- Remote actions on a branch run by its own server are forwarded to the branch writer, and approval decisions are branch records (J10).
- A written fallback exists for the same-address DNS trick (J8).
- Remote access starts with Tailscale (J11). Floor sessions last 12 hours. Docker is not used on droplets or branch servers (J5).
- Ideas borrowed from the other designs are listed in J.

Of the questions for the owner in section 14, J answers last resort, failover, offline suspension and remote access. The rest (printers, droplet size, backups in object storage, alert channel, smart plug, domain, phone management) are still open and belong to Session 4 or later.

## Plain-language summary (for the owner)

- **The app is one small website.** It has no page-building server. The phone downloads a small app once (about 130 kB compressed), keeps it, and then only exchanges data. Opening it again takes under half a second, with or without internet.
- **Each branch has one web address for floor work**, for example `floor-kilimani.wendo.co.ke`. Before the branch server arrives, that address leads to the cloud. After cutover, the same address leads to the branch server on the branch Wi-Fi. Phones keep the same app, the same login and the same queue of unsent orders. Cutover means changing where the address points. Nobody reinstalls anything.
- **The database itself is the guard.** It decides who may see which branch, which machine may write for a branch, and that payments are never overpaid or doubled. A bug or an AI-written query in the app cannot get past it.
- **Every change at a branch is written down as a "change record" at the same moment.** That one list feeds the kitchen screens and the sync to the cloud. Screens that drop off catch up from it, and the cloud receives every change exactly once.
- **Printing never touches the internet.** Phones send receipts straight to a small print relay on the branch network, and the relay drives the Bluetooth printer. This works even when the branch server is down.
- **You run the branch machines from one dashboard.** It shows health and last sync, raises alerts, runs nightly backups with weekly restore tests, rolls out updates one branch at a time with automatic undo, and gives remote access. A spare machine sits pre-imaged on a shelf.
- **On the laptop, three agents run everything in about 4.5 GB, or six agents run tests only in about 4.4 GB.** The tools were chosen because they are light: Vite (61 MB, against 610 MB for Next.js).
- **Biggest weakness:** the "same address, two places" trick relies on DNS behaving on the phones and the router. The Android "Private DNS" setting must be off, and the router must hand out the branch box as its DNS server. Section 13 covers this.

---

## 1. Summary and emphasis

**What I optimised**

| Goal | How |
|---|---|
| Phone speed | Client-only React app (no server rendering), service worker cache, a live in-memory **floor store** fed by one change stream, optimistic taps, virtualised lists, CSS without runtime cost |
| Small downloads | First load budget 130 kB JS gzip, 35 kB per route chunk, enforced in CI with `size-limit` |
| Instant feedback | Every floor command updates the screen and the phone queue first (under 50 ms), then the server confirms |
| Laptop memory | Vite instead of Next.js (A4); server dev runs on Node's built-in TypeScript stripping (no bundler); one Postgres, one Chromium; tests mostly pure functions |
| Branch machine memory | One Node process for the app, one for sync, Postgres tuned for 8 GB; the whole stack is about 1.6 GB (estimate built on section F numbers) |

**What I gave up**

| Given up | Cost |
|---|---|
| Server rendering and React Server Components | The first paint before the app is cached waits for JS. Mitigated by an inline HTML skeleton and the 130 kB budget |
| One universal data-loading pattern | Two patterns: the **floor store** for live floor data, TanStack Query for company pages. A lint rule picks one per side |
| Off-the-shelf sync and RPC frameworks | We own about 1,500 lines (sync, outbox, RPC glue, migration runner). These are the heaviest-tested parts of core |
| Separate apps per role | One SPA with lazy route chunks per role. Kitchen and waiter share the shell |

---

## 2. Stack decisions

Versions: pinned list from decision log A unless stated. Other versions were read from the npm registry (`registry.npmjs.org/<pkg>/latest`) and GitHub releases on **2026-11-04**. Adopt the latest release that is at least two weeks old, then upgrade deliberately.

| Layer | Choice | Why | Rejected | Evidence or cost |
|---|---|---|---|---|
| Language | TypeScript 7.0.2 (native compiler), `strict`, `noUncheckedIndexedAccess`, `exactOptionalPropertyTypes`, `erasableSyntaxOnly` | Fast type check; `erasableSyntaxOnly` lets Node run server files directly | TS 5.x (slower) | A1: 0.26 s on 200 modules |
| Runtime | Node 26.8.1, built-in type stripping in dev | No bundler or watcher process for the server in dev | tsx, ts-node, Bun | Saves one process per agent (estimate 80 to 150 MB) |
| Framework and build | **Vite 8.3.5 SPA**, React 19.3.1 | 61 MB and 0.18 s start against 610 MB and 5.5 s; a static SPA is what a service worker caches best offline; the branch server serves files, not rendered pages | Next.js 16.4.5 (memory, server-rendering model fights offline and the branch box), Remix/React Router framework mode | A4 |
| Routing | TanStack Router 1.170 (code-based routes, typed params) | Each feature exports its routes; core collects them with `import.meta.glob`. No generated central route file, so no merge clashes | File-based routing (generates one shared `routeTree.gen.ts`, a collision point) | npm 2026-11-04 |
| Floor data and state | **Own `core/floor-store`**: in-memory tables kept live by one server-sent event stream, persisted to IndexedDB (`idb` 8.0.4, about 1 kB), read with `useSyncExternalStore` selectors | Floor screens show live, shared, offline-capable data; one stream means one reconnect and one catch-up path (E3) | TanStack DB, Replicache, Zero, ElectricSQL (heavier, or need their own server, or are tied to one database writer model) | C(d): 3 to 7 ms tap to ticket with this pattern |
| Company data | TanStack Query 5.105 | Paged, cached server reads for HR, finance and reports. Loaded only in company route chunks | SWR (fewer features), own code | npm 2026-11-04 |
| Local UI state | React state plus `useSyncExternalStore`. No state library | Fewer bytes, one pattern | Zustand, Redux | |
| API style | **Typed commands and queries** (`defineCommand`, `defineQuery`) over `POST /rpc/<feature>.<name>`, built on Hono 4.13 | A contract file is shared by client and server; the handler stays server-only. Every entry names its permission, side and input schema. No URL design to argue over | REST (more decisions per endpoint), tRPC (heavier client, Next-oriented), GraphQL | Hono about 14 kB (estimate) on Node |
| Validation and shared types | Valibot 1.5 schemas in `api/contract.ts`; types inferred from schemas | Tree-shakes to about 1 to 2 kB per screen (estimate; Zod 4 full is several times larger) | Zod 4 (`zod/mini` is close; Valibot chosen for size) | npm 2026-11-04 |
| Database access | Kysely 0.29 on `pg` 8.23.1. Each feature declares its own table types in `data/schema.ts`; a database test checks them against the real schema | Typed SQL with no engine binary or runtime codegen; per-feature types avoid one central generated file | Prisma (engine memory, hides SQL), Drizzle (fine, but its schema-as-code migrations push toward one central schema) | |
| Migrations | Plain SQL files in each feature's `data/migrations/`, named `<UTC timestamp>_<name>.sql`; a 150-line runner in `core/db/migrate`; **expand-only** rules checked by `tools/lint-sql` | SQL is what row security, triggers and checks need; timestamps from separate folders never clash; expand-only keeps older branch machines working (G14) | dbmate 2.37 (one folder only), node-pg-migrate | |
| Background worker | graphile-worker 0.18, Postgres-based, **run inside the app process** in dev and on branch machines, as its own process on the cloud | No Redis; jobs live in the same transaction as the change that caused them | pg-boss 12 (fine; graphile-worker uses LISTEN/NOTIFY so jobs start in milliseconds), BullMQ (Redis) | |
| Real-time push | **Server-sent events (SSE)** from `GET /floor/stream?since=<seq>`, woken by Postgres `LISTEN/NOTIFY`, with `Last-Event-ID` resume; a 20 s heartbeat for Cloudflare | Push measured 63 to 265 ms against 439 to 522 ms for polling (C(e)); notify through the database costs 1 to 6 ms and allows several processes (E2); native browser reconnect | WebSockets (more code, no built-in resume), polling | C(e), E2, E3 |
| Floor and company split | `side: 'floor' \| 'company'` in each `feature.ts`. The server mounts features by mode; type-level read-only access for company data in floor handlers; database grants | See §6 and §8 | Separate codebases | |
| Sync protocol | The change log (`core.changes`) is written by a **database trigger** on every registered table in the same transaction; sequence-numbered; pushed and pulled by `core/sync`; idempotent upsert by primary key and row version; writer epoch fencing | Proven in miniature (section F: 4,600 events exactly once in 5.0 s) | Logical replication (can't filter per branch safely, fragile across versions), CRDTs (not needed with one writer) | F |
| Branch bundle | One tarball `wendo-<ver>.tar.zst`: server bundle (Rolldown via Vite), web assets, SQL, ops agent, relay. `wendo serve --mode=cloud\|branch`, `wendo relay`, `wendo agent` | One deployable for droplet and mini PC (G10) | Docker on branch (extra memory and moving parts) | |
| Installer and imaging | Debian 13 minimal image built in CI; `wendo-install <branch-code>` enrols the machine | Pre-imaged spare in one command (G13) | Hand-built machines | |
| Cutover | Ops dashboard action with writer-epoch fencing and a DNS switch (§8) | | | |
| Update compatibility | Expand-only migrations, a sync protocol version window N and N-1, a CI compatibility job running the new cloud against the old branch image | §8 | Lock-step upgrades | |
| Offline staff login | Device token plus 6-digit PIN, checked by a database function on the branch server against replicated, hashed credentials | §8 | Cloud-only login | |
| Sessions and "who is asking" | **Database-issued signed session tokens**, verified inside Postgres on every transaction (G6, §6) | | Plain `SET app.user_id` (fails G6) | |
| Components | React Aria Components 1.22 for hard widgets (dialog, menu, combobox, date), our own simple primitives for buttons, lists and tiles | Accessible, headless, tree-shakeable; the look stays ours | MUI, Mantine (heavy runtime styles), shadcn (fine, but Radix plus copying adds weight and drift) | npm 2026-11-04 |
| Styling and tokens | Tailwind CSS 4.3 with tokens defined once in `core/ui/tokens.css` (`@theme`), compiled by Lightning CSS. Arbitrary colour and spacing values are banned by a check | No runtime CSS, small output, agents write it well | CSS-in-JS (runtime cost), vanilla-extract (more files per component) | |
| Lists | `@tanstack/react-virtual` 3.14 inside `core/ui/List` (virtualises above 60 rows) | Long lists stay at 60 fps on a slow phone | react-window | |
| Offline shell | `vite-plugin-pwa` 2.0 (Workbox 7.4) precache | App loads with no network (C(d)) | Hand-written service worker | |
| Lint and format | Biome 2.6 (format plus lint plus GritQL plugin rules for boundaries) and `tools/arch` (an import-graph test) | One fast tool; A1: 0.22 s | ESLint 10 (kept out of the fast loop; memory) | A1 |
| Tests | Vitest 5.1 (unit and real-database), `fast-check` for money properties, Playwright 1.64.1 (e2e and performance, CI only), Lighthouse CI 0.16 | | Jest | A1 |
| Repository | **One package** (single `package.json`, pnpm 11.3), folders below | One install, one `node_modules`, one config each | Monorepo with workspaces (more configs, more memory) | |
| Hosting | Production droplet and staging droplet, each running Caddy, app, worker and Postgres 16. Backups in DigitalOcean Spaces (see Q3) | G10 | Managed database (extra cost; can be adopted later) | |
| Cloudflare | Free plan: DNS, proxy for `app.`, `floor-*.`, `sync.`, `staging.`; cache only `/assets/*`; Full (strict) TLS with Origin CA certificates; one rate-limit rule on login | §9 | | Free-plan terms to verify (I5) |
| CI | GitHub Actions: check, database tests, build, bundle budget, performance, e2e, compatibility, image | §9, §10 | | |
| Remote ops | `features/branch-ops` (company feature) plus `ops/agent` on each machine; Tailscale for access (Q4) | G13 | | |
| Feature registration | `defineFeature()` in `feature.ts`, found by glob on the server (`fs.globSync`) and the client (`import.meta.glob`) | No central list | Hand-edited index | |

---

## 3. Codebase map

```
wendo/
├─ features/                     one folder per feature ("shop"); nothing central to edit
│  └─ <feature>/                 shape in §4 (orders, kitchen-display, payments, receipts, notices,
│                                menu, staff, inventory, branch-ops, approvals-inbox, director-board ...)
├─ core/                         shared services; changes here are small, separate PRs
│  ├─ money/                     Cents and Shillings types, arithmetic, largest-remainder split, formatting
│  ├─ ids/                       device-made IDs (UUIDv7), retry keys, device letters
│  ├─ numbering/                 order and receipt numbers (online counter, offline letter prefix)
│  ├─ time/                      business day, controllable clock, phone clock-skew flag
│  ├─ actor/                     Actor union (staff, customer, guest, system, ai with actingFor)
│  ├─ auth/                      login, PIN, device tokens, signed sessions (client and server halves)
│  ├─ access/                    postings, permissions, "can"; role grants collected from feature.ts files
│  ├─ approvals/                 request, vote, decide; Branch Manager or Director only; never AI
│  ├─ audit/                     automatic audit entry per command, in the same transaction
│  ├─ events/                    feature events (emit and handle), stored and synced like rows
│  ├─ db/                        session-bound transactions, migration runner, SQL helpers, roles
│  ├─ sync/                      change log, push and pull, epoch fencing, version window
│  ├─ serving/                   served-by setting, writer lease, cutover steps
│  ├─ rpc/                       defineCommand, defineQuery, HTTP binding, client caller
│  ├─ feature/                   defineFeature, discovery, registry checks
│  ├─ floor-store/               client live tables, SSE stream, IndexedDB persistence, outbox queue
│  ├─ print/                     PrintJob, print adapter, drivers (relay, webserial, fake), ESC/POS builder
│  ├─ notify/                    in-app banners, optional Web Push when online
│  ├─ jobs/                      graphile-worker wiring; features add tasks in their jobs/ folder
│  └─ ui/                        tokens.css, primitives, List, Sheet, Toast, AppShell, role navigation
├─ web/                          SPA entry: index.html (inline skeleton), main.tsx, sw.ts, router assembly
├─ server/                       main.ts (modes: cloud, branch), Hono app assembly, SSE, static files
├─ ops/
│  ├─ agent/                     small program on each branch machine: health, updates, backups, rollback
│  ├─ relay/                     print relay: HTTPS on the LAN, job queue, printer transports (bt-rfcomm, usb, tcp9100)
│  ├─ image/                     Debian image recipe, wendo-install, dnsmasq, Caddy, systemd units
│  └─ deploy/                    droplet provisioning, release manifests, staged-rollout config
├─ db/                           database roles, bootstrap SQL (core schema), template-DB builder
├─ tools/                        new-feature, new-migration, check, features (lists the registry), lint-sql, arch, perf
├─ e2e/                          cross-feature journeys (order → kitchen → pay → receipt), perf scenarios
└─ .github/workflows/            ci.yml, release.yml, image.yml
```

---

## 4. The feature shape (real version)

```
features/payments/
├─ feature.ts                 sign on the door (defineFeature)
├─ domain/                    pure rules: billing.ts, split.ts, can-pay.ts (no I/O, no React)
├─ data/
│  ├─ schema.ts               Kysely table types for this feature's tables
│  ├─ queries.ts              reusable SQL fragments (only this feature uses them)
│  └─ migrations/20270105T0830_payments.sql
├─ api/
│  ├─ contract.ts             commands and queries: name, permission, input schema (shared with client)
│  └─ handlers.ts             server-only implementations (receive ctx; cannot open connections)
├─ events/
│  ├─ emits.ts                public event types other features may handle (e.g. saleClosed)
│  └─ handlers.ts             reactions to other features' events
├─ jobs/                      optional background tasks
├─ print/                     optional ticket and receipt templates (pure: data → PrintDoc)
├─ ui/
│  ├─ routes.ts               route list: path, role(s), lazy screen, budget
│  ├─ screens/TakePayment.tsx one file per screen, lazy-loaded
│  └─ parts/                  pieces used only by this feature's screens
├─ public.ts                  the only file other features may import (event types, contract, display parts)
└─ tests/
   ├─ domain/*.test.ts        pure, under 1 s
   ├─ db/*.test.ts            real database, each test in a rolled-back transaction
   └─ starter.test.ts         generated contract tests driven by feature.ts (see below)
```

**The sign on the door** (signature only):

```ts
export default defineFeature({
  name: 'payments',
  side: 'floor',                 // floor: runs wherever the branch's writer is; company: cloud only
  summary: 'Record cash, M-Pesa and card payments against bills',
  permissions: { read: 'payments.read', write: 'payments.write', verify: 'payments.verify', reject: 'payments.reject' },
  grants: { waiter: ['read', 'write'], branch_manager: ['read', 'reject'], accountant: ['read', 'verify', 'reject'] },
  tables: { payments: 'branch', payment_events: 'branch' },   // every table labelled (rule 17)
  readsCompany: ['menu.items', 'staff.people'],              // company data read through the read-only view
  emits: ['payments.recorded', 'payments.saleClosed'],
  handles: ['orders.cancelled'],
  settings: {},
})
```

| Concern | How it is expressed |
|---|---|
| Registration | Discovery globs `features/*/feature.ts`. Startup fails, naming the file, if two features share a name, a table, a permission or a route path |
| Floor or company | `side`. The branch server mounts only `floor` features. Floor handlers get `ctx.tx` typed with **only their own branch tables** and `ctx.read`, which exposes `selectFrom` on `readsCompany` tables only |
| Permissions | Strings declared in `feature.ts`; contracts reference `payments.permissions.write` by import, so a typo is a type error. Role grants come from each feature's `grants` and are loaded into `role_permissions` at boot (no migration, no central file) |
| Commands | `defineCommand({ name, permission, input, idempotent: true })` in `contract.ts`; the runner opens the transaction, binds the session, checks the permission, runs the handler, writes the audit entry and returns |
| UI | `ui/routes.ts` exports `[{ path: '/pay/$billId', roles: ['waiter'], screen: () => import('./screens/TakePayment'), budgetKb: 25 }]` |

**Generator commands** (each output passes `check` immediately):

| Command | Creates |
|---|---|
| `pnpm new-feature loyalty --side floor --tables loyalty_ledger:branch --perms read,write` | Folder, `feature.ts`, empty `domain/`, first migration calling `core.register_table`, `schema.ts`, contract with one sample query, `routes.ts` with one screen, `public.ts`, starter test |
| `pnpm new-migration payments add_service_charge` | Timestamped SQL file with the expand-only header |
| `pnpm new-command payments recordRefund` | Contract entry, handler stub, db test stub |
| `pnpm new-screen orders TableMap --role waiter` | Screen file, route entry, budget line |

**Starter tests** (`tests/starter.test.ts`, generated from `feature.ts` at test time, so it never goes stale):

1. A user without each permission gets nothing from each command and query.
2. Every table in `tables` exists, has `company_id`, row security enabled **and forced**, the standard policies, a scope label, and the change-log trigger.
3. Every view is `security_invoker`.
4. Every `*_cents` column uses the `cents` or `guest_cents` domain.
5. A floor command's write appears in `core.changes` in the same transaction (rolled back together).
6. Floor handlers cannot write company tables (the type check covers this; the test also tries it on a branch-mode database and expects a grant error).
7. `schema.ts` matches the database columns.

---

## 5. Core data model

All IDs are UUIDv7 made on the device (rule 13). Every table has `company_id`. Branch-scoped tables have `(company_id, branch_id)` with a composite foreign key to `branches(company_id, id)` (rule 9). Tables are created through `core.register_table(name, scope, side)`, which enables and forces row security, adds the standard policies, the change-log trigger and the scope label. Hand-written policies are allowed only with a `-- reviewed-policy:` comment, which `lint-sql` lists in the PR.

| Area | Table (scope) | Key columns and rules |
|---|---|---|
| Structure | `companies` (company), `branches` (company), `departments` (company), `stations` (company), `devices` (branch), `printers` (company) | `branches.currency`, `branches.code`; `printers.type` references `printer_types` (seeded from code) |
| Serving | `branch_serving` (company, written in cloud only) | `branch_id`, `served_by` (`cloud` \| `server`), `server_machine_id`, `epoch` (integer, only goes up), `state` (`active`, `draining`, `handover`) |
| Writer lease | `core.writer_lease` (local to each database, not synced) | `branch_id`, `epoch`. `core.writable_branches()` reads it |
| People | `persons` (company), `credentials` (company; password hash, PIN hash, bcrypt via pgcrypto), `devices` (branch) | Customers are in a separate `customers` table when added (rule 15) |
| Postings | `roles` (company), `role_permissions` (company, loaded from feature files), `postings` (company) | `postings(person_id, role, scope_branch_id null for company-wide, starts_on, ends_on)`; exclusion constraint: a floor role holds one branch at a time (rule 10) |
| Local suspension | `suspensions` (branch) | Written by a Branch Manager at the branch, immediate; synced up; checked on every session use |
| Transfer | `transfer_requests` (company), `transfer_clearances` (branch) | §8 (C8) |
| Actor | Column group on every audited row: `actor_type` (`staff`, `customer`, `guest`, `system`, `ai`), `actor_id`, `acting_for_person_id` (required when `ai`, check constraint) | Rule 16 |
| Audit | `branch_audit` (branch), `company_audit` (company), append-only | One row per command (written by the command runner), plus a session ID. No UPDATE or DELETE grant for any app role |
| Approvals | `approval_requests` (branch), `approval_votes` (company), `approval_decisions` (branch) | §8. A decision row's approver must hold `branch_manager` for that branch or `director` on its date, `actor_type <> 'ai'`, `reason` not null (rules 6, 7, 16) |
| Change log | `core.changes` (branch rows per branch; company rows) | `seq` (bigserial per database), `branch_id` (null for company rows), `table`, `pk`, `rev`, `epoch`, `row` (jsonb), `schema_version`, `sent_at` |
| Events | `branch_events` (branch), `company_events` (company) | Feature events stored as rows, so they sync like everything else |
| Notices | `notices` (company: HR and Director), `branch_notices` (branch: manager, local), `notice_reads` (branch) | Target: branch, department, role, person |
| Orders | `tables` (branch), `orders` (branch), `order_lines` (branch), `order_number_counters` (branch) | `orders.channel`, `orders.waiter_person_id` (nullable for future channels), `order_no` text (`17` or `B-12`), `business_day`, `device_time`, `received_at`, `clash_flag` |
| Bills | `bills` (branch), `bill_lines` (branch) | Totals frozen at close: `subtotal_cents`, `discount_cents`, `vat_cents`, `total_cents` (guest_cents), `tip_cents`, `tax_category`, `receipt_no` (not null when closed, rule 2) |
| Payments | `payments` (branch, append-only), `payment_events` (branch, append-only) | `retry_key` unique (rule 3), `method`, `amount_cents guest_cents`, `mpesa_code` unique per branch; status derived from events (rule 4); an overpay trigger takes a row lock on the bill (rule 5) |
| Cross-branch uniqueness | `mpesa_code_registry` (company, cloud only) | One row per code; a second branch using the same code → flag (detect and flag, F) |
| Cash | `cash_entries` (branch) | Per waiter per shift; "unsettled" means not yet received by the manager |
| Print | `print_jobs` (on the relay's own SQLite file, plus a `print_outcomes` row (branch) synced up) | Device-made job ID; the relay ignores a repeat |
| Ops | `machines`, `heartbeats`, `backups`, `restore_drills`, `releases`, `rollouts`, `alerts` (company, branch-ops feature) | |

**Money in the database:** `CREATE DOMAIN cents AS bigint` and `CREATE DOMAIN guest_cents AS bigint CHECK (VALUE % 100 = 0)` (rule 1, G2). Payment, price, tip, discount and change columns are `guest_cents`; VAT and cost columns are `cents`.

**Permissions mapping:** `core.can(permission, branch_id)` is a `STABLE` SQL function over `postings` and `role_permissions` for the session's person on the business date. Policies call it once per query in the form `branch_id = ANY ((SELECT core.visible_branches('payments.read')))` (C(c): 18 to 30 ms on 100,000 rows).

---

## 6. Boundaries and enforcement

| # | Rule | Type | Lint / arch | Test | Database |
|---|---|---|---|---|---|
| 1 | Whole cents; guest amounts whole shillings | `Cents`, `Shillings` branded integers; constructors reject non-integers; no `number` arithmetic on money | Biome rule: no `*`, `/`, `parseFloat` on money-typed values | `fast-check` properties (100,000 bills, C(a)) | `cents`, `guest_cents` domains; starter test 4 |
| 2 | Sale stores tax category, receipt number, final amounts | `ClosedBill` type | | db test | NOT NULL checks once `closed_at` is set |
| 3 | Idempotent payments; one use per M-Pesa code | `idempotent: true` on the contract requires `retryKey` | | db tests (C(b)) | unique `retry_key`; unique `(branch_id, mpesa_code)`; cloud registry flags cross-branch reuse |
| 4 | Append-only payments | No update command exists | `lint-sql`: no UPDATE or DELETE grant on append-only tables | starter test | Triggers raise; app role has INSERT and SELECT only |
| 5 | Never overpaid | | | concurrent db test | Trigger with `SELECT ... FOR UPDATE` on the bill |
| 6, 7, 14 | Approvals: reason and approver recorded; Branch Manager or Director only; offline requests queued | `requestApproval()` is the only way to create cancel, comp, discount or refund effects | Arch: only `core/approvals` writes `approval_decisions` | db tests | CHECK and trigger on `approval_decisions` (posting check, reason, not AI) |
| 8 | Branch and company walls | | | Starter tests 2 and 3; the global lint test over `pg_class` (every table with `company_id` has RLS enabled, forced and a policy; every view is `security_invoker`) | Row security, once per query |
| 9 | Company agrees with branch | | | | Composite foreign keys |
| 10 | Postings with dates; one branch at a time | | | | Exclusion constraint |
| 11 | No transfer with open orders or cash | | | db test (both modes) | Trigger on `postings` requires a clearance (§8) |
| 12 | Verification by Accountant or Director | | | db test | Policy plus trigger |
| 13 | Orders never lost or doubled | `Outbox` API is the only client write path | Lint: floor UI may not call `fetch` | Offline e2e (C(d)) | Unique ID; upsert on retry |
| 15 | Staff and customer separate | `StaffSession` and `CustomerSession` are separate types | Arch: customer entry cannot import staff features | | Separate tables and database roles |
| 16 | AI never approves | `Actor` union; `approve()` takes `StaffActor` only | | db test | CHECK `actor_type <> 'ai'` on decisions |
| 17 | Every table labelled | `tables` in `feature.ts` | | Starter test 2 | `core.register_table` |
| 18 | Floor write records a change in the same step | Handlers never see a connection, only `ctx.tx` | Arch: only `core/db` imports `pg` | Starter test 5 | **Trigger** writes `core.changes`, so it cannot be forgotten |
| 19 | One writer per branch | | | Cutover tests | Write policies include `branch_id = ANY (core.writable_branches())`; sync rejects a stale epoch |
| 20 | Floor reads company data only through the read-only copy | `ctx.read` exposes `selectFrom` only | Arch: floor features cannot import company features | Starter test 6 | Branch database: the app role has SELECT only on company tables |
| G7 | Printing never depends on the internet | `PrintDriver` interface | Arch: `core/print` cannot import `core/rpc` | Relay tests | |
| — | Feature isolation | | Biome GritQL plus `tools/arch`: imports only from `core/*` and other features' `public.ts` | | |

**How invalid states are made impossible:** the command runner is the only door (no connection reaches feature code). Tables are created by a function that adds the walls, the trigger and the label, so a "bare" table fails the starter test before review. Money cannot be a plain number. A floor handler's types do not include company tables for writing. The database refuses the remaining cases even if the types are bypassed.

### G6 defence: "the database trusts the app to say who is asking"

**Mechanism.**
1. **The database issues sessions.** `auth.login(identifier, secret, device_token)` is a `SECURITY DEFINER` function owned by `auth_owner`. It checks the bcrypt hash (pgcrypto `crypt`), the device and the postings, then returns a token: claims (`session_id`, `person_id`, `device_id`, `expires_at`, `kind`) plus an HMAC-SHA256 signature made with a key in `auth.keys`. The app role **cannot read** that table.
2. **Every transaction presents the token.** The runner calls `SET LOCAL app.session = '<token>'`. `core.actor()` (also `SECURITY DEFINER`) checks the signature, expiry, the `sessions` row (not revoked) and `suspensions`, once per transaction (cached in a transaction-local setting). With no valid token, every policy returns nothing and every write fails.
3. **The app role is narrow.** No `BYPASSRLS`, no table ownership, no DDL, no access to `auth.*`, INSERT-only on append-only tables. Migrations run as a separate `migrator` role used only by the deploy step.
4. **Step-up for sensitive acts.** Approvals, refunds, verification and posting changes require a `kind = 'stepup'` token less than 60 seconds old, minted by `auth.stepup(session, pin)`.

**What remains possible for a hijacked app server:** acting as any user whose token passes through it while it is compromised (tokens are bearer tokens in transit), and capturing PINs or passwords typed during that time. It **cannot** mint a session for someone who has not logged in, raise a user's postings, bypass the walls, alter audit or payment history, or approve without a fresh step-up.

**Limits and detection:** floor sessions last 12 hours and company sessions 8 hours, and both are bound to the device ID. Every audit row carries `session_id`. A nightly job flags commands whose session's device does not match the request's device header, sessions used from two IP ranges, and bursts of sensitive commands. The ops dashboard shows "sessions active now" per person, and the System Admin can revoke all sessions and rotate `auth.keys` in one action.

---

## 7. UI architecture and performance budget

**Shell and routes.** One SPA. `web/main.tsx` builds the router from every feature's `ui/routes.ts`. The navigation shows only routes whose `roles` match the user's postings. Each screen is a lazy chunk. The waiter's start screen is preloaded right after login. Role "apps" (waiter, station, manager, office) are route groups with their own layout, so a kitchen tablet never downloads office code.

**Components and tokens.** `core/ui/tokens.css` defines colour (light, dark for stations, and a high-contrast kitchen theme), spacing (4 px grid), radius, type scale, elevation and motion (durations 120, 200 and 320 ms; one easing). Tailwind reads only these tokens. The design-system session (I7) fills the values; the structure is fixed now. Primitives in `core/ui`: `Button` (pressed state within one frame), `Tile`, `List`, `Sheet`, `Toast`, `NumberPad`, `MoneyText` (formats `Shillings` only), `StatusDot` (saved, sending, waiting, flagged). Motion uses only `transform` and `opacity`, and respects `prefers-reduced-motion`.

**Screens that update instantly, then confirm.**

```
tap → command(input, retryKey)               client
    → floor-store applies the optimistic change (status: sending)   < 16 ms
    → outbox (IndexedDB) saves it                                   < 50 ms
    → POST /rpc/orders.place (800 ms timeout, then "offline" for 10 s, E1)
    → server transaction: row + change log + audit
    → SSE delivers the confirmed row (rev) → replaces the optimistic one (status: saved)
```

The phone outbox sends oldest first and sends requests after the orders they depend on. One refused item stays visible as "waiting" and never blocks the rest (E1).

**Live data.** The floor store holds today's open orders, tickets, bills, tables, menu, notices and staff names for the branch: hundreds of rows, a few hundred kB in memory. Kitchen and bar screens subscribe to their station's tickets with selectors, so a new ticket re-renders one tile, not the board. On reconnect: `GET /floor/stream?since=<last seq>` replays what was missed (E3). If the gap is too old (over 24 hours), the client takes a fresh snapshot.

**Long lists.** `core/ui/List` virtualises automatically above 60 rows, keeps row heights fixed where possible, and uses `content-visibility: auto` for cards. The Biome rule `noRawListMap` warns on `.map()` returning JSX inside screens over arrays typed `LiveRows`.

**Fonts and images.** System font stack for the floor, one subset variable font (at most 30 kB) for office screens; menu photos are optional and lazy, served as AVIF/WebP through Cloudflare cache with fixed dimensions.

**Print adapter (G7).**

```ts
interface PrintDriver { id: 'relay' | 'webserial' | 'fake'; supports(p: Printer): boolean; print(job: PrintJob): Promise<PrintOutcome> }
interface PrintJob { id: Uuid; printerId: Uuid; doc: PrintDoc; copy: boolean }   // id made on the device
function renderEscPos(doc: PrintDoc, profile: PrinterProfile): Uint8Array        // pure, snapshot-tested
```

- **Relay driver (main).** The phone posts the job straight to `https://print-<branch>.wendo.co.ke/jobs` on the branch LAN. The relay (`ops/relay`) owns each printer, queues jobs, ignores a repeated job ID, retries after a Bluetooth drop, and reports the outcome. The branch server is **not** in the print path, so receipts print even when it is down, and in cloud mode with the internet down.
- **Where the relay runs:** on the branch server if the printers are within Bluetooth range of it or use USB or LAN; otherwise on a small Linux box next to the printer (C5, Q1). In the cloud-served pilot, a relay box is required.
- **HTTPS for the relay:** a Let's Encrypt certificate obtained through the DNS-01 challenge with a Cloudflare API token limited to DNS edits (Caddy's Cloudflare DNS module). The public DNS record (DNS only, not proxied) points to the LAN IP. A local `dnsmasq` answers it when offline.
- **Web Serial (optional):** for a fixed PC with a USB printer, same `PrintJob`, same ESC/POS bytes.
- **Second device:** only the relay connects to the printer, so other devices queue behind it (I3a).

**Performance budget (checked in CI on every pull request).** "4x" means Chrome DevTools CPU throttling at 4x (the stand-in for a mid-range phone). Cloud profile: 150 ms RTT, 1.6 Mbps down. LAN profile: 5 ms RTT.

| # | Measure | Budget | Tool, failing the build |
|---|---|---|---|
| P1 | Initial JS for login plus the waiter start screen (gzip) | ≤ 130 kB | `size-limit` |
| P2 | Any single route chunk (gzip) | ≤ 35 kB (per-route value in `routes.ts`, default 25 kB) | `size-limit`, entries generated from `routes.ts` |
| P3 | CSS total (gzip) | ≤ 15 kB | `size-limit` |
| P4 | Cold load, cloud profile, 4x: Largest Contentful Paint | ≤ 2.0 s | Lighthouse CI |
| P5 | Warm load (service worker), 4x: interactive | ≤ 600 ms | Playwright trace |
| P6 | Tap to visible feedback, 4x (INP) | ≤ 100 ms p95; ≤ 50 ms target | Playwright event timing |
| P7 | Order saved locally (outbox committed), 4x | ≤ 50 ms p95 | Playwright marks |
| P8 | Order confirmed by server, cloud profile | ≤ 1 s p95 | e2e |
| P9 | Order to kitchen screen, cloud profile / LAN profile | ≤ 2 s / ≤ 300 ms p95 | e2e with two browser contexts |
| P10 | Scroll 1,000-row list, 4x | no long task over 50 ms | Playwright long-task observer |
| P11 | Waiter phone JS heap after 4 simulated service hours | ≤ 60 MB | Playwright plus CDP heap |
| P12 | Server: `orders.place` handler time | ≤ 30 ms p95 on CI Postgres | Vitest benchmark |
| P13 | Director board, 100,000 rows | ≤ 50 ms query | db benchmark (C(c): 18 to 30 ms) |

Estimate for P1: React and React DOM about 60 kB, router about 13 kB, Valibot 2 kB, floor store and outbox about 10 kB, shell and waiter screen about 30 kB. These are estimates to be replaced by the first CI measurement. Real phones are measured at the pilot (open risk G.1).

---

## 8. Branch server mode

**Floor and company split.**

| | Cloud | Branch server |
|---|---|---|
| Features mounted | All | `side: 'floor'` only, plus `core/sync` |
| Writes branch tables | For branches with `served_by = cloud` | Its own branch, only while it holds the lease |
| Company tables | Authority | Read-only copy (SELECT grant only), filled by sync |
| Who uses it | Everyone for company work; floor devices of cloud-served branches | Floor devices of its branch |

**Data placement.** Branch server: this branch's floor tables, plus copies of `menu`, `persons` and `credentials` (only people with a posting at this branch), `postings`, `role_permissions`, `notices` targeted at the branch, `settings`, `approval_votes` for its requests, `transfer_requests` for its people. Cloud: everything, including a copy of every branch's floor rows (which is what the Director board reads, with `last_synced_at` per branch).

**Sync protocol.**
1. Every change to a registered table fires the trigger, which inserts into `core.changes` in the same transaction (rule 18).
2. `wendo sync` (its own process on the branch, 81 MB in F) pushes unsent branch rows to `https://sync.wendo.co.ke` in batches of up to 500, oldest first. The cloud applies them in one transaction per batch as the `sync_apply` role (upsert by primary key where `rev` is newer), records the highest `seq` per branch, and replies. Only then does the branch set `sent_at` (F rule).
3. Pull: the branch holds a long-lived SSE connection for company changes relevant to it, filtered by a per-branch cursor; it applies them and acknowledges.
4. Each batch carries `epoch` and `schema_version`. The cloud refuses a stale epoch (§ cutover). Unknown columns are dropped and missing ones take their defaults (version window).
5. One bad row is put in `sync_quarantine` with the error and flagged on the dashboard; the rest continue (F).
6. Check after apply on the cloud: triggers (overpay, approvals) run again; a violation quarantines the row instead of blocking.
7. The same `core.changes` sequence drives `/floor/stream` for devices, so screens and the cloud read one list.

**Staff login without internet.** Devices are registered by a Branch Manager (a device token in the phone's storage and `devices`). Floor login is a person picker plus a 6-digit PIN, checked by `auth.login` in the branch database against the replicated hash. Five wrong PINs lock the person on that device for 15 minutes. Company login (password) happens in the cloud at `app.wendo.co.ke`.

**Suspended or ended staff at an offline branch (recommendation for I11).** A Branch Manager can **suspend locally, effective at once** (a `suspensions` row checked by `core.actor()` on every transaction). Floor sessions expire after 12 hours and are re-checked against local postings at every use. An HR change in the cloud reaches the branch within seconds when the link is up. During an outage, the HR change waits; the manager's local suspension covers the gap. Worst case if nobody suspends locally: access until the link returns or the 12-hour session ends.

**Notices.** HR and Director notices are company rows in the cloud. Sync delivers them to the branch within seconds. The branch server pushes them over SSE to the targeted phones as a banner, and they are stored for later reading. Manager notices at the branch are branch rows and reach phones at once, even offline. Web Push is used only as an extra when internet is up.

**Approvals while offline.** Requests are branch rows. A Branch Manager on site approves locally (the decision is written by the branch writer at once). A Director far away writes an `approval_vote` (company row) in the cloud. When it arrives, the branch writer turns the **first valid vote or local decision** into the single `approval_decision`. Later contradicting votes are recorded and flagged. While the branch is offline, remote votes wait, so the order stays "waiting for approval" (rule 14). In cloud mode, the cloud is the writer, and the same code turns a vote into a decision in the same transaction.

**Rule 11 across two machines (C8).**
1. HR starts a transfer in the cloud: a `transfer_requests` row (company).
2. It syncs to the branch. The floor handler `staff-floor/events/handlers.ts` checks locally: no open orders where the person is in charge, and no unsettled cash entries. If clear, it writes a `transfer_clearances` row (branch) with `cleared_at`, and from that moment the branch refuses new orders by that person (a check in `orders.place`). If not clear, it lists the blockers, and the manager can reassign orders or settle cash.
3. The clearance syncs up. The cloud trigger on `postings` refuses to end a floor posting at that branch unless a clearance exists for that person and branch from the current epoch.
4. **Offline:** the request simply waits, and HR sees "waiting for Kilimani (last synced 14:02)". The rule can be delayed, never broken. In cloud mode, steps 2 and 3 happen in one database within seconds.

**HTTPS on the branch network (G8).** Each branch has one floor name, `floor-<branch>.wendo.co.ke`, a first-level subdomain so Cloudflare's free certificate covers it in cloud mode.
- Cloud mode: public DNS proxied by Cloudflare to the cloud droplet.
- Server mode: the public record is switched to "DNS only" pointing to the branch server's LAN IP. The branch server holds a Let's Encrypt certificate for that name, obtained by DNS-01 through the Cloudflare API (no inbound port needed). It renews every 60 days, which needs internet for a few seconds. Caddy serves the app, the API and SSE on port 443.
- Offline name resolution: the router's DHCP hands out the branch server (or the relay box in the pilot) as the first DNS server. `dnsmasq` on it answers `floor-<branch>` and `print-<branch>` locally and forwards everything else. The second DNS server is public. Both give the same answer because the public record is changed at cutover.
- Phone setup: Android "Private DNS" off (it would bypass the local DNS); the setup checklist on the ops dashboard checks for this through a probe (Q6).
- **Same origin, same service worker:** the phone's cached app and its outbox carry over a cutover untouched.

**Last-resort layer (recommendation for I11).** In server mode, when the branch server or Wi-Fi fails:

| Failure | What happens | Recommendation |
|---|---|---|
| Branch server down, Wi-Fi up | Phones keep taking orders into their outbox (the same code as cloud mode, at no extra cost); receipts still print through the relay; kitchen screens stop updating | **Keep the phone-side queue.** Waiters walk tickets to the kitchen. If it lasts more than 10 minutes and the internet is up, use manual failover to the cloud |
| Wi-Fi down | Nothing on the network works | **Numbered paper pad** per waiter; entered later by the manager with "entered late" flagged |

**Failover policy (recommendation for I11): manual, one button, fenced.** The owner or a Director presses "Serve Kilimani from cloud" on the ops dashboard. The cloud raises the epoch, takes the lease and switches the public DNS record back to the cloud. The router DNS is told through the agent if reachable; otherwise `dnsmasq` on the dead server is irrelevant and phones use the public answer. When the branch server returns, it sees a higher epoch, becomes read-only and pushes any unsent rows from its old epoch. Rows are mostly new orders with device-made IDs and are accepted as "late arrivals". A row changed in both epochs keeps the newer epoch's version and flags the older one for the manager. Automatic failover is rejected: a split network could produce two writers.

**Cutover to a branch server and back (G14, W5).**

| Step | Cloud → branch server | Branch server → cloud |
|---|---|---|
| 1 | Install from image, enrol (`wendo-install kilimani`); the agent reports healthy | Press "Serve from cloud" (planned, both sides up) |
| 2 | Branch server pulls a full snapshot of company copies and the branch's rows (follower, no lease) | Branch server sets `state = draining`, stops accepting writes (lease removed), pushes the rest |
| 3 | Dashboard: "ready" once lag is 0. Press "Cut over" at a quiet time | Cloud confirms highest `seq` = branch's, raises the epoch, takes the lease |
| 4 | Cloud sets `draining`: removes its lease for the branch (writes now refused by the database), waits until the branch server has applied everything (seconds) | Public DNS → cloud (TTL 60 s); local `dnsmasq` record removed |
| 5 | `epoch + 1`, the branch server takes the lease; public DNS → LAN IP; `dnsmasq` record on | Phones retry from their outbox; no data is lost |
| 6 | Phones that tried to write during the switch got "not writer, retry" and kept the item in their outbox; they now reach the branch server | Branch server stays as a warm follower or is switched off |

Writes for the branch are refused for about 5 to 30 seconds during the switch (estimate), and phones queue them during that time.

**Update compatibility (G14).** See W10 in §11. Rules: migrations expand only (add tables, nullable columns or columns with defaults, new functions under new names); contract steps (drop, rename) wait in `migrations/contract/` until the dashboard shows every branch server at or beyond the required version; the cloud accepts sync protocol versions N and N-1; the cloud sends a branch only the columns that branch's schema version knows.

---

## 9. Hosting and operations

| Item | Design |
|---|---|
| Production droplet | 2 vCPU / 4 GB to start (estimate for 10 branches; Q2): Caddy (TLS with a Cloudflare Origin CA certificate), `wendo serve --mode=cloud` (web, API, SSE, sync endpoint), `wendo worker`, Postgres 16 tuned (shared_buffers 1 GB). Systemd units. Daily DigitalOcean droplet snapshots as an extra |
| Staging droplet | Same layout, smaller, seeded with fake data; receives every merge to `main` |
| Cloudflare | DNS for `app.`, `staging.`, `sync.`, `floor-<branch>.` (proxied in cloud mode), `print-<branch>.` (DNS only, LAN IP). Cache rule: `/assets/*` cached 1 year (hashed names); `index.html` and `sw.js` not cached; `/rpc/*`, `/floor/stream`, `/sync/*` bypass. SSL Full (strict). Free managed WAF rules. One rate-limit rule on `/rpc/auth.login`. SSE gets a 20 s heartbeat to stay under the proxy's idle timeout (to verify, I5) |
| Database backups (cloud) | pgBackRest: continuous WAL archive and a nightly full backup to DigitalOcean Spaces (Q3), 30-day retention |
| Branch backups | The cloud copy is the first backup (seconds behind). Also a nightly `pg_dump` kept on the machine (7 days) and uploaded to Spaces |
| Restore drills | A weekly job on staging restores the latest cloud backup and one branch dump, runs row-count and checksum checks against the source, and posts the result to the dashboard. A drill older than 8 days raises an alert |
| Deploys | `release.yml`: build once, one tarball, signed (minisign). Staging automatically → production cloud (manual approval) → canary branch → remaining branches |
| Monitoring | Agent heartbeat every 30 s (CPU, memory, disk, Postgres size, sync lag, last backup, relay and printer status, UPS through NUT, certificate expiry, clock drift). Cloud health from the same agent running on the droplet |
| Alerts | Telegram or email (Q5) when: heartbeat missing 2 minutes, sync lag over 5 minutes with internet up, quarantined rows, backup or drill missing, disk over 80 %, certificate under 20 days, printer offline during opening hours, rollback happened |

**Remote tooling for branch machines (G13).**

| Need | Mechanism |
|---|---|
| Health and last-sync dashboard | `features/branch-ops` (company feature): one row per branch with served-by, epoch, version, sync lag, last heartbeat, backup and drill status, printers |
| Alerts | `branch-ops/jobs/evaluate-alerts.ts` every minute |
| Backups and restore drills | As above; the "restore drill" button also runs on demand |
| Staged updates with automatic rollback | The agent downloads the release, checks the signature, unpacks to `/opt/wendo/releases/<ver>`, runs expand-only migrations, switches a symlink, restarts, and runs health checks (HTTP, a test order in a rolled-back transaction, sync round trip). If any check fails within 10 minutes, it switches the symlink back. The database is never rolled back; expand-only migrations make the old binary safe on the new schema |
| Secure remote access | Tailscale on every machine (SSH only through the tailnet, no open ports), or self-hosted Headscale on the production droplet (Q4) |
| Pre-imaged spare | A spare mini PC flashed from the CI image. `wendo-install kilimani --replace` pulls the branch's state from the cloud and takes the lease through the normal cutover |
| Remote power cycle | Hardware watchdog (systemd `RuntimeWatchdogSec`) reboots a hung machine; BIOS "power on after AC loss"; optional smart plug |
| Branch machine footprint | Postgres about 1.2 GB (shared_buffers 1 GB), app 150 MB (F: 91 MB), sync 81 MB (F), Caddy 40 MB, agent 50 MB, relay 50 MB: about 1.6 GB of 8 GB (estimate) |

**CI jobs** (GitHub Actions, cancel superseded runs, cache pnpm):

| Job | Runs | When |
|---|---|---|
| `check` | tsgo, Biome, `tools/arch`, `lint-sql`, all Vitest unit tests | every push |
| `db` | All database tests against a Postgres service, including the global row-security lint test | every push |
| `build` | Vite build plus server bundle; `size-limit` (P1 to P3) | every push |
| `perf` | Playwright plus Lighthouse CI on the production build, 4x CPU (P4 to P11) | pull requests |
| `e2e` | Cross-feature journeys, offline journey, two-context kitchen test | pull requests |
| `compat` | New cloud image against the N-1 branch image: sync both ways, replay a fixture day | pull requests touching `data/`, `core/sync`, `core/db` |
| `image` | Branch image and release tarball | tags |

---

## 10. Agent experience

**Finding code without documents.**
- Folder names are glossary words (`orders`, `payments`, `approvals`). One feature is one folder with the same rooms.
- `pnpm features` prints the registry from every `feature.ts`: name, side, tables, permissions, commands, routes and events. An agent asks "who writes `bills`?" and gets one line.
- Startup and lint errors name the fix ("table `loyalty_ledger` has no scope label: add it to `tables` in features/loyalty/feature.ts").

**Verification loops.**

| Tier | Command | What runs | Time |
|---|---|---|---|
| ~10 s | `pnpm check <feature>` | tsgo (whole project, incremental), Biome on the feature, `tools/arch` for the feature, domain tests | about 3 to 6 s (estimate from A1: 0.26 s, 0.22 s and 0.7 s per 200 modules, scaled for a larger tree) |
| ~30 s | `pnpm check <feature> --db` | Above, plus the feature's database and starter tests against the agent's own database (cloned from the template, 0.4 s, A5) | about 10 to 20 s (estimate) |
| Screen check | `pnpm dev` and `agent-browser` on the shared Chromium | Vite dev (about 61 MB) plus the server in Node | on demand, full mode only |
| Full | CI | Everything in §9 | minutes |

**Memory plan (G9).** Numbers from decision log A, upper values; my estimates are marked.

| Item | MB | Source |
|---|---|---|
| Shared: Postgres (one server, a database per agent) | 400 | A2 upper |
| Shared: one Chromium | 550 | A2, A3 |
| Shared: portless proxy | 75 | A2 |
| **Shared total** | **1,025** | |
| Per full agent: coding agent | 415 | A2 upper |
| Vite dev server on the real app | 150 | estimate (2.5x A4's 61 MB) |
| API server in Node (with the worker inside) | 120 | estimate (F: 91 MB) |
| Type check or test burst | 250 | A1 (250 MB PSS for the full test run) |
| portless wrapper | 73 | A6 |
| Browser session | 140 | A3 |
| **Per full agent** | **1,148** | |
| Per light agent: coding agent plus test burst | 665 | A2, A1 |

- **3 full agents:** 1,025 + 3 × 1,148 = **4,469 MB** (2.5 GB spare from about 7 GB).
- **6 light agents:** 400 (Postgres only) + 6 × 665 = **4,390 MB**; with Chromium kept open, 4,940 MB.
- **Mixed (2 full and 3 light):** 1,025 + 2,296 + 1,995 = 5,316 MB.
- Full tests and production builds run only in CI.

**Avoiding collisions between parallel agents.**

| Risk | Rule |
|---|---|
| Two agents, two features | Nothing central to edit: features, routes, permissions, grants, jobs and migrations live in feature folders |
| Migration order | Timestamped names; CI fails if a PR's migration is older than the newest on `main`, and `pnpm new-migration --rebase` renames it; the template database is rebuilt when the hash of all migrations changes |
| Shared types | Cross-feature types live only in each feature's `public.ts`; changes to `core/` go in their own small PR, merged first |
| Ports, URLs, databases | Worktree per agent; `portless` names and `wendo_<agent>` databases assigned by `pnpm agent:init` |

---

## 11. Worked tasks

Starting state per C3. Paths are relative to the repository root. "Gen" marks files produced by a generator.

**W1 Loyalty points** (floor feature; earn on a closed sale; redeem as a discount through the existing approval flow)
| File | |
|---|---|
| `features/loyalty/feature.ts` | gen; side floor; tables `loyalty_enrolments:branch`, `loyalty_ledger:branch`; handles `payments.saleClosed` |
| `features/loyalty/domain/points.ts` | earn and redeem rules (pure, `Cents`) |
| `features/loyalty/data/schema.ts` | gen |
| `features/loyalty/data/migrations/20270110T0900_loyalty.sql` | gen, then filled |
| `features/loyalty/api/contract.ts` | `enrol`, `lookup`, `redeem` |
| `features/loyalty/api/handlers.ts` | `redeem` calls `orders` public `requestDiscount` with reason `loyalty` |
| `features/loyalty/events/handlers.ts` | on `saleClosed`, append ledger points |
| `features/loyalty/jobs/merge-members.ts` | cloud job: merges enrolments with the same phone number (detect and flag) |
| `features/loyalty/ui/routes.ts`, `ui/screens/MemberLookup.tsx` | gen, then filled |
| `features/loyalty/public.ts` | gen |
| `features/loyalty/tests/domain/points.test.ts`, `tests/db/ledger.test.ts`, `tests/starter.test.ts` | starter gen |
**Total 14 files, all in one folder.** Stopped from: writing to `bills` (not in its typed `tx`), skipping approval for the discount, using floats.

**W2 Service charge** (reverses B6; per-branch rate setting; applied after discount, before tip; Q8 for tax treatment)
`features/payments/feature.ts` (setting `serviceChargeBp`), `features/payments/domain/billing.ts`, `features/payments/domain/split.ts`, `features/payments/data/migrations/<ts>_service_charge.sql` (adds `service_charge_cents guest_cents NOT NULL DEFAULT 0`, expand-only), `features/payments/data/schema.ts`, `features/payments/ui/parts/BillSummary.tsx`, `features/payments/print/receipt.ts`, `features/payments/tests/domain/billing.test.ts`, `features/payments/tests/domain/split.property.test.ts`, `features/payments/tests/print/receipt.test.ts`. **Total 10.** Stopped from: storing it as a float or as cents a guest cannot pay (the `guest_cents` domain); breaking split totals (the property test); breaking older branch machines (the column has a default).

**W3 Store Attendant counts stock**
`features/inventory/feature.ts` (add `grants: { store_attendant: ['counts.record', 'stock.read'] }`), `features/inventory/ui/routes.ts` (add the role to the count screen), `features/inventory/tests/db/attendant.test.ts` (can record counts; cannot change prices, approve or see payroll). **Total 3.** Grants load at boot, so no migration and no central role file. The starter test checks every other permission is still refused.

**W4 Waiter's table map** (orders feature; positions editable by the Branch Manager)
`features/orders/data/migrations/<ts>_table_positions.sql` (`x`, `y`, `shape`, nullable), `features/orders/data/schema.ts`, `features/orders/api/contract.ts` (`tables.setLayout`, `orders.layout.edit` permission), `features/orders/api/handlers.ts`, `features/orders/feature.ts` (new permission and grant), `features/orders/ui/routes.ts` (gen entry with budget 25 kB), `features/orders/ui/screens/TableMap.tsx` (gen), `features/orders/ui/parts/TableTile.tsx`, `features/orders/tests/db/layout.test.ts`. **Total 9.** The map reads tables and open orders from the floor store (live, offline). The route budget fails CI if the screen grows past 25 kB.

**W5 Cut a branch over and back.** **0 code files.** Ops dashboard steps in §8. Tests that already guard it: `core/sync/tests/cutover.test.ts` (two databases, writes refused on the old writer, nothing lost or doubled) and the CI `compat` job. Stopped from: two writers (the database write policy uses the lease; sync refuses a stale epoch).

**W6 New printer type** (example: a LAN ESC/POS printer on port 9100)
`core/print/printer-types.ts` (type, paper width, code page, transport), `ops/relay/transports/tcp9100.ts`, `ops/relay/transports/tcp9100.test.ts` (against a fake socket server). **Total 3.** If it speaks a different command language: add `core/print/encoders/<name>.ts` and its snapshot test (5). `printer_types` is seeded from code, so no migration.

**W7 Customer ordering app** (new identity, new channel, own origin `order.wendo.co.ke`)
| File | |
|---|---|
| `core/auth/customer-session.ts`, `db/roles/customer_app.sql` | separate session type and database role |
| `features/customers/feature.ts`, `data/migrations/<ts>_customers.sql`, `data/schema.ts`, `api/contract.ts`, `api/handlers.ts` (phone OTP), `tests/db/isolation.test.ts`, `tests/starter.test.ts` | company feature; customers see only their own rows |
| `features/online-orders/feature.ts`, `domain/basket.ts`, `data/migrations/<ts>_online_order_requests.sql`, `data/schema.ts`, `api/contract.ts`, `api/handlers.ts`, `events/handlers.ts` (branch writer turns a request into an order with `channel = 'customer'` and no waiter), `ui/routes.ts`, `ui/screens/Menu.tsx`, `ui/screens/Basket.tsx`, `ui/screens/Track.tsx`, `tests/domain/basket.test.ts`, `tests/db/request-to-order.test.ts`, `tests/starter.test.ts` | |
| `web/customer/index.html`, `web/customer/main.tsx`, `size-limit` entry in `web/customer/budget.json` | separate entry and budget |
| `ops/deploy/caddy/cloud.Caddyfile` | new hostname |
**Total 27.** Payment for online orders (M-Pesa integration) is a separate later task. Stopped from: customers seeing staff data (separate role and RLS; arch rule: the customer entry cannot import staff features); the cloud writing a branch's order directly (requests are company rows that the branch writer converts).

**W8 Bug in split-bill rounding**
`features/payments/domain/split.ts`, `features/payments/tests/domain/split.test.ts` (a named regression case first, failing), and if found by the property test, its seed in `features/payments/tests/domain/split.property.test.ts`. **Total 2 or 3.** `pnpm check payments` runs in seconds with no database or browser. Stopped from: a float slipping in (`Cents` type), list-order dependence (property test shuffles input).

**W9 HR notice reaches a waiter during an internet outage.** **0 files** (behaviour of `notices` and `core/sync`). HR posts at 10:00 in the cloud; the branch's link is down. The notice waits in the cloud's change log for that branch. Floor work at the branch continues. When the link returns, the sync pull delivers it within seconds; the branch server pushes it over SSE to waiters on shift; it shows as a banner, and the read receipt syncs back to HR. HR sees "delivered to branch at 10:47" and "read by 3 of 5". If it cannot wait, the Branch Manager posts a local branch notice, which reaches phones at once. Guarded by `features/notices/tests/db/delivery.test.ts`.

**W10 Ship an update without breaking older branch servers**
1. The change uses expand-only migrations (`lint-sql` refuses DROP, RENAME, NOT NULL without a default, or changed function signatures outside `migrations/contract/`).
2. If the sync shape changes, `core/sync/version.ts` bumps the protocol, and the cloud keeps a reader for N-1 (`core/sync/compat/v<N-1>.ts`).
3. CI `compat` runs the new cloud against the previous branch image.
4. Release: staging → production cloud → canary branch (auto-rollback on failed health) → the rest, one at a time, at a quiet hour.
5. Contract migrations run in a later release, after the dashboard shows every branch at the required version.
Files: the feature's own files plus, if the protocol changes, `core/sync/version.ts` and `core/sync/compat/v<N-1>.ts`. Stopped from: removing a column an older branch still sends (lint), a branch running a version the cloud no longer reads (the cloud refuses with a clear "update required" alert, and the branch keeps working locally).

---

## 12. Stress-test

| Scenario (domain-model §7 plus C6) | What changes in this design |
|---|---|
| Customer app, accounts, delivery | New features `customers` and `online-orders`, own entry and role (W7). Orders already allow no waiter and carry `channel` |
| Loyalty | New feature (W1) |
| Reservations | New feature `reservations` (floor; bookings reference `tables`). Online bookings are company requests converted by the branch writer, as in W7 |
| Delivery apps | New feature; `orders.external_ref` unique per channel (expand migration in `orders`); bulk payouts in finance |
| QR self-ordering | Same pattern as W7, with a guest actor (no account); a new route group with its own budget |
| Second company | Already isolated by `company_id` and RLS; `branches.currency` exists; add a company switcher for the System Admin |
| AI: ask-anything | Feature `ai-ask` (company) runs read-only queries under the asker's session token, so RLS limits it to the asker's postings |
| AI: acts for people | `actor_type = 'ai'` with `acting_for_person_id`; a separate AI session kind minted by `auth` for the asker |
| AI: reconciliation | Feature `reconciliation` (company): deterministic matching first, AI on leftovers; payments already carry method and settlement account |
| AI: anomaly detection | Job in a company feature over synced events; reasons are already required |
| AI: drafts | Records with `status = 'draft'` created by the AI actor; humans confirm through a normal command |
| AI: deeper automation | Same: the AI acts through commands with its own actor; the approval check refuses it |
| Branch server in each branch | Built in from day one (§8) |
| **KRA eTIMS (C6)** | New feature `etims` with a print hook: receipts print offline as today and are queued for signing; the signed fiscal number is attached when the link returns; a reprint shows it. If KRA requires signing **before** printing, a local signing device at the branch (an eTIMS device behind the relay) is the change. `bills` already holds tax category and receipt number |

---

## 13. Weaknesses and risks

| Risk | What would make me wrong | Mitigation | Cost to change later |
|---|---|---|---|
| **DNS-based "same address, two places"** (biggest) | Phones with Private DNS on, routers that ignore the DHCP DNS setting, or long DNS caches send phones to the cloud while the branch server is the writer | Both public and local DNS give the same answer after cutover; the cloud refuses writes for a server-served branch with "not writer, retry", so phones queue rather than lose data; a probe on the setup checklist; MDM setting for Private DNS | Medium: the fallback is a separate floor origin per mode, at the cost of the outbox migrating across origins |
| Chrome Local Network Access prompt | In cloud mode, the page is public and the relay is private; Chrome asks for "local network" permission once per origin (from my knowledge of Chrome's 2025 change; not checked today) | Grant it during phone setup; in server mode the page itself is local, so no prompt | Cheap |
| Own sync, outbox, floor store and RPC (about 1,500 lines) | Bugs in our own core code | Proven shape (F, C(d)); heaviest tests in core; property and fault-injection tests in CI | Medium |
| Let's Encrypt certificate lifetimes getting shorter | Renewal needs internet; a long outage near expiry | Renew at 30 days left; alert at 20 days; a long outage can always fall back to the cloud | Cheap |
| Client-only rendering | First visit on a poor link is slower than server rendering would be | 130 kB budget, inline skeleton, Cloudflare cache; most loads are warm | Hard (framework change) |
| Two data patterns | Agents mixing them | Lint per side | Cheap |
| Bundle estimates | React and the router may exceed my estimates | Budgets fail CI; Preact compatibility is a last resort only if the owner allows it (React is fixed) | Cheap |
| Printers unknown (C5) | Bluetooth Classic on the chosen relay box may be flaky | The I3a experiment before the pilot; transports are swappable (W6) | Cheap |
| Manual failover | The owner unreachable during an outage | The Director also holds the button; the phone queue plus paper pad covers the gap | Cheap |
| Branch machine variety | 8 GB mini PCs differ | One image, CI-built, tested on the spare | Cheap |
| Tailwind arbitrary values | Off-token styling creeps in | A check bans arbitrary values | Cheap |

**Hard-to-change decisions** (decide carefully now): the change-log trigger and epoch model, the session-token mechanism, the expand-only migration discipline, the `cents`/`guest_cents` domains, and the client-only SPA. **Cheap ones:** component library, router, worker library, alert channel, remote-access provider.

---

## 14. Questions for the owner

| # | Question | My recommendation |
|---|---|---|
| Q1 | Printer models and where each sits (I12)? This decides whether the relay runs on the branch server or on a small box by each printer | Record them before the pilot; buy one relay box for the pilot |
| Q2 | Droplet size budget for production and staging? | Production 2 vCPU / 4 GB, staging 1 vCPU / 2 GB to start (estimate) |
| Q3 | May backups go to DigitalOcean Spaces (object storage, not a droplet)? | Yes: off-machine backups are needed and Spaces is the same provider |
| Q4 | Remote access: Tailscale (hosted, free tier to verify) or self-hosted Headscale on the droplet? | Tailscale for simplicity |
| Q5 | Alerts by Telegram, email or SMS? | Telegram (free, instant) |
| Q6 | Can every waiter phone have "Private DNS" off, and can you change each branch router's DNS setting? | Yes; the ops checklist enforces it |
| Q7 | Last resort when the branch server or Wi-Fi is down: phone queue plus paper pad? | Yes (§8) |
| Q8 | If a service charge is ever added: is it taxed, and does it count in splits by food share? | Ask the accountant |
| Q9 | Loyalty redemptions: may they skip manager approval up to a limit? | Keep approval at first |
| Q10 | Failover: may a Director also press "Serve from cloud", or only you? | Both |
