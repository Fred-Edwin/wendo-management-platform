# Decision log: Session 2 (domain, stress-test, experiments)

Each entry: the decision, the alternatives, the reason, the risk, and the evidence. Experiments were throwaway and have been deleted. Only their answers are kept here. Edit freely.

## A. Laptop, tools and agent budget

| # | Decision | Alternatives | Reason and evidence | Risk |
|---|---|---|---|---|
| A1 | **Fast loop is comfortably inside the targets.** Type check 0.26 s, lint 0.22 s, tests for one changed file 0.7 s, all 200 test files about 15 s. | n/a | Measured on a 200-module synthetic project. Whole-tree memory: 129 / 119 / 195 MB, and 250 MB (PSS) for the full test run. | A real codebase is larger. We have about 30x headroom. |
| A2 | **Memory budget: at most 3 agents in full mode, about 5 to 6 in light mode.** | More agents | About 7.2 GB free at the time. A coding agent process costs 290 to 415 MB. Light mode about 0.7 GB per agent, full mode about 1.5 GB. Shared: Postgres 160 to 400 MB, one browser 550 MB, proxy 75 MB. | A real app dev server may be heavier than the test page. |
| A3 | **One shared full Chromium for agents. Lightpanda is dropped.** | A full browser per agent (about 320 to 560 MB each); Lightpanda (about 20 MB each) | Lightpanda could not run a React page's click handler (Orders stayed 0; full Chrome gave 2). Playwright adds about 140 MB per extra isolated session. | agent-browser sessions sharing one Chromium (`--cdp`) held memory flat, but I did not prove they are isolated. |
| A4 | **Dev-server weight is a Session 3 input.** | Vite 61 MB, 0.18 s start. Next.js 610 MB for a trivial page, 5.5 s first compile. | Measured. A 10x memory gap matters with 3 agents on 7 GB. | Next.js dev mode is heavier on a real app. |
| A5 | **One shared database server, one database per agent, cloned from a template.** | A server per agent | Postgres idle 20 MB; 20 databases 156 MB; a 100k-row clone takes 0.4 s. | First container start was 28 s on a busy laptop. |
| A6 | **portless over plain HTTP on a high port** gives each agent a named URL. | Fixed ports; HTTPS on port 443 (needs sudo) | Worked with two named apps, about 73 MB per wrapper. Install scripts were not needed. | Pre-1.0. |
| A7 | **opensrc works** for reading dependency source. | n/a | First fetch 10 s, cached 0.06 s. It caches in `~/.opensrc`. | Experimental. |

**Pinned versions tested** (install the same ones, then upgrade deliberately): Node 26.8.1, pnpm 11.3.0, TypeScript 7.0.2, Biome 2.6.4, ESLint 10.14.0 (installed, not measured; the chosen architecture keeps it out of the fast loop and uses Biome plus an import-graph test, see J), Vitest 5.1.0, Playwright 1.64.1, Chromium 152.0.7977.82 (system), agent-browser 0.41.2, portless 0.16.0, opensrc 0.7.3, Next.js 16.4.5 with React 19.3.1, Vite 8.3.5, `pg` 8.23.1, PostgreSQL 16.15 (Docker image `postgres:16-alpine`). agent-browser, portless, opensrc and webreel come from the `vercel-labs` GitHub organisation (checked from npm metadata; source code not audited). Their install scripts were not run. webreel was not installed.

## B. Domain decisions

| # | Decision | Alternatives | Reason | Risk |
|---|---|---|---|---|
| B1 | **Roles are "postings" (called "hats" until 2026-11-04, see K-11): role + scope (branch or company-wide) + start and end dates.** Floor staff hold one branch at a time. Branch managers can hold several. Accountant, HR, Director and System Admin are company-wide. | One home branch per person; global roles plus a branch list | Fits a manager of two branches and keeps transfer history. | Postings are company-wide data, so a branch server (adopted in I9; see F and J) sees them late. |
| B2 | **Payments are append-only. Status is worked out from history.** | A "paid" flag on the order; a full accounting ledger now | A late or repeated M-Pesa message can't corrupt anything. Accounting waits for the finance feature. | None found. |
| B3 | **Pending payments count as paid.** The guest can leave. The accountant verifies against the M-Pesa statement. Branch Manager, accountant and Director may reject or flag. | Hold the order open until verified; manager verifies | Owner decision. Keeps service moving. | A fake code is accepted until checked. Mitigated by rejection, one use per code, and anomaly detection later. |
| B4 | **M-Pesa integration is deferred.** First phase records payments by hand. The idempotent-callback rule stays for when the SDK arrives. | Integrate from the start | Owner decision. The table already accepts automatic confirmations. | None found. |
| B5 | **Money is whole cents. Guest-paid amounts are whole shillings.** | Whole shillings everywhere | Bank statements, supplier invoices, recipe costing and VAT may carry cents. Changing the unit later would rewrite every money column. | 100x mix-ups. A `Money` type should prevent this. |
| B6 | **No service charge.** | Optional setting | The client did not ask. It came from the example restaurant in the workflow document. | Small addition if asked for later. |
| B7 | **"Void" is renamed "Cancel".** Cancel is before payment, Refund after. | Keep "void" | Owner decision. | The word is also used for reservations. Keep the objects distinct. |
| B8 | **One waiter per order. Bills can be split.** One guest may pay by several methods. The waiter picks food share or equal split each time. | Several waiters per order; no splits | Owner decision. | Split math is hard to get exactly right. It is covered by 100,000 random bills. |
| B9 | **No transfer while a person has open orders or unsettled cash.** A department head can reassign an open order. | Move orders automatically | Owner decision. | Reassign is not limited to changing the waiter. It should be a dedicated action. |
| B10 | **Approvals: Branch Manager (own branch) or Director only.** No acting manager. A Director covers an absence. | Temporary acting manager | Owner decision. | Service can stall when both are unavailable, as the brief warned. |
| B11 | **Numbering:** online plain per branch per day, offline device-letter prefix. Receipt number is separate from order number. A clash keeps both orders and flags it. | Reserved number blocks per device; gapless offline | Cannot clash even offline; matches "online by default". | Gaps can appear in the offline series. Confirm with the accountant. |
| B12 | **VAT is included in menu prices.** | Add on top | Owner confirmed. | Confirm VAT registration and any eTIMS need with the accountant. |

## C. Experiment results

| Experiment | Question | Answer | Tests | Deliberate breaks caught |
|---|---|---|---|---|
| **(a) Billing** | Does money always add up in whole numbers? | Yes. 100,000 random bills, splits, shared items, several methods. A bug with an all-comped bill and a repeatability bug with ties were found and fixed. | 17 | 5 of 5 |
| **(b) Payments** | Do duplicate, late, early, wrong and missing M-Pesa messages, and manual entries, stay correct? | Yes. A gap was found and closed: re-verifying a rejected payment could overpay a bill. Person-verification added. | 24 | 7 of 7 |
| **(c) Branch walls** | Can a Branch A user ever read Branch B? | No, in 33 tests: postings, transfers on given dates, company boundary, joins, counts, views. A leaky view trap was shown. First version was 15 to 35 times slower on bulk reads; set-based policies reduced a 100,000-row board from 364 to 471 ms to 18 to 30 ms. | 33 | 20 of 20 |
| **(d) Offline** | Can a phone take orders with the network off and sync without duplicates? | Yes. 3 to 7 ms tap to ticket offline. App loads offline. Lost replies, same letter by mistake, stale menu and offline cancel all handled. Two real bugs found and fixed (head-of-line blocking; rejecting a real sale). | 12 | 11 of 11 |
| **(e) Speed** | Order to kitchen and taps vs the targets? | Server push 63 to 265 ms; polling 439 to 522 ms (worst 1,237). Waiter sees ticket 34 to 438 ms. Tap 20 to 30 ms. Cold load 370 to 520 ms. All inside the targets. | measured | n/a |
| **(f) Branch server** | Do operations survive a dead internet and sync back exactly once? | Yes. See section F. | 14 | 10 of 10 |

In every suite the first round of deliberate breaks revealed gaps in my own tests. Each was fixed and re-proven. **Proof method to keep:** break the rules on purpose and confirm a test fails.

**Caveats for (e):** desktop Chrome, not a phone. CPU slowdown only approximates a mid-range phone. The test page is tiny. Networks are my assumptions, not Kenyan measurements. Page load must be measured on the real app's production build in CI.

## D. Database rules and tools

| # | Decision | Reason | Risk |
|---|---|---|---|
| D1 | **Branch and company walls live in the database** (row security), checked once per query, driven by postings and an explicit date. | Protects against forgotten filters and AI-written queries. Same rules run on any copy of the database. | **The database trusts the app to say who is asking.** A hijacked app server could impersonate someone. Session 3 should consider signed session tokens verified by the database. |
| D2 | **Lint tests are mandatory:** every table with a company ID has row security enabled, forced and a policy; every view is `security_invoker`. | A normal view leaks across companies. | None found. |
| D3 | **Company ID is tied to its branch by composite foreign keys.** | Two layers: permissions, then keys. | None found. |
| D4 | **Some functions bypass the walls on purpose.** They need review. | Needed to evaluate permissions. | A mistake there is a leak. |

## E. Offline and kitchen delivery

| # | Decision | Reason |
|---|---|---|
| E1 | **Phone design:** device-made IDs and retry keys; a phone-side queue; the app cached on the phone; an 800 ms timeout that then remembers "offline" for 10 s; orders sync oldest first and requests after their orders; one failing item never blocks the rest. | Black-hole network: first order 806 ms, then 1.4 ms. A refused item stays visible as waiting. |
| E2 | **Kitchen delivery: server push.** Push through the database costs 1 to 6 ms more and keeps several server processes possible. Polling is 4 to 7 times slower and adds constant load. | Measured in (e). |
| E3 | **Kitchen screens must catch up after reconnecting** by asking for anything missed. | Found necessary in (f). |

## F. Branch server (on-prem): planned for every branch after the pilot

*Explored as a what-if in Session 2. The owner then decided (section I, I9) to plan it for every branch. The results below show the design works in miniature.*

- **Outcome: planned for every branch, rolled out per branch after the pilot (I9).** The owner's experience of the old system failing on unreliable internet outweighed the brief's earlier "no on-site hub". The floor and company split must be designed in from day one. The cloud mode stays for development, staging, the pilot and fallback.
- **Proven in miniature:** with the link to the central server dead, phone to kitchen stays at 33 ms. 4,600 queued events arrived exactly once in 5.0 s while the kitchen stayed at 33 ms median (51 ms worst). Four random fault schedules converged in 0.4 to 1.0 s. Killing the sync program or the branch server loses nothing and doubles nothing. Two hops (phone, branch server, central server) are safe. A branch cannot send or receive another branch's records. One bad record doesn't block the rest. Footprint: server 91 MB, sync 81 MB, database about 200 MB.
- **Rules that must hold:** the sync event is saved in the same step as the business change; a record is marked sent only after the central server confirms; every record is either branch-scoped or company-wide, never both.
- **What weakens:** cross-branch uniqueness becomes "detect and flag" (M-Pesa code at two branches); a race here was found and fixed with a one-row-per-code registry on the central server. **Permissions go stale during an outage**: an ended posting stays valid at the branch until the link returns. **Open policy question for the owner.**
- **Not tested:** the transfer guard across two databases; a Director approving remotely while a branch is offline; real hardware; real link speed (estimate 15 to 40 s to drain that volume on typical to poor mobile data); the cost of running ten machines.

## G. Open risks (deferred or unresolved)

1. **Bluetooth printing from a web app** and **real Android phone speed** (deferred, no hardware). The app shape is settled as web only (I1, I2); what remains is proving the print relay and phone speed on hardware.
2. **Whether a branch server removes enough outage pain to justify its cost.** Measure the pilot branch's real outage minutes before and after.
3. **The database trusts the app server** to say who is asking (D1).
4. **Staleness of permissions on a branch server** (F).
5. **Department-head reassign** is not limited to changing the waiter.
6. **Phone clock skew:** an offline order's business day comes from the phone's clock.
7. **Page load** on the real production build can only be measured in CI.
8. **VAT registration and eTIMS**, and **payroll deductions**: confirm with the accountant.
9. **AI provider data rules** (separate session). AI-actor rules are design only.
10. **Gapless receipt numbers offline:** confirm with the accountant.
11. **A fake M-Pesa code** is accepted until verified.
12. **Agent-browser isolation** over a shared Chromium is unproven.

## H. For Session 3 (architecture competition)

**Inputs:** product brief, this log, `domain-model.md`, the pinned tool list (A), the memory budget (A2), and the 13 stress-test scenarios.
**Preferences that emerged** (not decisions): PostgreSQL with row security; TypeScript; a lighter dev server if feasible (A4); a dedicated `Money` type; per-feature folders with automatic registration; per-agent databases cloned from a template; lint rules for D2 and for labelling every table branch-scoped or company-wide; migrations compatible across versions (for a future branch server); Chrome only for agents; tests heavy on money, permissions and payments, light on screens.
**Judging additions:** can agents check the front end in a text-based browser? (not with Lightpanda today); how many files change to add a feature; whether invalid states are impossible to write.

## I. Decisions made after the Session 2 review

These override earlier text where they differ.

| # | Decision | Notes |
|---|---|---|
| I1 | **Web-based, one web codebase.** | Branch servers are planned (I9). |
| I2 | **No Android wrapper, no Capacitor, no native app.** | Replaces an earlier decision that allowed a Capacitor wrapper. |
| I3 | **Printing runs through a print relay on the branch network.** The printers are already bought (Bluetooth thermal; models to be confirmed). Where the relay physically sits depends on the models: next to each printer, or by USB or LAN. Web Serial over USB is an optional extra for a fixed PC. | The earlier approach (an app polling the cloud database every 3 seconds) failed on unreliable internet, so **no printing path may depend on the internet**. Kitchen ticket printing is not needed while orders reach screens; guest receipts still are. Most Bluetooth printers accept one connection at a time, so each printer is owned by one relay or device. A print adapter holds interchangeable drivers. |
| I3a | **How the printer experiment will be judged:** 100 prints in a row with no failure; tap to paper under 2 s; recovery after the printer is switched off and on, or Bluetooth drops mid-print; what happens when a second device tries to print. | To be run when printer models are known and a printer is available. |
| I4 | **Fixed starting points: TypeScript, PostgreSQL (row security), React.** | The web framework around React (Next.js or lighter) is left to the Session 3 designers, who must justify it against the offline and branch-server need, the 610 versus 61 MB measurement, agent error-proneness, and phone load time. |
| I5 | **Hosting: DigitalOcean droplets for everything, a production droplet plus a staging droplet. Cloudflare's free plan in front** (CDN for static files, DNS, HTTPS certificates, basic protection). | No Kenyan DigitalOcean region as far as we know; Cloudflare lists Nairobi and Mombasa (2015 and 2021 sources; check the current list). Cloudflare speeds the app's static files only, not live traffic. **To verify:** the current free-plan terms, a timing test from a branch's real internet provider with and without it, and that live streams to screens work through the proxy. |
| I6 | **Session 3 format:** five designers (balanced, agent-friction, extensible, fast-light, correct-safe), one judge, an optional attack round, and a decision step. An **orchestrator session** reads the inputs, restates Session 3 for the owner's approval, writes the designer prompts, **acts as the judge** (the owner's decision), and helps the owner decide; it never designs. To limit bias, it lists its own interpretations before scoring, runs the placement test with blind helpers, and the owner makes the final choice. **Manual pasting**, 2 or 3 designers at a time. | The shared brief and orchestrator prompt were removed after Session 3 (the outcome is section J). |
| I7 | **A UI architecture and performance budget** are part of Session 3. A separate **design-system session** for the look and feel follows before the first feature. | |
| I8 | **Domain additions:** Printer, Print job, Cloud, Branch server, Served by, Floor operations, Company operations. | See the glossary in `domain-model.md`. |
| I9 | **A branch server in every branch, rolled out per branch after the pilot.** **Floor operations** (orders, kitchen and bar screens, waiter, chef and barista tasks, payment recording, printing, a local copy of menu, staff and notices) run on it. **Company operations** (HR, payroll, finance, inventory, reporting, AI, the Director's view, management) run in the cloud and are reachable from anywhere with the same login. They sync within seconds when the internet is up; when it is unreliable the branch keeps working and resyncs. | **Cloud first:** all branches start cloud-served; a branch is cut over to its own server and can be cut back; servers may arrive together or one by one. **One writer per branch at any moment.** In server mode floor work needs the branch network, so Wi-Fi coverage of the whole floor is a requirement. Payroll and other company features work from anywhere but not while the branch's own internet is down. Hard parts: update compatibility, staleness of permissions, cross-branch uniqueness (detect and flag), HTTPS on the branch network. |
| I10 | **The owner physically installs and looks after the branch machines. The remote tooling is built by the coding agents as part of the product** (a company feature in the cloud plus a small program on each branch machine): secure remote access, a health and last-sync dashboard, alerts, nightly backups with restore drills, staged updates with automatic rollback, a standard image with one-command install, a pre-imaged spare, remote power cycle. | A UPS is needed at each branch (power cuts). This tooling is not in the product brief's feature buckets yet; it should be added as a known feature. |
| I11 | **Settled in section J (was: open owner decisions):** the last-resort layer when the branch server or Wi-Fi is down; the failover policy between the branch server and the cloud (manual recommended); how fast a suspended or ended staff member loses access at an offline branch (proposal: the branch manager can suspend locally and immediately, with short sessions). | |
| I12 | **Still needed from the owner:** the printer models and where each printer sits. | Decides print boxes versus USB or LAN, and whether the hardware experiment can run. |

## J. Architecture decision (Session 3)

Decided 2026-11-04. Evidence: the judging report (14 gates, a blind placement test, scores), archived outside the repository on 2026-11-04; the results are summarised here. The chosen design is in `docs/architecture.md`. Five designs competed and all passed all 14 gates. Scores out of 65: fast-light 55, balanced 53, agent-friction 53, extensible 50, correct-safe 49. The top three were within 2 points, so the judge's recommendation was a lean, not a clear win. The owner chose **fast-light with the borrowed ideas and fixes below** and answered yes to the five questions put to them.

| # | Decision | Alternatives | Reason | Risk |
|---|---|---|---|---|
| J1 | **Base design: fast-light.** | balanced, agent-friction, extensible, correct-safe | Highest score. Lightest: 3 full-mode agents about 4.5 GB, branch server about 1.6 GB of 8 GB (estimates built on A and F). Simplest to operate. Fullest UI budget (13 CI-checked limits). | Client-only rendering. About 1,500 lines of our own sync, outbox and store code. |
| J2 | **Frontend:** Vite 8 single-page app, React 19, service worker (vite-plugin-pwa), TanStack Router, Tailwind 4 with tokens in one file, React Aria Components, TanStack Virtual. | Next.js 16, React Router framework mode, TanStack Start | Works offline. Static files can be served by the cloud and by the branch server. 61 MB dev server against 610 MB (A4). | First visit on a slow link about 1.5 to 2.5 s (estimate), once per phone. Real-phone speed unproven (G.1). |
| J3 | **Backend:** Node 26, Hono 4, typed commands and queries (`POST /rpc/<feature>.<name>`; each declares permission, side, input schema and retry key). Kysely on PostgreSQL 16 with row security. Plain SQL migrations per feature, add-only. | Express, Fastify, tRPC, REST, GraphQL; Prisma, Drizzle | Our rules need SQL control (row security, triggers, per-transaction identity). No central schema file for agents to collide on. | Our own small command layer. Agents know REST and Prisma better. |
| J4 | **Live data:** server-sent events fed by Postgres LISTEN/NOTIFY, resumable by cursor, 20 s heartbeat. Floor data: our own live store mirrored to IndexedDB, with an outbox for unsent commands. TanStack Query for company pages. Jobs: graphile-worker. No Redis. | Socket.io or WebSockets; Replicache, Zero, ElectricSQL; BullMQ with Redis; pg-boss | One-way push is enough and catch-up is built in (E3). No extra service on every branch server. | Browser storage can be cleared, which would lose unsent orders (see fixes). |
| J5 | **Packaging:** one signed release tarball, unpacked to a versioned folder, a symlink switch, systemd, a CI-built Debian image. No Docker on droplets or branch servers. | Docker images (four of five designs) | About 100 MB saved and one layer fewer. Rollback is a symlink flip. | Machine drift (covered by the one image). Local Postgres for development may still use Docker (A5). |
| J6 | **Folders and names (from agent-friction):** `features/floor/<name>` runs on the branch's writer; `features/company/<name>` runs in the cloud only. Fixed file endings. `pnpm where`. Error messages name the fix. A glossary lint. | A `side` field only | Best result in the blind placement test. | Hard to change once many features exist. |
| J7 | **One writer:** a per-branch epoch. The database refuses floor writes from a node without the current epoch, and the cloud refuses stale epochs. Every change is recorded in a change log by a trigger in the same transaction. Failover is manual. A returning old server's unsent rows are accepted as late arrivals and flagged. | Automatic failover | Two writers become impossible. A real sale is never rejected. | A cutover pause of 5 to 30 s (estimate). |
| J8 | **Cutover address:** the same hostname in both modes. Public DNS switches to the branch server's LAN IP, dnsmasq answers locally, and the cloud gets Let's Encrypt certificates by DNS-01. **Written fallback** if this fails on real phones: a separate LAN name per branch, with phones draining their queues first. | A different address per mode | The offline queue stays in place. | Untested on real Android phones and routers. It is the first hardware test. |
| J9 | **Who is asking (G6):** database-issued signed session tokens, verified in the database on every transaction. A narrow app role. A fresh PIN for approvals, refunds, verification and posting changes. | The app sets the user | A hijacked app server can act only as users logged in through it during the compromise. | Residual risk accepted. Audit entries carry the session, and alerts watch for misuse. |
| J10 | **Remote actions on a branch run by its own server** (a Director approving, a customer order): the cloud forwards the command to the branch writer, which runs it as the original actor. **Approval decisions are branch records.** The first decision applied wins; later ones are marked superseded. | Request rows converted by the branch (fast-light's own method) | Three of five designs. The branch writer stays the only writer. | Remote actions queue while the branch is offline. |
| J11 | **Remote access:** Tailscale first. | Headscale or a self-run WireGuard hub | The owner's choice. | Its free plan was reported non-commercial only (agent-friction, 2026-11-04). Confirm current terms and budget for a paid plan if needed. WireGuard stays the fallback. |

**Ideas borrowed from the other designs**
- agent-friction: the folder and file naming (J6).
- correct-safe: a CI suite that breaks each rule on purpose and expects the tests to fail; one trigger chain for writer check, audit and sync; write-once sales.
- extensible: floor code typed so it cannot see company tables; a commit fails if a floor row changed outside the command runner; test data always has two companies.
- balanced: the spare machine takes each update first (ring 0); a text outline of any screen (`pnpm see`).

**Owner decisions left open in the brief (section 4), now settled**
- **Last resort when the branch server or Wi-Fi is down:** the phone-side queue (already required in cloud mode) plus a numbered paper pad. A pre-imaged spare replaces a dead server.
- **Failover policy:** manual, one button on the ops dashboard, pressed by the owner or a Director. An alert fires after 2 minutes of silence.
- **Ended or suspended staff at an offline branch:** the Branch Manager suspends locally and immediately. Floor sessions are capped at 12 hours (a setting; owner decision 2026-11-04).

**Fixes required in the base design**
- Support branch versions N, N-1 and N-2 (fast-light had only N-1), with CI against both.
- The written DNS fallback (J8).
- The codebase map must show which features are copied to branches and which feature owns bills (the placement test tripped on both).
- Ask for persistent browser storage and show a visible count of unsent orders, so a cleared browser cannot silently lose the queue.
- Every floor action shows its state on screen: sending, saved, waiting or flagged.

**Open for Session 4**
- Reconcile file naming: fast-light's `api/contract.ts` and `handlers.ts` against agent-friction's one file per command.
- Zod or Valibot (leaning Zod unless the first build misses the 130 kB budget).
- How Postgres runs on the laptop (Docker or not), and how Node and Postgres are installed on the branch image.
- What each role's phone store holds, and what is cleared on logout or a lost phone.
- Printer models and positions (I12).
- The tripwire tests below.

**Stack tripwires (tests that would tell us to change something)**
- Real mid-range Android phone: first load about 2 s, tap at most 100 ms.
- Offline queue: refresh, killed browser, cleared storage, address change. Zero lost or duplicated orders.
- Floor store under dropped, duplicated and reordered events: every screen converges.
- Trigger cost per floor write (proposed bar: under 5 ms).
- Same-address DNS on a real phone and router.
- Live streams through Cloudflare with the 20 s heartbeat.
- Printing (I3a): 100 prints in a row, tap to paper under 2 s, recovery after drops.
- Fast loop at 50, 200 and 1,000 modules: about 10 s and 30 s.
- Real dev-server memory: 3 full-mode agents in about 7 GB.
- Blind agent trials and an attack round: mistakes caught by tools; no collisions between parallel agents.
- First-try correctness of agents with Kysely, Valibot and Hono.
- Update, rollback, restore and cutover rehearsals on staging.

**Conflicts with other documents (all applied or resolved on 2026-11-04 with the owner's approval)**
1. `domain-model.md` section 2 lists approval decisions as company-wide. Change to branch-scoped (J10).
2. `domain-model.md` rule 11 says "database trigger". Change to a two-step transfer: the branch writer clears the person, and a trigger on postings ends the floor posting only with that clearance.
3. `domain-model.md` glossary, Printer: "usually its station tablet". Change to "owned by one relay or device" (I3).
4. `decision-log.md` B1 says "option F, not adopted". It is adopted (I9).
5. `decision-log.md` G1 says "web, wrapped or native". It is web only (I1, I2).
6. `product-brief.md` section 5 lists open owner decisions that J now settles, and `decision-log.md` I11 is settled by J.
7. `decision-log.md` section A pins ESLint 10.14.0. The chosen design uses Biome plus an import-graph test and keeps ESLint out of the fast loop.
8. I11 said "short sessions". The owner settled on 12 hours for floor sessions (resolved).

## K. Agent experience (Session 4)

Measured 2026-11-04 on the owner's laptop (i7-8665U, 4 cores, 7.4 GB available, already busy). A throwaway slice (order, payments, receipts, 5 files per feature, copied to 50, 200 and 1,000 modules) with the pinned versions from A: TypeScript 7.0.2, Biome 2.6.4, Vite 8.3.5, Vitest 5.1.0, Hono 4.13.16, Kysely 0.29.8, Zod 4.6.7, Valibot 1.5.0, `pg` 8.23.1, graphile-worker 0.18.0, Playwright 1.64.1 (system Chromium), agent-browser 0.41.2, Postgres 16 in Docker. All throwaway files, the container and the scratch folder were deleted. The spec is `docs/agent-experience.md`. **Not measured:** real phones, real app size, React and Tailwind in the type check, portless (A6 figure reused), 5 to 6 light agents running together (parts measured).

### Measurements

| # | Question | Result |
|---|---|---|
| K1 | Fast loop at 51, 201, 1,001 modules (median) | Type check warm 0.35, 0.44, 0.97 s (cold 0.42, 0.64, 2.78 s). Biome 0.15, 0.18, 0.77 s. One feature's tests 0.84 to 1.05 s. Edit then type check, lint and feature tests at 1,001 modules: 2.5 s. Three at once: 4.0 s median, 4.7 s worst. Targets (10 s, 30 s) hold with a wide margin. Valibot and Zod type-check alike (0.74 s against 0.97 s warm) |
| K2 | Memory of real dev stacks | Vite 306 to 404 MB (not the 150 MB estimated), API with worker 46 MB, one Chromium with 3 contexts 939 MB, Postgres with 3 databases 109 MB, a type-check burst 339 MB. 3 full agents about 4.4 GB, 6 light agents 3.6 to 4.6 GB, of 7.4 GB. Database clone 46 to 78 ms |
| K3 | Trigger chain per floor write | Row security alone 0.67 ms (unindexed postings table with 200 rows), chain (epoch check, actor stamp, audit, change log, notify) adds 0.15 ms. Token check 0.085 ms per transaction. A full command of 4 writes over local TCP: 7.6 ms median, 13.2 p95. Bar was 5 ms for the chain: met 33 times over. The 7.6 ms is inside the 30 ms handler budget P12 |
| K4 | Offline queue and floor store | Queue prototype, 10 of 10 checks: refresh, killed browser, network down, lost replies (13 attempts for 4 orders, 4 stored), one invalid order not blocking the rest. Break test: removing the unique key gave 8 rows for 4 orders and failed the check. Floor store: 5,000 of 5,000 random drop, duplicate and reorder schedules converged; without the revision check 431 converged; with a "highest seen" cursor 1,077 converged. Cleared storage loses unsent orders (cannot be recovered by code). A different address has a separate store. `persist()` returned false in desktop Chromium |
| K5 | Zod or Valibot | First-try, 6 tasks each, no iteration, haiku at low effort: type check clean 12 of 12. Tests: Zod 6 of 6, Valibot 5 of 6 after fixing my own test fault; the Valibot miss was my spec not saying to reject extra keys (the Zod agent happened to use a strict object). A tie. Bundle: Zod about 26 kB gzip for 1 or 40 schemas, Valibot 2.1 and 2.7 kB. Small sample, one weak model |
| K6 | agent-browser isolation (risk 12) | **Not isolated.** Two sessions on one Chromium shared cookies and localStorage. Playwright contexts on the same Chromium were isolated. An agent-browser session with its own Chromium is about 765 MB |
| K7 | Biome rules | GritQL plugin rules work, with a custom message naming the fix and per-folder scope: "no `fetch` in floor UI", "no `*` or `/` in domain code". They match syntax only, not types |
| K8 | Node type stripping | Runs `.ts` with `#core/*` imports from `package.json` `imports`; `tsc` with `nodenext` accepts them. `tsconfig` `paths` do not work |

### Decisions

| # | Decision | Alternatives | Reason | Risk |
|---|---|---|---|---|
| K-1 | **Valibot.** Resolves the Zod lean in J | Zod 4 | 2.7 kB against 26.5 kB gzip; correctness tied. The 130 kB first-load budget gets 24 kB back | Weaker model familiarity; the agent first-try rate was equal in this small test |
| K-2 | **One file per command and query.** `x.command.ts` and `x.query.ts` (shared contract), `x.command.server.ts` (server only), `x.screen.tsx`, `x.event.ts`, `x.handler.ts`, `x.print.ts`, `x.job.ts`. Replaces `api/contract.ts` plus `handlers.ts` | fast-light's two shared files; one file per command holding schema and handler | Two agents adding commands to one feature do not touch the same file; the client never imports server code (arch check) | Hard to change once many features exist |
| K-3 | **Command names:** `pnpm new feature\|command\|query\|screen\|migration`, `pnpm where [word]` (no word lists the registry), `pnpm check`, `pnpm see`, `pnpm agent:init`. Replaces `new-feature`, `features` | architecture §4 names | One verb with kinds, fewer commands | |
| K-4 | **Postgres on the laptop: Docker, image `postgres:16-alpine`, one container, one database per agent.** Branch image: Postgres 16 from the PGDG apt repository, because Debian 13's own package is probably 17 (from my knowledge, not checked); Node bundled in the release, not from apt | Native Postgres (needs sudo; Arch's version is newer than 16); move everything to 17 | No sudo, already pulled, pinned version | Docker's first start was 28 s on a busy laptop (A5) |
| K-5 | **Browser isolation:** `pnpm see` uses a fresh Playwright context per call on the shared Chromium; `agent-browser` only for exploration, with a per-agent hostname | A Chromium per agent (765 MB each); trust agent-browser sessions | Isolation proven for contexts, cost about 140 MB | Hostname isolation is untested |
| K-6 | **Phone store by role, logout, lost phone, persistent storage, unsent count, lost-queue detection by heartbeat.** See spec section 7 | | Contents hold no secrets, so a lost phone exposes only branch floor data | Heartbeat detection is design only; Android persistence is unproven |
| K-7 | **Memory rule:** at most 3 full or 6 light agents, enforced by `agent:init`. Vite figure in architecture §10 raised from 150 to 400 MB | | K2 | Real app is larger |
| K-8 | **Rule 1 enforcement changed:** branded types plus a path-scoped Biome rule, not a type-aware rule | A type-aware rule | K7 | |
| K-9 | **CI plan:** Team plan recommended; pipeline as in spec section 9 | Free plan | About 2,200 minutes a month at 3 agents (estimate) against 2,000 Free | The estimate has no real job times yet |
| K-10 | **Keep Biome** (owner, 2026-11-04). **Fallback:** if a speed target is missed and Biome is the cause, or a needed rule cannot be written, time ESLint and oxlint on the same slice and switch. ESLint and oxlint were never timed by us | ESLint plus Prettier; oxlint | Biome: fastest measured, one tool, plugin rules worked (K7) | No type-aware rules |
| K-11 | **"Hat" is renamed "Posting"** (owner, 2026-11-04): a role, a scope and two dates. `hats` becomes `postings`. A rename pass over `domain-model.md`, `architecture.md`, `product-brief.md` and this log is still to be done, before any code | Role assignment; Appointment | "Assignment" sounds like "reassign" | Missed occurrences; do a search before the first migration |
| K-12 | **Two extra commands:** `pnpm check` with no name (checks what changed) and `pnpm evidence` (prints the PR evidence block) | | Fewer steps for agents | Built with the other tools in the foundation |
| K-13 | **No hooks for now** (owner, 2026-11-05); add one only when a real problem needs it. A first set was built and tested (43 sample events) and removed. **Lessons for any future hook:** (1) match commands, never quoted text or file-writing heredocs, or commit messages and PR bodies trigger false alarms; (2) a rewrite of the agent's file by a hook makes its next edit fail; (3) run the full check once when the agent finishes, only after code edits, never after every edit; (4) hooks are guard rails, not a security boundary; (5) give an off-switch; (6) a hook is live the moment its settings are saved | Full check after every edit; hooks from day one | Hooks cannot be proven without real code, and each one is a place for friction | Without hooks, an agent can forget `pnpm check`; CI and the PR template still catch it |
| K-14 | **Repository** `Fred-Edwin/wendo-management-platform`, **public from 2026-11-05** (owner decision, to get free CI minutes and free required checks) and to be made private after the Team plan is bought. Contents: `CLAUDE.md`, `.claude/settings.json` (allow rules only: deny and ask-first rules removed on the owner's decision), issue and pull request templates, CI workflow (jobs skip until `package.json` exists; official actions pinned to commit hashes at least 2 weeks old), Dependabot for actions. **Settings applied and read back:** `main` needs a pull request and the checks gate, perf, e2e, compat, check, db and build (skipped ones count as passing), no force push or deletion, and admins are bound too; squash merge only with branch deletion and auto-merge allowed; secret scanning with push protection; Dependabot alerts and security updates; workflow token read-only; first-time outside contributors need approval before workflows run. **Tested:** a direct push to `main` is refused; a pull request showed `gate` passing and six jobs skipped, state CLEAN, and was squash-merged (#1) | Private on Free (no required checks) | | **Everything pushed while public (architecture, security design, hostnames) can be cloned and forked permanently.** **Order matters when going private: buy Team first.** On a private repository without a paid plan, branch protection is not enforced. The CI jobs after `gate` are untested until code exists |

### Open risks after Session 4

1. All numbers are from a synthetic slice. Re-measure at about 30 real features (spec section 1, 10).
2. Cleared browser storage loses unsent orders. Only persistence, the visible count and heartbeat detection reduce this. Android behaviour is unproven.
3. The Valibot versus Zod agent test was small and used a weak model.
4. Hostname-based browser isolation, portless, and 5 to 6 light agents together are not measured.
5. Section G risks 1 to 11 are unchanged. Risk 12 is now answered: agent-browser sessions are not isolated (K6).
6. Tailscale commercial terms (J11) were not checked.

## Appendix: feature shape

See `docs/feature-shape.md`.
