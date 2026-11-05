# Wendo design system

Output of Session 5 (2026-11-05), designed with the owner in Paper. This file is for coding agents and reviewers. Where it differs from `architecture.md` section 7, this file and decision-log section L win. Tokens are in `docs/design-tokens.json` (the source of truth). The design is drawn in the Paper file "wendo-management-platform-design-system" (index board: "Index · component names and where they are drawn").

**Status.** Everything here is designed and approved by the owner. Nothing is built: `core/ui`, `tokens.css`, the catalogue and the `pnpm` tools are specified only. Items marked UNVERIFIED or UNMEASURED were not checked; section 11 lists them.

## 1. The one rule that matters

**Assemble; do not design.** Every screen is a pattern (section 7) filled with components (section 6). If a screen needs a part that is not in this file, stop and open a component request (section 8). Do not draw it, build it inside a feature, or copy something similar.

## 2. Principles

- P1 **Frequency first.** Floor screens are used all day: feedback on touch-down, no decoration. Office screens may use standard motion.
- P2 **Quiet.** No cards, no decoration, no unnecessary ticks or crosses. Information sits on the page, grouped by space and hairlines.
- P3 **Fewer clicks.** Whole rows and whole labels are the target. Smart defaults and remembered filters.
- P4 **Status is a word.** Never colour alone. Dot or icon plus a written word.
- P5 **Errors name the fix.** Next to the problem, no blame, no "oops".
- P6 **Nothing moves under your hand.** Rows arrive and update without shifting what you read or are about to click.
- P7 **The floor never stops.** Floor screens open from the saved copy, never show a spinner, and show the unsent count whenever it is above 0.
- P8 **Inherited language.** The look is the previous system's (ruled editorial ledger, mono uppercase labels, quiet zeros, dot-and-word status, underlined links), improved. It is not new (decision-log L).

## 3. Layouts and sizes

The layout is chosen by width and device, never by pointer type. An office laptop with a touch screen still gets the office layout (owner, 2026-11-05).

| Layout | When | Control height | Text | Row | Notes |
|---|---|---|---|---|---|
| Office | laptop, 768 px and wider | 28 px (field 28, button 28, tab 32) | 13 px, prose 14 px | data table 28 px, worklist 40 px | sidebar 236 px or 56 px rail under 1280 px, global bar 48 px |
| Phone | under 768 px | 48 px | 16 px, label 14 px | list 56 or 64 px, rows 48 px | header 56 px plus safe area, tab bar 56 px, inputs 16 px |
| Floor device | waiter, kitchen, bar | 56 px for primary actions, 48 px otherwise | system font 16 px | floor row 56 px | system font, no skeletons, always-visible unsent count |

- S1 Targets are at least 28 px high in the office, 48 px on the phone and floor, with 8 px between touch targets. A visual smaller than its target gets a transparent hit area.
- S2 Press feedback scales to 0.98 and responds on touch-down within one frame (0.97 versus 0.98 is not decided; 0.98 is drawn).
- S3 The page treatment is one: every page fills the width with 24 px gutters. A component keeps its own sensible width inside the page (a field about 360 px, a paragraph at most 640 px).
- S4 Desktop forms are never full pages. They open in the Form panel (section 7, P3).

## 4. Tokens (summary)

Full values in `docs/design-tokens.json` (about 189 entries); Paper holds 114 of them as variables. Gradients and shadows live only in the JSON.

- **Colour.** Cool neutrals (hue 255, eleven steps from `#FCFCFC` to `#131618`), espresso and caramel scales, earthy status colours (success `#2F6438`, warning `#8A5A16`, error `#97281D`, info `#2C5670`, each with a background and a border). Roles: page `#FCFCFC`, surface `#FFFFFF`, ink `#131618`, muted text `#5A5F66`, faint text `#666C73`, control border `#8E939B` (3:1 on the page and white only), primary `#693C1B`, link `#A65700` (always underlined), focus ring a 2 px ink ring with a 2 px gap (no blue). Caramel is decoration, never text. Brand gold, olive and cream are for brand moments only and never mean a status.
- **Gradients** only on the sidebar (`#2E1806` to `#0F0601`), the phone header (`#241609` to `#4E2C14`), section rules and progress fills. Never on buttons, tables, text blocks, floor lists, kitchen boards or disabled states. No grain.
- **Type.** Geist and Geist Mono for the office, the system font stack for the floor. Sizes: KPI 28/34, title 24/30, section 15/20, body 14/20, body-small 13/18 (tables), caption 12/16, mono label 11/14 uppercase with 0.04em tracking (labels only), floor body 16. Weights 400, 500, 600. Tabular figures on every number.
- **Space.** 4 px grid: 0, 1, 2, 4, 6, 8, 12, 16, 20, 24, 32, 40, 48, 56, 64.
- **Corners.** 0 px for tables, panes, rules and buttons; 4 px for inputs, tabs, menu items and checkboxes; 6 px for floating layers (menus, popovers, sheets, modals, toasts, tooltips); circles only for avatars, status dots and count badges.
- **Elevation.** Surface (hairline), raised, floating, modal; ink-tinted neutral shadows. Floor lists use borders, never shadows.
- **Glass.** Thin and regular levels on floating layers on office laptops only, with a solid fallback. Never on the sidebar, working panes, the floor, or any panel that carries a form or an action link (toasts, modals, panels, calendars and the command palette are solid).
- **Motion.** Three curves (ease-out, ease-in-out, drawer), durations 120, 200 and 320 ms, only `transform` and `opacity`, reduced motion means fades, never zero feedback. No generic spinners.
- **Layers (z).** base 0, sticky 10, sidebar 20, dropdown 30, sheet 40, modal 50, toast 60.

## 5. Rules (with the tier that enforces each)

Tier: **[T]** a tool fails the build, **[P]** built into a primitive so screens cannot forget it, **[R]** reviewed by a person or agent. The tools are specified, not built (section 9).

### Colour and type
- C1 Use tokens only; no hex or arbitrary values in feature code. [T]
- C2 Contrast: text 4.5:1, large text and UI parts 3:1, computed unrounded (WCAG 2.2). A token pair must pass the token contrast script. [T]
- C3 One colour, one meaning. State colours are never the brand accent, and status is never colour alone. [R]
- C4 One primary (brown filled) action per screen. [R]
- C5 Caramel and brand colours are never text and never a status. [R]
- T1 Tabular figures on every number; `MoneyText` formats whole shillings with "KSh" (`KSh 1,250`) and never truncates. [P]
- T2 Never truncate amounts or dates. Wrap names. Cut codes in the middle. [R]
- T3 Floor body at least 16 px, tertiary at least 12 px, tappable at least 14 px. [T]
- T4 The mono uppercase style is for labels only. [R]

### Interaction
- I1 Respond on touch-down; press scale 0.98; `touch-action: manipulation` on controls. [P]
- I2 Hover only inside `(hover: hover) and (pointer: fine)`; hover-only actions also exist in the detail panel or sheet. [T]
- I3 Everything is keyboard operable with a visible focus ring that is never covered; no custom gesture without a tap alternative. [P]
- I4 Update optimistically; offer Undo; confirm only hard-to-undo actions (Confirm modal). [R]
- I5 Do not pre-disable a submit button: show what is missing. [R]
- I6 Errors appear when leaving a field, keep what was typed, and name the fix. [P]
- I7 A disabled control says why and who can do it (tooltip in the office, written text on the phone). [R]
- I8 Statuses never fade; only a plain "Saved" toast fades. The unsent count is always visible above 0. [P]
- I9 Nothing you are reading or about to click may move; new rows arrive with a pale tint that fades over 200 ms. [P]
- I10 Inputs on the phone are 16 px so the page never zooms; never disable zoom. [T]

### Motion
- M1 Animate only `transform` and `opacity`. No `transition: all`, no ease-in on UI, no scale-from-zero entrances. [T]
- M2 Exit is faster than enter. Popovers grow from their trigger; dialogs stay centred. [P]
- M3 Stagger 30 to 80 ms, at most about 300 ms, never on floor lists. [R]
- M4 Under reduced motion everything fades; feedback is never removed. [P]

### Writing
- W1 Plain words, "you", verb-first sentence-case labels. "Tap" on touch, "click" never on touch screens. No "oops", no blame.
- W2 Use glossary words exactly (Cancel before payment, Refund after, never "void"; Posting never on floor screens). [T]
- W3 Buttons are verbs that name the effect and the count ("Send 89 payslips"); never OK, Yes or Confirm. [R]
- W4 Dates are "12 Mar 2024"; time is 24-hour "08:30"; money is "KSh 1,250". [R]

## 6. Components

All components are closed: they take typed options and no `className` or `style`. Built on Base UI where a hard widget is needed (the architecture's React Aria choice is replaced, section L). Code names are the names in the Paper index board. Office, phone and floor sizes follow section 3.

### Data
- **KpiStrip.** Mono label, brown number, caption. Red or amber numbers always carry a word. Quiet zero uses faint text. Two columns on the phone, never a sideways scroll. Loading skeleton.
- **DataTable** (code name chosen because the glossary reserves "Table" for a place where guests sit). 28 px rows, 0 px corners, right-aligned tabular numbers, neutral header tint with the sort direction, hover shows a row action link, selected row is pale espresso with a 2 px bar, keyboard focus, disabled at 45%, loading skeleton, inline error with "Try again", pinned totals row. Long names wrap; amounts never cut. Each column must be marked primary, secondary, status, amount or desktop-only; the build fails otherwise (the phone shows a ruled list of 56 px rows built from those marks). At scale: virtualised rows, pages of 100, the count in the header ("47 of 1,284"), pinned header and totals, Back to top, end of list, load-failed row, no-matches row. Rows arrive on their own with no "new rows" bar.
- **ActionList** ("Needs you"). A worklist with 40 px rows: unread dot, age (red when overdue), kind as dot and word, name then detail on one line, amount, chevron on hover. Sorted by urgency, filtered by kind with tabs, no column header. A whole-row click opens the Detail panel. Phone: two-line 64 px rows.
- **FloorRow.** 56 px, system font 16, the whole row is the target. Four send states with icon, word and colour: Sending, Saved, Waiting to send, Flagged. Waiting and Flagged never fade.
- **MoneyText.** See T1.
- **TableFilters.** One row: search, Filter button (key F), quiet toggle chips (off: muted text, no fill; on: light neutral fill and heavier text; no tick, no cross), the count. At most three quick filters, extras collapse into "+2 more"; fields with 4 or fewer values are chips, many values is one dropdown chip, everything else is Filter button, column, value. Same field is "or", different fields are "and". Locked chips show a padlock. Phone: the Filter button opens a full-screen sheet.

### Controls
- **Button.** Four kinds: primary (flat `#693C1B`, 0 px corners), outline (3:1 control border), destructive (filled red only on the final confirm step), text link (underlined). States: default, hover, pressed, focus, disabled (45%), loading (same label and width, a 2 px line slides along the bottom, a second tap is ignored). 28 px office, 48 px phone, 56 px floor. Phone action pairs are full width and pinned.
- **Field.** Label always above (never a placeholder), hint, error that names the fix with an icon and a 2 px border. 4 px corners, 28 px office, 48 px phone with 16 px text. Variants: text, money (KSh outside the digits, right-aligned), search (no label, has an accessible name), picker. Disabled and read-only use a neutral tint and always give the reason. **Textarea** is Field with lines: 3 lines to start (72 px office, 96 px phone), grows to 8 lines, count only when there is a limit.
- **Checkbox, Radio, Switch.** The whole label row is the target (28 px office, 48 px phone). Checkbox 16 px with 4 px corners and a thin tick; the on state is brown. Switch is a flat 4 px track with a square thumb and always shows a word (On, Off, Saving…); it applies at once and never needs a Save button. Radio is a circle, because the shape tells "one of many". A partly selected checkbox shows a dash and names the part.
- **Tabs.** Underline, 32 px, counts in the label in a lighter weight, selected shown by darker text, heavier weight and a 2 px brown bar. Phone: 48 px, at most four tabs plus Menu in the tab bar; tabs inside a page never scroll sideways.
- **Menu.** Floating layer, 6 px, glass on laptops. Items 28 px with 4 px corners and verb labels, one level only, a check lane, shortcut hints on the right, "…" after a label means the action asks for more. A destructive item is last, below a separator, in red text, never filled. Keyboard highlight has a fill and a 2 px ink ring. Phone: a bottom sheet with 48 px rows and a Cancel.
- **Select and Combobox.** Four or fewer choices are radios on the page; 5 to 12 is a Select; more than 12 is a Combobox where typing filters (first match highlighted, matched letters bold, the count while filtering, a no-match message that names the text). Phone and floor: a bottom sheet with a 48 px search field; the floor login person picker is this sheet.
- **CommandPalette.** Ctrl K, office only, 640 px, solid panel over a scrim. Groups in a fixed order (Pages, Actions, People), at most five rows each, recent pages when empty, permission-aware. An action never approves or refunds straight from the palette. Built with cmdk (office only).
- **NumberPad.** Our own pad; the display uses `inputmode="none"` so the system keyboard never opens. PIN is 4 digits, submits at the fourth, no action key. Amount mode: whole shillings, a "00" key, no decimal key, the action key is a tall key at the bottom right named for the job ("Add payment") and disabled at 0. Keys 48 px (phone) and 56 px (floor). A wrong PIN turns the dots red for 400 ms with a small shake (none under reduced motion) and clears.
- **QuantityStepper.** Minus, number, plus; 56 px floor, 48 px phone, 28 px office. At 1 the minus becomes a bin and removal shows an Undo toast naming the item. Tapping the number opens the NumberPad. A limit is written next to the line, not only greyed.
- **DateField, TimeField, DateRangeField.** Typed or picked, one format "12 Mar 2024", week from Monday, 24-hour time with minutes in steps of 15, range presets (Today, This week, Last 7 days, This month, Last month). The range shows through the two brown end days and is written in the field. Phone: a bottom sheet with 48 px days and a button that names the choice ("Choose 12 Nov"). Dates are shown in Africa/Nairobi and stored in UTC.
- **Avatar.** Initials only (V4): on dark a translucent cream circle, on light a pale espresso fill with a hairline. 28 px in rows, 44 px in headers. Initials are the first letters of the first and last word. A group is a row with 4 px gaps and no overlap, at most three then "+N".
- **Progress.** 6 px bar (8 px on the phone), 0 px corners, brown or olive gradient fill (each at least 3:1 against the track). The number is always written next to the bar; over the limit the bar is full and red words say how far over; unknown length shows a sliding segment.
- **StatusLabel.** Dot plus word (office) or icon plus word (floor). Five tones: success, warning, error, info, neutral. One component and a written mapping of words to tones. Never a pill or a bare dot. (The architecture calls this `StatusDot`; renamed because it is never a dot alone.)
- **Tooltip.** Office only. An ink panel, 6 px corners, opens after 500 ms of hover or at once on keyboard focus, never holds interactive content. It names icon-only buttons and gives the disabled reason. Nothing important lives only in a tooltip.
- **Icon.** Lucide, 1.5 px stroke, 16 px office, 20 px phone and floor, 24 px for the NumberPad delete key, colour follows the text. Icons alone (without a word) are limited to close, search, filter, more, bell, menu, stepper plus and minus, and picker chevrons. The icon list in Paper is UNVERIFIED against the package.

### Messages and layers
- **Alert.** A ruled band in the page, 0 px corners, no card or side stripe. Title names the problem, text names the fix, at most two verb actions. Errors and warnings never close by themselves; only info and success may be dismissed. Phone: the action is a full-width 48 px button.
- **Toast.** A solid white 6 px panel (the only floating layer that is never glass, because it carries actions). Bottom right on the office layout, full width above the tab bar on the phone. Statuses never fade; only "Saved" fades after 4 s. Undo lasts 8 s with a shrinking 2 px line and names its result. At most three show. Sonner is the engine, wrapped so every toast is this panel.
- **Sheet and Modal.** Side panel, centred modal and phone bottom sheet. A modal is used only for hard-to-undo choices, forms that must not be lost, or actions that affect others; everything else is a toast with Undo. One at a time, never a modal over a modal. Esc, the close icon and a click on the scrim close it, and with unsaved input they ask first. Opens in 200 ms, closes in 120 ms. Floor devices use the phone sheet.
- **AppShell.** Office: sidebar 236 px (the S1 gradient, a branch scope row on top, signed-in person with the avatar at the bottom, 28 px rows, a count only for things that need the person, collapses to a 56 px icon rail under 1280 px with Ctrl B), a 48 px global bar (breadcrumb only when two or more levels deep, a quiet connection status that shows a word only when something is off, search that opens the command palette, the bell). The page header lives in the content: title, one sentence, the page's one primary action, page tabs. Phone: header H1 (page title, bell, avatar), four tabs chosen per role plus "Menu", which opens a 320 px drawer of the same grouped links with 48 px rows; no hamburger; no edge swipe; the header does not hide on scroll.
- **EmptyState.** Text only, left aligned where the data would be, no illustration. Says what is empty, why, and one next step. Four causes with four texts: first use, filters match nothing (names the filters, offers "Clear all filters"), all done (no button), no access (names who can see it). A load failure is not an empty state.
- **Skeleton (Loading).** Appears at once with the real header, labels and row height; only values are grey bars. Opacity pulse only, static under reduced motion. A signature animation slot (32 px) appears after 300 ms and stays at least 500 ms; the concept (Steam, Ticket or Pour) is deferred. After 10 s a warning band offers "Try again". Floor screens never show a skeleton.

## 7. Patterns (page recipes)

A pattern is a fixed arrangement of components with its states built in. Agents fill the slots and nothing else.

- **P1 ListPage.** PageHeader, then TableFilters, then DataTable (ruled list on the phone), then pinned totals. Built-in states: loading, empty first use, empty after filters, error (saved rows stay, marked with their saved time), offline. Remembers filters per table.
- **P2 DetailPanel.** 460 px right panel over the page (no scrim), phone full-screen sheet. Header, Facts (32 px rows, 48 px on the phone, key value in semibold), History, a pinned decision area. Up and Down move to the next record. If the record is decided elsewhere the buttons are replaced by the outcome. Hover-only table actions live here.
- **P3 FormPanel.** 520 px right panel with a light scrim, a pinned Save bar ("2 unsaved changes", Discard changes, Save changes), an error summary after a failed save with links to each field, sections with 15 px headings, one column of fields that fill the panel. Add and Edit are the same panel. Close with unsaved changes opens the Confirm modal. Phone: a full-screen sheet with one column and a pinned Save. There is no full-page form on the office layout.
- **P4 ApproveWithPin.** Detail panel, then a PIN modal (office, keyboard digits) or sheet (phone and floor, NumberPad), then a result toast without Undo and the panel shows the outcome; in a queue the next waiting record opens by itself. The PIN screen names the signed-in person, the action and the amount, and the PIN is checked against that person only.
- **P5 ConfirmModal.** Three variants and nothing else: destructive (red button naming the amount), affects others (brown button naming the effect and count), losing input ("Keep editing" is primary, "Discard changes" is outline). The safe choice takes the focus; Esc and a click outside choose it.

## 8. How agents stay fast

1. **Patterns and the catalogue.** Most screens are a pattern with slots. A catalogue route in `core/ui` renders every component and pattern in every state (office, phone, floor). It is the first thing an agent opens and it replaces the deferred `docs/design-preview/` review tool. `pnpm see <route>` outlines a screen as text.
2. **Closed primitives and lints.** No `className` or `style`. Lint rules (Biome GritQL for JavaScript and CSS) ban raw `<button>`, `<input>`, `<select>`, `<textarea>`, `<table>` and hex or arbitrary colours in feature code, with messages that name the fix. `pnpm where <word>` lists each component, its options and an example.
3. **Stop and ask.** If a part is missing, the agent opens a "component request" issue (a new issue template). The part is designed in Paper, built as a small core pull request (CLAUDE.md rule 9), then used. Agents never create it inside a feature.
4. **Order of work.** Small core pull requests: tokens, primitives, data components, shell and patterns, catalogue. Feature agents start after the catalogue exists.
5. **Design speed.** New screens are assembled by cloning approved Paper nodes (`x-paper-clone`), not redrawn. The index board lists each node.
6. **Definition of done for a component in code.** In the catalogue in every state; loading, empty and error states built in; a test for the behaviour that would break if it were deleted; Playwright probes for sizes, overflow at 200% text and transform-and-opacity-only animation; contrast pairs pass the token script.

## 9. Enforcement design (specified, not built)

Tokens reset with `--*: initial` so only defined tokens exist; closed primitives (no `className`, no `style`); `tools/arch` import rules (a feature imports only `core/*` and other features' `public.ts`; floor code never imports company features; server files are never imported by client code); Biome rules for JavaScript and CSS; a token contrast script; Playwright probes (48 px targets, overflow at 200% text, `getAnimations()` limited to transform and opacity, axe-core); generator templates; the rule-break suite. The table primitive requires phone column marks. The `pnpm` tools do not exist yet.

## 10. Staff writing guide

- Plain words, "you", no jargon, humour or slang. Verb-first, sentence case. Consistent flow words (Continue, Done).
- Say what happened, why, and the fix: "Enter 10 digits, for example 0712 345 678. You have 8."
- Glossary words: Cancel (before payment), Refund (after), Comp, Discount, Approval, Posting (office only), Bill, Payment, Order, Branch.
- Offline wording: "Order saved. Waiting to send." Order numbers and errors are selectable.
- Buttons name the effect and the count ("Refund KSh 1,250", "Send 89 payslips"). The safe option is "Cancel", "Not yet" or "Keep editing", never "No".
- The glossary's "Approval" covers cancel, comp, discount and refund. The designs also approve pay changes and similar HR requests; the glossary needs an entry for these (decision-log L, open items).

## 11. Known gaps (see decision-log L for decisions and risks)

- **Not designed, required later (agents stop and ask):** Tile (menu item, table), the floor shell (waiter and kitchen devices), print layouts (receipt, kitchen ticket, payslip), the schedule grid and calendar, charts and sparklines, the signature loading animation (deferred).
- **Unmeasured or unverified:** Geist subset size against 30 kB (download declined), Base UI, Motion, Sonner and cmdk sizes against the 130 kB JS and 15 kB CSS budgets, blur cost, press feel and the 0.98 versus 0.97 scale, contrast in bright light and in the kitchen, the 12 px tab labels in sunlight, the logo badge at 28 px, quiet row arrival with people, the drawer with 20 or more links, tabular figures in Geist, the Lucide icon names and paths, how office tables get their data, offline behaviour of approvals. The full contrast table and size audit were skipped by the owner in this session.
- **Review tool deferred.** The in-code catalogue replaces it. The agent speed plan is also kept as `docs/agent-speed-plan.md`.
