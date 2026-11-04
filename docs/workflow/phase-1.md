# Phase 1: Understand and decide

My workflow for starting a large software project with coding agents. This phase happens before any feature is built.

**Purpose.** Make the few decisions that are expensive to change later, record why, and make sure agents will work fast and smoothly. No product code is written. Only throwaway experiments, which are deleted afterward.

**Worked example (imagination only, not being built).** A restaurant management system for a 10-branch company. See [the example product](#the-example-product) at the end.

---

## 1. Principles

1. **Agents check their own work.** If they can't, I become the checker and the bottleneck.
2. **Strong foundation, built to extend.** Start with the foundation every feature relies on, not a small first feature.
3. **One repeatable shape for every feature,** created by a generator command, so adding the next one is easy.
4. **Settle uncertain questions with small experiments,** not arguments or long paper plans.
5. **Never accept the first design.** Compare several, then choose.
6. **Enforce rules with tools** (types, lint, tests), not with documents. If a rule matters, breaking it should be impossible or loudly fail.
7. **Keep documents few and short.** The code is the real record.
8. **Prove it, don't claim it.** Agents show evidence: screenshots, test results, measurements.

## 2. Handling unknown features

Sort decisions by how hard they are to change later.

- **Hard to change, so decide carefully now:** the core data model, how data is separated (for example by branch), roles and permissions, how money is stored, the audit trail, how code is organized.
- **Cheap to change, so decide when we get there:** individual features, screens, reports, workflows.

Practices:
1. **Stress-test with imagined future features.** List features that might come later and ask the structure "what would this change?" If the answer is "everything," the foundation is wrong. If it's "add a module," it's right.
2. **One repeatable shape per feature,** produced by a generator command.
3. **Three feature buckets:** known, likely, unknown. Only known and likely shape the design. Unknown is covered by the stress-test.

## 3. Speed and flow

Agents move fast when answers come in seconds. Targets, to be confirmed by measuring on my real laptop:

| Tier | Where | Time | What runs |
|---|---|---|---|
| **1. Fast loop** | Laptop | Seconds (about 10s type check and lint, about 30s related tests) | Type check, lint, tests for the changed module |
| **2. Full check** | GitHub Actions | A few minutes | Full tests, browser tests, visual checks, production build |
| **3. Merge gate** | GitHub Actions | A few minutes | Same checks, required before merging |

Rules:
- **Full test suites and production builds never run on the laptop.**
- Agents verify against the fast development server, never the slow production build.
- Most work should be verifiable with fast tests on plain logic, with no dev server, browser or full database.
- **Right tests, not many tests.** Heavy on money, stock, permissions and payment callbacks. Light on simple screens. Every test protects observable behavior and must answer "what real behavior breaks if this is deleted?" Tests use a real database that resets quickly, not piles of mocks.
- One command starts the app, with fake data and a test user for every role. External services (such as payments) are replaced with fakes. The clock can be controlled.
- Errors tell the agent what to do next. Lint messages name the fix.
- Each feature is self-contained, so an agent reads and checks only one small area.
- A **generator command** creates a new feature in the correct shape (folders, files, wiring and a starter test that already runs).

## 4. Parallelism and hardware

Agents themselves are cheap. What they start (dev servers, databases, browsers) is expensive.

My laptop: Intel i7-8665U (4 cores), about 15 GB RAM, already near full. So:
- **Memory budget per agent.** Written down and measured.
- **One shared database server with a separate database per agent,** not a full database server per agent.
- **One shared browser with isolated sessions,** not a full browser per agent.
- **Each agent works in its own copy of the code** (worktree), with its port, URL and database name assigned automatically.
- **Independent features.** Two agents on different features should almost never edit the same file. No central files everyone must edit. Features register themselves automatically.
- **Clear rules for things that collide:** database migrations and shared types.
- **A light mode.** Most tasks are verifiable with fast logic tests alone. The full app runs only when needed.
- Heavy work goes to the cloud (section 5). A remote workspace is a later option.

## 5. GitHub as the control center

| Item | Role |
|---|---|
| **Issue** | A task, with acceptance criteria that can be checked |
| **Branch and pull request** | One agent's unit of work, with evidence attached |
| **Actions** | Runs the heavy checks in the cloud, including headless browser tests, screenshots and videos |
| **Required checks** | The merge gate. Needs the Team plan for private repos |
| **Artifacts** | The proof: screenshots, videos, reports |
| **Pull request template** | Forces each PR to state what changed, the evidence, and the remaining risks |
| **Dependabot** | Free dependency updates |
| **Secret scanning** | A free scanner in Actions for now. Secret Protection is a paid add-on to consider later |

Free plan: 2,000 CI minutes per month. Team plan ($4 per user per month): 3,000 minutes, plus repository rules and required reviewers. Confirm current numbers on GitHub's pricing page.

Stretching the minutes: cache dependencies, cancel superseded runs, run only checks relevant to changed files, run browser tests on pull requests only.

Cloud screenshots are the reference for visual checks. Generate baselines in the cloud, not on the laptop, because fonts and rendering differ. Cloud browser checks are scripted and repeatable. Agents still use a light local browser for exploration.

## 6. Tools to evaluate in Session 2

| Tool | Use | Watch for |
|---|---|---|
| **agent-browser** | Agent-driven browsing as short commands, with the page read as a compact text outline | Memory per session. Test its lighter engine option. Complements, not replaces, Playwright |
| **Playwright** | Repeatable browser tests in the cloud | |
| **portless** | Named URLs and automatic ports per agent | Pre-1.0. Prefer plain HTTP (`--no-tls`) on a dev machine |
| **opensrc** | Lets agents read the real source of dependencies | Experimental |
| **webreel** | Optional proof videos from CI | Experimental. Playwright already records video |
| **skills.sh** | Reading material for ideas. Not blind installs | Read a skill before installing it |

Not needed: json-render, just-bash, fx. Worth a later look: emulate, lat.md, deepsec.

Pin tool versions, since many are young.

## 7. The sessions

| # | Session | What happens | Output |
|---|---|---|---|
| 1 | **Discovery** | The agent interviews me, restates the product in its own words, finds gaps and challenges assumptions. I correct it. Features are sorted into known, likely and unknown. | Product brief |
| 2 | **Domain, stress-test, experiments** | Define the core things and how they relate. Stress-test against imagined future features. Run small throwaway experiments on the risky parts. Measure tool speed and memory on my real laptop. | Domain model |
| 3 | **Architecture competition** | Several fresh agents each propose a code structure from the same inputs. A separate judge compares them against a rubric. I choose. | Decision log entry |
| 4 | **Agent experience design** | Set speed targets, test policy, memory budget, three-tier checks, the environment, the GitHub pipeline and the rules enforced by tools. | Agent experience spec |

### Stress-test vs experiments
- **Stress-test:** thinking on paper. "Does the structure have room for this feature?"
- **Experiment:** a few dozen lines of real code, run in a scratch folder and then deleted. "Does this actually work the way we assume?" Its output is an answer, not product code.

### Architecture competition
- **Inputs (same for every agent):** product brief, feature buckets, domain model, hard rules, stack, experiment results, future-feature scenarios and the judging criteria. Each agent also gets a different emphasis (for example simplicity, extensibility, correctness of money and stock) so the answers don't converge.
- **Each agent proposes a design package, not working code:**
  1. A usage sketch: how a developer adds a feature.
  2. A folder and module map.
  3. Core types and data model.
  4. Public boundaries between modules.
  5. A short rationale, including weaknesses.
- **The judge scores:**
  - Places to change to add a feature.
  - Whether an agent finds where code goes without documentation.
  - Whether invalid states are impossible to write.
  - Survival of the future-feature scenarios.
  - Isolation of money and stock logic.
  - Simplicity.
  - Feedback-loop speed.
  - Memory use.
  - Ease of parallel work.
- The judge recommends a base design and which good ideas to borrow from the others. I make the final call.
- Only worth the cost for architecture, where mistakes are expensive. Not for everyday features.

## 8. Outputs of Phase 1

1. **Product brief** (including the feature buckets).
2. **Domain model** and the non-negotiable rules.
3. **Decision log:** each choice, the alternatives, the reason. Includes risks and experiment results.
4. **Agent experience spec:** speed targets, test policy, memory budget, environment, GitHub pipeline, enforced rules.

## 9. Phase 1 is done when

- I can explain every major decision and why it was made.
- The foundation passes the future-feature stress-test.
- Speed and memory targets are measured on my laptop and are achievable.
- Every important rule has a tool that enforces it.
- The cloud pipeline is designed and fits the free minutes, or the Team plan is chosen.

---

## The example product

A web-based restaurant management system for a company with 10 branches, used by everyone from waiters to accountants. Built from scratch, fully online, internal tool for one company.

**Order flow.** A waiter opens a table and takes the order on their phone. The order goes to the station screen (kitchen, bar or coffee, each with its own screen). When guests finish, the waiter bills them and sends an M-Pesa STK push to the guest's phone. The system records the payment when M-Pesa confirms and the order closes. There are no cashiers.

**Billing** includes tips, service charge, discounts, voids and comps. Voids, comps and discounts need approvals and an audit trail.

**Inventory** deducts automatically from recipes when an item is sold.

**Roles:** waiters, chefs and baristas, store staff, branch managers, HR, accountants, head office.

**Accountants** reconcile closed orders against the bank statement and the M-Pesa statement.

**Modules:** orders and payments, menu and pricing, inventory, staff and HR, finance and reconciliation, administration and permissions, reporting.

**Hard parts:**
1. Money, because STK push callbacks can arrive late, twice or never.
2. Reconciliation of orders against statements.
3. Real-time delivery to station screens.
4. Permissions and branch isolation across 8 roles and 10 branches.
5. Inventory deduction, including a void after an item was made.

**Experiments for this example (Session 2):**
- Billing math in integer shillings with service charge, discount, comp and tip.
- Inventory deduction and reversal on a void.
- Duplicate, late and out-of-order payment callbacks against a fake M-Pesa.
- Order-to-screen delivery time.
- Whether a Branch A user can ever read Branch B data.

**Imagined future features for the stress-test:** loyalty points, reservations, delivery, supplier ordering, multi-currency.

**Candidate stack (to be confirmed in Phase 1):** TypeScript, Next.js, a small background worker, PostgreSQL, Docker Compose behind a reverse proxy on one DigitalOcean server, nightly backups to object storage.

**Hard rules:**
- Money is stored as integer amounts, never decimals or floats.
- Payment callbacks are idempotent.
- Every void, comp and discount records who approved it.
- Permissions are enforced on the server and scoped to branch.
