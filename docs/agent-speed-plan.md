# Agent speed plan: how the design system makes agents fast

Owner decision, 2026-11-05 (Session 5). Status: APPROVED and folded into `docs/design-system.md` section 8 and decision-log section L-15; kept here as the short version. Nothing here is built; the `pnpm` tools named below are specified, not existing.

## 1. Goal
Speed in design and speed in coding with no friction. **No agent designs or builds a component from scratch.** Agents assemble approved components and patterns. If something they need does not exist, they stop and ask; they never invent it.

## 2. Principle
Design once, build once, assemble many times. A feature screen is a pattern filled with components. Anything missing is a request, not an improvisation.

## 3. Inventory (status on 2026-11-05)

### Designed and approved
- Data: KPI strip, table (compact, at scale, phone list), "Needs you" worklist, floor row, money text, filters (quiet chips, Filter button).
- Controls: Button (primary, outline, destructive, text link), Field (text, money, search, closed picker), checkbox, radio, switch, Tabs, Menu, NumberPad (4-digit PIN, amount), Avatar, Progress.
- Messages and shell: Alert, Toast, Sheet and modal, AppShell (desktop, laptop rail, phone with drawer), Empty state, Loading (skeleton).

### To design NOW (needed by almost every screen)
StatusDot and status label, Tooltip, Textarea, Select and combobox (open list, typing to search) with the command palette (Ctrl K), date, time and date-range pickers, quantity stepper, icon component and approved icon list.

Progress 2026-11-05: StatusLabel, Tooltip, Textarea (approved), then Select and combobox, command palette, quantity stepper, date, time and range pickers, icon list drawn on Paper board "Components · controls · batch 4 · for approval", awaiting owner approval. Icon names and paths are UNVERIFIED against the Lucide package.

### Patterns (drawn 2026-11-05 on Paper board "Patterns · page recipes · for approval", awaiting owner approval)
- List page: page header, filters, table, built-in empty, loading and error states.
- Detail panel: 460 px side panel and its phone sheet.
- Form panel (owner 2026-11-05: no full-page forms on the office layout): 520 px right panel with sections, fields, a pinned save bar and an unsaved-changes warning; the phone uses a full-screen sheet.
- Approve with PIN: side panel then PIN sheet.
- Confirm modal: refund-style confirmation.

### Required but NOT designed (left for their screen sessions; agents must stop and ask)
Tile (menu item, table), floor shell (waiter and kitchen devices, always-visible unsent count), receipt, kitchen ticket and payslip print layouts, schedule grid and calendar, charts and sparklines (Director board), signature loading animation (deferred by owner; slot reserved in Loading).

## 4. The six mechanisms
1. **Patterns.** Five named patterns (section 3) are designed once so most feature screens are "fill in a pattern".
2. **One catalogue in the code.** A route in `core/ui` that renders every component and pattern in every state (office, phone, floor). It is the first thing an agent opens, it renders the real code so it cannot go stale, and it replaces the deferred review tool in `docs/design-preview/`. `pnpm see` outlines a screen as text.
3. **Closed primitives and lints.** Components take typed options only: no `className`, no `style` (already approved). Add lint rules that ban raw `<button>`, `<input>`, `<select>`, `<textarea>`, `<table>` and hex or arbitrary colours in feature code, with messages that name the fix (CLAUDE.md rule 14). `pnpm where [word]` lists each component with its options and a one-line example.
4. **Stop and ask.** If an agent needs something not in the catalogue it stops and opens a "component request" issue (new issue template). It does not create the component inside a feature pull request (rule 9). The request is designed in Paper, built as a small separate core pull request, then used.
5. **Order of work.** Core pull requests, each small: (1) `tokens.css` generated from `docs/design-tokens.json`; (2) primitives; (3) data components; (4) shell and patterns; (5) catalogue. Feature agents start after the catalogue exists.
6. **Design speed.** The Paper file gets a clean library page with each component once, named exactly like the code. New screens are assembled by cloning those nodes (`x-paper-clone`), not redrawn.

## 5. Definition of done for a component in code
- Appears in the catalogue in every state, office and phone (and floor where it applies).
- Loading, empty and error states are built in, so a screen cannot forget them (the table primitive also requires each column marked primary, secondary, status, amount or desktop-only).
- At least one test for the real behaviour that would break if the component were deleted (CLAUDE.md rule 12).
- Playwright probes for sizes (28, 48, 56 px), overflow at 200% text, and animations limited to transform and opacity.
- Contrast pairs pass the token script.

## 6. What this does not solve
It specifies the work; none of it is built. The risk is that the foundation core pull requests take time before feature agents can start; the order in section 4.5 keeps each one small. Sizes of Geist, Base UI, Motion and Sonner against the 130 kB JS and 15 kB CSS budgets are still unmeasured.
