# Product brief: Wendo Business Operating Platform

Output of Session 1 (Discovery). Sources: `docs/inputs/product-vision.md` and the interview. Updated after Session 2 to match `docs/decision-log.md` section I, which overrides anything older. Edit freely.

## 1. The product

An internal, AI-native operating platform for Wendo Coffee and Bistro, a Kenyan restaurant company with 3 branches today and up to 10 planned. It covers floor service, kitchen and bar, inventory, staff, payments, finance and reporting. Directors see every branch live. Routine work (reconciliation, reminders, reports, ordering suggestions, rosters) is automated by AI, and humans approve anything involving money, people or compliance.

**Floor operations come first and must never stop.** Taking orders, getting them to the kitchen, printing receipts, and recording payment keep working through internet outages of minutes or hours. UI quality and speed are first-class requirements on every feature, not finishing touches.

This is a rebuild. The first attempt failed on code quality, unmeasured performance, amateur UX, bloated spec documents, and the cost in time and tokens of every change. It was also built without knowing how the scope would grow. The new build is engineered from the foundations up, with the menu and any other needed data imported from the old system. Nothing else carries over.

The builder is one person (the product owner) working with coding agents. One branch pilots first, then the rest follow.

## 2. Users, roles and scopes

Roles are assigned per person **per branch**. One person can hold several roles and work at several branches (for example a manager of two branches). HR transfers floor staff between branches easily, and history stays attached to the branch where it happened.

| Role | Scope |
|---|---|
| Director | Everything, all branches. Approves anything, needs no second approver, is still audited |
| Branch Manager | Their branch(es). The only branch approver. Creates and edits users in their branch within allowed roles (not Directors or Accountants). Receives waiters' cash and sees expected amounts |
| HR Manager | Company-wide. Hiring, contracts, transfers between branches |
| Accountant | Company-wide. Reconciliation, expenses, payroll processing |
| Store Manager, Store Attendant | Central store |
| Department Heads | Their department in their branch. **Never approve.** Their duties arrive with the features that need them (such as shift schedules) |
| Waiter | Their branch. Takes orders, bills, takes payment |
| Chef, Barista, Housekeeper | Their branch |
| System Admin | Technical only: integrations, M-Pesa settings, device and printer registration, backups, health, read-only audit logs, emergency access recovery. Cannot see payroll or finance, cannot approve, cannot edit audit logs. The product owner holds it at first |

## 3. Foundations versus features

**Foundations (built first, shared by every feature):**
- Company, branch and department structure, with a company identifier on every record
- Users, per-branch roles and permissions, enforced on the server
- Approvals (Branch Manager or Director, with queueing when offline)
- Audit log
- Notifications
- Money model: whole cents in storage, everything a guest pays in whole shillings, menu prices including VAT, and each sale holding the fields a tax receipt would need (tax category, fixed receipt number, final amounts)
- The floor and company split: floor operations can run in the cloud or on a branch server, with a sync-ready write path, device-made IDs and a per-branch "served by" setting
- Offline-capable device layer and a print adapter (receipts through a print relay on the branch network)
- Data import (menu first)

**Features:** everything else in the vision catalogue (A to J), built one at a time, each end to end.

## 4. Feature buckets

- **Known:** foundations; orders and service; kitchen and bar displays; payments (cash, M-Pesa, card); receipt printing (kitchen ticket printing is optional while orders reach screens); waiter cash tracking for the manager; director oversight; **branch operations tooling** (remote access, health and last-sync dashboard, alerts, backups with restore drills, staged updates with rollback, machine imaging), needed once branches run their own servers. See `docs/decision-log.md` section I (I9, I10).
- **Likely:** the rest of the vision catalogue (menu management, inventory and procurement, workforce, tasks, finance, reporting) and the AI layer (ask-anything, daily briefings, reconciliation, anomaly detection first). Also the **customer ordering app** with customer accounts and delivery, loyalty points, reservations, delivery apps and QR self-ordering.
- **Unknown:** not predictable. Covered by the stress-test in Session 2.

## 5. Constraints and quality bar

- **Speed (mid-range Android phones):** tap response under 100 ms; order saved and visible under 1 s; kitchen ticket on screen within 2 s of sending; pages under 2 s.
- **Devices:** Android only for waiters (company-owned assumed), used through the web app in the browser: no Android wrapper. Android tablets for kitchen and bar. Bluetooth Classic printers, already owned. A small branch server in each branch, installed and looked after by the owner, with a UPS and Wi-Fi covering the whole floor.
- **The internet is allowed to fail.** Floor operations (orders, kitchen and bar screens, the tasks of waiters, chefs and baristas, receipts, payment recording) run on the branch server and never depend on the internet. Company operations (HR, payroll, finance, inventory, reporting, AI, management) run in the cloud and are reachable from anywhere with the same login. Online speed and simplicity still come first.
- **Branch servers (decision I9):** every branch gets its own server, rolled out per branch after the pilot. Branches start served by the cloud, are cut over to their server, and can be cut back. The branch syncs with the cloud within seconds whenever the internet is up and resyncs when it returns, so Directors see every branch. One writer per branch at any moment. Floor work needs the branch network. Until a branch has its server, the waiter's phone queues orders and syncs later, with duplicate protection and offline-safe numbers (proven in Session 2). **Settled in decision log J:** the phone-side queue and a numbered paper pad stay as the last resort when the branch server or Wi-Fi is down; failover between the branch server and the cloud is manual, by the owner or a Director; a Branch Manager can suspend staff locally and immediately at an offline branch, and floor sessions are capped at 12 hours.
- **Payments:** cash, M-Pesa and card, each settling to a different account. Each branch has its own pay-bill number.
- **Hosting:** DigitalOcean droplets for everything: a production droplet and a staging droplet, with Cloudflare's free plan in front for static-file delivery, DNS, HTTPS and basic protection. One deployable bundle runs on a droplet and on a branch machine (decision I5).
- **Quality:** UX and performance apply to every feature. Performance is measured during development.
- **Agent-first codebase (top priority, equal to UX and speed).** The codebase exists to be built, maintained and extended by coding agents. Each feature is self-contained, has one repeatable shape created by a generator command, and can be checked in isolation. Agent feedback speed is a hard requirement. The detail is in `docs/workflow/phase-1.md` (sections 3 to 5):
  - Fast loop on the laptop: about 10 s for type check and lint, about 30 s for related tests.
  - Full tests and production builds never run on the laptop, only in GitHub Actions.
  - Laptop limits (4 cores, about 15 GB RAM) mean a memory budget per agent, one shared database, one shared browser and a worktree per agent.
  - Heavy tests on money, permissions and payment callbacks; light on simple screens.
  - Rules are enforced by tools (types, lint, tests), not documents.
- **Process:** few, short documents. Rules enforced by tools.
- **Pilot:** one branch first, after the core floor operations are built. It starts served by the cloud and is cut over to its own branch server.

## 6. Decisions made

- Rebuild from scratch, importing only needed data such as the menu. *The old code and documents were unmanageable.*
- Core floor operations keep working through internet outages of minutes or hours, by running on a branch server. *Kenyan internet is unreliable and the old system failed here.*
- **A branch server in every branch, rolled out per branch after the pilot** (decision log I9). Floor operations run on it; company operations stay in the cloud; they sync whenever the internet is up. *The owner's experience of the old system failing on unreliable internet. This replaces the earlier leaning to no on-site hub.*
- **Printing:** the printers are Bluetooth Classic and already owned, so a browser alone can't reach them (Web Bluetooth is BLE-only). Printing runs through a **print relay on the branch network**, behind one print adapter, with ESC/POS bytes built in TypeScript and unit-tested on the laptop. **No Android wrapper (no Capacitor), no companion app, and no fully native Android app** (decision log I2, I3). Web Serial over USB is an optional extra for a fixed PC. **No print path may depend on the internet:** the old approach, an app polling the cloud database every 3 seconds, failed on unreliable internet. Kitchen ticket printing is optional while orders reach screens; guest receipts are needed. Where the relay sits depends on the printer models and where each printer sits.
- **Hosting:** DigitalOcean droplets for everything (production plus staging), with Cloudflare's free plan in front (decision log I5). *The owner's preference. The same bundle runs on a branch machine.*
- **Money is stored as whole cents; everything a guest pays is whole shillings. No service charge. Menu prices include VAT** (decision log B5, B6, B12).
- Multi-company door left open cheaply with a company ID on every record. *No second company is planned.*
- Waiters keep their own cash tally. The system shows expected cash per waiter only to the manager, director and accountant. Waiters see order totals but no running total. *An independent check against theft.*
- Cards use a separate bank terminal and the waiter records the approval code. *No card integration to build.*
- M-Pesa payments are **recorded by hand at first**: the guest pays to the pay-bill and the waiter keys in the code. The entry is "pending verification", counts as paid, and the accountant verifies it against the M-Pesa statement; a Branch Manager, the accountant or a Director may reject or flag it. Automatic M-Pesa integration comes later. *Prevents fake or reused codes.*
- Approvals: Branch Manager or Director only, never Department Heads. No acting manager: a Director covers an absence. *Owner's decision.*
- Offline cancels, discounts and comps are queued and applied after approval. *Avoids abuse.* ("Cancel" replaces "void".)
- No KRA eTIMS tax receipts for now. *The client hasn't asked for them.*
- Customer ordering app is "likely, later". Staff and customer identities stay separate. *The vision said staff-only.*
- System Admin is a narrow technical role. *Keeps separation of duties.*

## 7. Assumptions still open

- Waiter phones are company-owned and Android; the phone model is not chosen. The printers are already owned; their models and where each one sits are still to be recorded.
- The print relay is untested until a printer is available. The printer models and positions decide whether it sits next to each printer or uses USB or LAN.
- Money is Kenyan shillings only.
- The bill and the staff's cash handling match the owner's description (confirm at the pilot branch).
- Whether Wendo is VAT-registered or must use eTIMS: **confirm with the accountant.**
- eTIMS is deliberately out of scope for now. If added later, KRA signing may need internet, which could conflict with offline receipt printing. Revisit then.
- Payroll deductions (PAYE, NSSF, SHIF, Affordable Housing Levy) apply: **confirm with the accountant** before payroll is built.
- What an AI provider may see: to be decided in the AI research session.
- How much outage pain the branch servers remove: measure the pilot branch's real outage minutes before and after.

## 8. Hard or risky parts

- **Branch servers and sync:** syncing, duplicates, ticket numbering, one writer per branch, update compatibility between the cloud and older branch machines, permissions going stale at an offline branch, HTTPS on the branch network, and a last resort when the server or Wi-Fi is down.
- **Bluetooth Classic printing:** browsers can't reach Classic printers (Web Bluetooth is BLE-only). It needs a print relay within Bluetooth range of each printer, or a USB or LAN connection, and has real-hardware quirks (permissions, character sets, paper width).
- **Power and the branch network:** power cuts need a UPS for the server, router and printers; Wi-Fi must cover the whole floor; a failed branch machine needs a pre-imaged spare. Someone must look after up to ten branch machines (the owner, with remote tooling built by the coding agents).
- **M-Pesa:** callbacks arriving late, twice or never, and manual pay-bill entries.
- **Cash:** waiter-held cash is the main theft and variance risk.
- **Approvals stall when the only approver is away** and offline requests wait. Service can suffer.
- **Per-branch roles, transfers and the System Admin boundary.**
- **Speed targets** on mid-range phones in a busy service.
- **AI reconciliation** needs a deterministic matching layer, with AI helping only on the leftovers.

## 9. Session 2 focus

*Historical: Session 2 is complete. Its results are in `docs/decision-log.md`, and later decisions override this section.*

**Domain model** (company, branch, person, per-branch role, order, payment, approval, audit entry), then the stress-test against: customer app, delivery, loyalty, reservations, QR ordering, a second company, eTIMS.

**Recommended experiments (throwaway code):**
Experiments use only what is available now: the desktop, the laptop and the internet. **Bluetooth printing and Android-phone tests are deferred** until the hardware is available, and stay an open risk. When hardware arrives, the check is the print relay printing to the real printers (decision log I3, I3a). Session 3 designs one print adapter with the relay as its main driver, with the ESC/POS builder testable on the laptop. Record the printer model(s) before buying more.
1. Offline order taking and sync, simulated in a desktop browser (network switched off), with duplicates and ticket numbers.
2. Order creation speed against the 100 ms and 1 s targets, in a desktop browser.
3. Duplicate, late and out-of-order M-Pesa callbacks against a fake, including manual pay-bill entries.
4. Billing math in integer shillings with service charge, discount, comp and tip.
5. A Branch A user can never read Branch B data, with per-branch roles and transfers.
6. Order-to-kitchen-screen time against the 2 s target.
7. Tool speed and memory on the real laptop. Run this early and in full: it sets the memory budget per agent.

**Also in Session 2:** a rough "feature shape" sketch showing what one self-contained feature looks like (folders, files, wiring, starter test). It gives the Session 3 architecture competition a clear target.

A separate session should cover AI research (data rules, approvals, audit).
