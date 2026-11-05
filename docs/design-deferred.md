# Design backlog: what Session 5 deferred

Recorded 2026-11-05 at the end of Session 5. These are the parts of the design that were not done, with what is already decided about each, what is needed to do it, and a suggested order. The owner will design the remaining components when there is time. Until a part here is designed, agents must stop and open a component request (rule 15 in `CLAUDE.md`) if a feature needs it. Decisions made so far are in `docs/design-system.md` and decision-log section L.

## A. Components and screens not designed (suggested order)

The order is by how many features each part blocks.

| # | Part | Why it matters | Already decided | Needed to design it |
|---|---|---|---|---|
| 1 | **Floor shell** (the frame for waiter and kitchen devices: navigation, always-visible unsent count, device status) | Blocks every floor feature. AppShell covers the office and the manager phone only. | System font, 56 px primary actions, no skeletons, the "N unsent" count is always visible above 0 (agent-experience section 7), statuses never fade | Owner's choices on what a waiter sees first; whether stations and waiters share one frame |
| 2 | **Tile** (menu item, table) | The architecture names it; the waiter ordering screen needs it. | No cards, whole tile is the target, 56 px minimum, quantity stepper exists, money text exists | Owner's taste on a tile with no card; menu photos or not (brand assets) |
| 3 | **Print layouts** (guest receipt, kitchen ticket, payslip) | Receipts block payment features; payslips block payroll. Different medium (thermal paper, A4). | Printing never depends on the internet (I3); ticket states New, Preparing, Ready, Served; money as `KSh 1,250` | Printer models and paper width (open item I12), the brand logo on paper, receipt wording |
| 4 | **Kitchen and bar board** (ticket cards, station screen) | Kitchen features. Removed from Session 5 so the owner designs each screen. | Light is mixed across branches (design for bright glare); large item names (about 40 px or more, to measure on the real tablet); no shadows on floor lists | The real tablet, kitchen light, a decision on the high-contrast kitchen theme (item 9) |
| 5 | **Schedule grid and calendar** | HR scheduling. Wide data. | Phone: sideways scroll with the first column pinned, or "open on a larger screen" with a read-only summary, decided per screen | How shifts look; conflict and thin-day rules |
| 6 | **Charts and sparklines** (Director board) | Director and manager reporting. | Owner dislikes charts; common chart types only, descriptive headline, colour-blind-safe palette, no draw-in on mount, tables for raw data (guidelines K6) | Which numbers actually need a chart |
| 7 | **Signature loading animation** | Branded wait state; the Loading component reserves a 32 px slot after 300 ms. | Concepts were Steam, Ticket or Pour; transform and opacity only; never on the floor critical path; static under reduced motion | The brand: the logo (round Wendo badge with a coffee-bean motif) is now available, so the animation can be drawn from the brand's shapes |
| 8 | **The key screens** (waiter order, Director overview, HR people, and the kitchen, payment and approvals screens) | These are screens, not components; they were removed from Session 5 so the owner designs each one by assembling the patterns. | Everything in `docs/design-system.md` | Each screen designed by assembling P1 to P5; clone nodes from the Paper index board |

## B. Themes and variants deferred
9. **Dark theme and a high-contrast kitchen theme.** Light only for now (L-2). The kitchen theme was wanted as a device-set third choice on station devices only. Needs the real kitchen tablet and light.
10. **Time picker on the phone.** Described (two 48 px lists in the bottom sheet), not drawn.
11. **A wizard for very long forms.** Not designed on purpose: long forms scroll inside the Form panel. Revisit only if a real form is too long.
12. **Which four tabs each role gets on the phone.** Decided with each role's screens.

## C. Decisions still open
13. **Department or kitchen-staff filter.** The owner's sentence about "kitchen staff" and a department filter was unclear and is not in the design. "Department" exists in the Filter list. Ask again when the people screens are designed.
14. **Press scale 0.98 versus 0.97.** 0.98 is drawn; decide on a prototype on a real phone.
15. **Glossary entry for HR approvals** (pay changes, leave). "Approval" in the glossary covers cancel, comp, discount and refund only. A domain decision for `domain-model.md`.
16. **Waiter "sales today" without a payment split.** Treated as hidden; unconfirmed (L-17).
17. **PIN hardening.** The 4-digit PIN with no lock-out is recorded as simple for now (L-10). Hardening options (delay after wrong tries, weak-PIN blocklist, 6 digits for approvals, rate limit per device) were declined for now.

## D. Checks that are not design but block confidence
18. **Measure sizes:** Geist subset against 30 kB (the owner declined the download), Base UI, Motion, Sonner, cmdk against the 130 kB JavaScript and 15 kB CSS budgets, blur cost. Foundation build.
19. **Verify the 30 Lucide icon names and paths** against the installed package. Foundation build.
20. **Real-phone checks:** 12 px tab labels in sunlight, the 28 px logo badge on the sidebar, the header under a real Android status bar, the drawer with 20 or more links, press feel, quiet arrival of table rows with people.
21. **Full contrast table and size audit**, skipped in Session 5. Foundation build (token contrast script and Playwright probes).
22. **How office tables get their data** (paged from a server or a local copy): check against `architecture.md` before building the table at scale.

## E. How to design one of these later
Open the Paper file and the index board; clone approved nodes (`x-paper-clone`) instead of redrawing; reuse tokens; draw every state (default, hover, pressed, focus, disabled, loading, error) at office, phone and floor sizes with realistic Kenyan data and worst-case content; compute contrast before writing a number in a caption; mark anything unmeasured; then add the part to `docs/design-system.md` and the index board.
