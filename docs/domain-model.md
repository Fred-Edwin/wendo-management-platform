# Domain model: Wendo Business Operating Platform

Output of Session 2. The product brief wins wherever it differs from the vision. Edit freely.
"Tested" means a throwaway experiment proved it (see `decision-log.md`). "Design only" means reasoned on paper and not yet tested.

## 1. Glossary (use these words in code and docs; do not invent synonyms)

| Term | Meaning | Not the same as |
|---|---|---|
| **Company** | The tenant (Wendo). Its ID is on every record. | Branch |
| **Branch** | One restaurant location. Owns its tables, stations and devices. | Department |
| **Department** | A team inside a branch (Service, Kitchen, Barista, Housekeeping). | Station |
| **Station** | A kitchen, bar or coffee screen that receives tickets. | A person |
| **Printer** | A thermal printer owned by one print relay or device. Most Bluetooth printers accept only one connection at a time. | Station |
| **Print job** | A ticket or receipt sent to a printer, routed to the relay or device that owns it. No print path may depend on the internet. | Order |
| **Cloud** | The central servers on DigitalOcean droplets: company operations, plus a copy of every branch's records. | Branch server |
| **Branch server** | A small machine in a branch, installed and looked after by the owner, that runs floor operations and syncs with the cloud. | Phone |
| **Served by** | A per-branch setting: whether that branch's floor operations run in the cloud or on its own branch server. One writer at a time. | Role |
| **Floor operations** | Orders, tickets, kitchen and bar screens, waiter, chef and barista tasks, payment recording, printing. Run on the branch server when a branch has one. | Company operations |
| **Company operations** | HR, payroll, finance, inventory, reporting, the AI layer, management. Always in the cloud, reachable from anywhere with the same login. | Floor operations |
| **Table** | A place where guests sit, belonging to one branch. | Order |
| **Person** | A staff member's identity. | Customer |
| **Customer** | Someone using a future ordering app. Separate identity and tables. Never a Person. | Guest |
| **Guest** | Someone dining. Has no account. | Customer |
| **Role** | A job (waiter, chef, branch manager...). | Permission |
| **Permission** | One thing a role may do (for example `payments.verify`). | Role |
| **Scope** | Where a role applies: one branch, or the whole company. | Role |
| **Posting** | A role given to a person, for a scope, between two dates. The end date is the first day it no longer applies. | A user account |
| **Transfer** | Ending one posting and starting another. Blocked while the person has open orders or unsettled cash. | Reassign |
| **Order** | What guests asked for. **One waiter is in charge.** (Later: may start with nobody in charge, for customer, QR or delivery-app orders.) | Bill |
| **Order number** | The number on tickets. Restarts at 1 every day, per branch. | Receipt number |
| **Bill** | The amount to pay for part or all of an order. An order has one or more bills (split bills). | Order |
| **Payment** | Money received against a bill, by cash, M-Pesa or card. A bill can have several payments, each with its own method. | Receipt |
| **Pending verification** | A manually typed M-Pesa payment not yet checked against the M-Pesa statement. It counts as paid, and the guest can leave. | Rejected |
| **Verified** | The accountant (or a Director) confirmed the payment against the statement. | Approved |
| **Receipt** | Proof of sale, one per bill. Its number never repeats and never changes once printed. | Order number |
| **Sale** | A closed, paid bill with frozen final amounts, tax category and receipt number. | Order |
| **Cancel** | Remove an item or a whole order **before payment**. Needs approval and a reason. | Refund |
| **Comp** | Give an item free. Needs approval and a reason. | Discount |
| **Discount** | Reduce a price. Needs approval and a reason. | Comp |
| **Refund** | Give money back **after payment**. Needs approval and a reason. | Cancel |
| **Approval** | A recorded permission to cancel, comp, discount or refund. Only a Branch Manager (own branch) or a Director can approve. | Verification |
| **Reassign** | A department head hands an open order to another waiter in the same branch. Recorded. It is not an approval. | Transfer |
| **Audit entry** | Append-only record of who did what, to what, and when. Saved in the same step as the action. | Log file |
| **Actor** | Who did something: staff, customer, guest, system, or AI acting for someone. | Person |
| **Company-wide** | Applies to the whole company and is not tied to one branch. Directors, HR, Accountants and System Admins hold company-wide postings. | Branch |
| **Platform** | The web system itself: one system, used from phones, tablets and laptops. | A branch |

## 2. The things and how they connect

```
Company
 ├─ Branch ── Department
 │     ├─ Table, Station, Device
 │     └─ Order (one waiter in charge, order number, branch, table)
 │           ├─ Order lines
 │           └─ Bill (one or more) ── Payment (one or more) ── Payment history (append-only)
 │                                          └─ Receipt number (one per bill)
 ├─ Person ── Posting (role + scope + start date + end date)
 ├─ Approval (cancel / comp / discount / refund: requested by, approved by, reason; branch records)
 └─ Audit entry
Customer: separate, added later
```

**Where each record belongs:**
- **Branch-scoped:** orders, bills, payments and their history, cash entries, tickets, receipts, approval requests and approval decisions (decision log J10: the branch writer decides which decision came first).
- **Company-wide:** menu and prices, people, postings, permissions, settings.

When a branch is served by its own server, the branch server is the authority for that branch's records, and the cloud is the authority for company-wide records. Each side keeps a copy of what it needs from the other.

## 3. Money

- **Stored as whole numbers of cents.** 450 shillings is `45000`. Never decimals or floats.
- **Everything a guest pays is whole shillings:** prices, discounts, tips, payments, change. Cents are refused with an error. Only the VAT portion keeps its cents.
- **Menu prices include VAT.** The tax portion is stored per item for reference and never added on top.
- **There is no service charge** (the client did not ask for one). Adding one later is a small change.
- **Order of calculation:** comp first, then discount on what remains, then the total. Tip is separate: no discount and no tax.
- **Rounding:** half a shilling rounds up. Leftover shillings are shared by largest remainder, ties settled by item ID so the result never depends on list order.
- **Splitting:** the waiter chooses **by food share** (the tip follows each guest's food) or **equally**. A shared item splits equally. Shares always add up exactly.
- **Cash** may be overpaid (the rest is change). **M-Pesa and card** may not exceed what is due.

## 4. Numbers

| Number | Purpose | Rule |
|---|---|---|
| Internal ID | Hidden identity | Made on the device, so it is unique forever and works offline |
| Order number | Called out in the kitchen | Online: the server hands out 1, 2, 3 per branch per day, no gaps. Offline: the device adds its own letter (`B-12`), which cannot clash. |
| Receipt number | Proof of sale | Same rule. One per bill. Never repeats or changes once printed. |

If two devices ever print the same number, both orders are kept, told apart, and flagged. A real sale is never rejected.

## 5. Who can do what

| Role | Scope | Can | Cannot |
|---|---|---|---|
| Waiter, Chef, Barista, Housekeeper | One branch at a time | Work in their branch. Waiters take orders and record payments. | See other branches. See payroll. Approve. |
| Department Head | Own department, own branch | Reassign an open order to another waiter | **Approve** |
| Branch Manager | Their branch(es) | Approve. Reject or flag payments. Add floor staff to their branch. | Add accountants, Directors or other managers. See other branches. |
| Accountant | Company-wide | See all branches' orders and payments. **Verify** payments. Reject or flag. | See payroll. |
| HR Manager | Company-wide | Payroll. Create postings (not Director or System Admin). End postings (transfers). | See orders or payments |
| Director | Company-wide | Everything, including approve and verify. Still audited. | Nothing, but every action is recorded |
| Store Manager, Store Attendant | Central store (company-wide) | **Not modelled in Session 2.** Their duties arrive with the inventory feature. They hold company-wide postings and get no access to orders, payments or payroll until then. | |
| System Admin | Company-wide, technical only | Read the audit log. Technical settings. | See orders, payments or payroll. **Approve.** |
| AI | Acts as the person who asked | Draft, suggest, remind | **Approve.** Act beyond the asker's postings. (Design only) |

Approvals: only a Branch Manager (own branch) or a Director. There is no acting manager. If the manager is away, a Director approves and requests wait in a queue.

## 6. Non-negotiable rules and the tool that enforces each

| # | Rule | Enforced by | Status |
|---|---|---|---|
| 1 | Money is whole cents. Guest-paid amounts are whole shillings. | A `Money` type, plus database checks on amount columns | Tested |
| 2 | Every sale stores tax category, receipt number and final amounts | Not-null database columns | Design only |
| 3 | Payment entries are idempotent. An M-Pesa code can be used once. | Retry key plus a unique index. Later callbacks use the same path. | Tested |
| 4 | Payment records are append-only. Status is worked out from history. | Database triggers and a restricted app role | Tested |
| 5 | A bill can never be overpaid, even when phones pay at the same moment | Database trigger with a row lock | Tested |
| 6 | Every cancel, comp, discount and refund records who approved it and why | Not-null columns and a rule that the approver is the acting person | Partly tested (approver rule tested; reason is design only) |
| 7 | Only a Branch Manager (own branch) or a Director approves | Permission table plus database policy | Tested |
| 8 | A user can never read another branch or company | Database row security, checked once per query. A test fails the build if any table with a company ID lacks it, or any view lacks `security_invoker`. | Tested |
| 9 | Company ID agrees with its branch on every record | Composite foreign keys | Tested |
| 10 | Roles belong to a person per branch with dates. Floor staff hold one branch at a time. | The postings table and an overlap constraint | Tested |
| 11 | No transfer while a person has open orders or unsettled cash | Two steps: the branch writer checks open orders and cash in its own database and records a clearance; a trigger on postings ends a floor posting only with that clearance (decision log J) | Tested on one database; the two-machine path is design only |
| 12 | Only the accountant or a Director verifies. A manager, accountant or Director may reject or flag. Every review records who. | Permission table plus database policy | Tested |
| 13 | An order saved on a phone is never lost or doubled, even offline | Device-made IDs, a phone-side queue, a unique order ID | Tested |
| 14 | Offline cancel, comp and discount are queued for approval, never applied directly | Server-side queue | Tested |
| 15 | Staff and customer identities stay separate | Separate tables | Design only |
| 16 | An AI never approves and acts only within the asker's postings | Actor type check in the approval rule | Design only |
| 17 | Every table is labelled branch-scoped or company-wide | Label per table, checked by lint | Design only |
| 18 | A floor write records a sync event in the same step as the change; an event is marked sent only after the cloud confirms; repeats are harmless | One generated write path plus a test | Tested in miniature (experiment f) |
| 19 | A branch has one writer at a time: the cloud or its branch server | A per-branch served-by setting; manual failover (recommended) | Design only |
| 20 | Floor code reads company-wide data only through a local read-only copy | Lint boundary between floor and company features | Design only |

## 7. Stress-test results (paper)

| Imagined feature | Needs | Foundation change | Verdict |
|---|---|---|---|
| Customer app, accounts, delivery | Customer identity, delivery details, rider role | An order's person-in-charge can be empty. Customers see only their own orders. | Add a module |
| Loyalty | Points ledger, points redeemed as a discount | None | Add a module |
| Reservations | Bookings tied to tables | Tables belong to a branch | Add a module |
| Delivery apps | External orders, bulk payouts | External order reference, unique | Add a module |
| QR self-ordering | Guest orders with no account | Same as customer app | Add a module |
| Second company | Separate data, settings, currency | Company boundary tested in the database. Currency stored per company. | Add a module |
| AI: ask-anything | Sees only what the asker may see | AI runs with the asker's postings | Rule |
| AI: acts for people | Reminders, chasing, drafts | Actor type with "acting for". AI cannot approve. | Small addition |
| AI: reconciliation | Imported statements matched to payments | Payments carry method and settlement account | Add a module |
| AI: anomaly detection | History of cancels, comps, discounts, cash | **A reason is required** on each | Small addition |
| AI: drafts (rosters, reorders, invoices) | AI proposes, human confirms | AI-created records start as drafts | Pattern |
| AI: deeper automation | Whole workflows | Same rules hold because the AI acts as a posting | Works |
| **Branch server in each branch** (planned after the pilot) | Syncing, remote approvals, staleness policy | Scope labels. A single write path that records a sync event. Cross-branch changes are events. Cross-branch uniqueness becomes "detect and flag". | **Add a module** (tested in miniature; designed in from day one) |

**Foundation changes the stress-test produced:** (1) order's person-in-charge may be empty and the order records its channel; (2) actors can be staff, customer, guest, system or AI, with "acting for"; (3) tables belong to branches; (4) cancel, comp, discount and refund require a reason; (5) company isolation tested alongside branch isolation; (6) currency stored per company; (7) AI-created records start as drafts.

**Designed in from day one for branch servers (decision I9):** label every table branch-scoped or company-wide; one write path for floor records that records a sync event in the same step; no database-wide counters for business identity; cross-branch changes as events; cross-branch uniqueness as detect-and-flag; a per-branch served-by setting; floor code reads company-wide data only through a local read-only copy.
