# Product Vision: Wendo Business Operating Platform

## 1. Summary

An **AI-native business operating platform** for **Wendo Coffee and Bistro**, a restaurant company in Kenya. Everything about how the company runs lives in one system: floor service, kitchen and bar, inventory, staff, payments, finance and reporting.

- The company has 3 branches today and plans to grow to up to 10. The platform must support that growth without a redesign.
- The purpose is to digitize and automate management, make everything easy to track, and let the company run like the best companies with as few people as possible.
- The platform is AI-native. AI is part of how the system works, not an add-on chatbot. It automates routine work, flags problems and drafts decisions for humans.
- It is custom-built for Wendo first. Serving other companies later is desirable if possible.
- It is an internal tool, used by staff only. There are no customer or guest accounts for now.
- This is a rebuild. A first attempt was made earlier. This version is to be built properly, using agent-driven development and Claude Code to deliver quickly with maximum results.

## 2. Goals

1. Management can see and track everything happening in every branch.
2. Routine work is automated: reconciliation, reminders, reports, ordering suggestions, scheduling drafts.
3. Every order, payment, stock movement and approval is traceable and transparent to the right people.
4. Adding a branch is a matter of configuration, not redesign.
5. The company scales without a matching rise in headcount or overhead.
6. The codebase stays clean and well structured.

## 2a. Quality bar: design, experience and performance

Two qualities matter above all others.

1. **Product design and user experience.** The UI/UX must be best-in-class, top-tier and world-class. Every role, from a waiter on a phone to a director on a laptop to a chef at a kitchen display, should get an interface that feels purpose-built, effortless and polished.
2. **Performance.** The platform must be exquisitely fast and responsive, top-tier and world-class, including in busy service hours and across every device used.

The product should look, feel and perform as if it were built by the best product teams in the world. These two qualities apply to every feature and are not left for the end.

## 3. Company structure

```
Company (Wendo Coffee and Bistro)
 └── Branch
      └── Department
           └── Team members (led by one Department Head per branch)
```

Each branch has its own departments, and each department has its own head in that branch.

| Department | Department head | Team members |
|---|---|---|
| Service | Head Waiter | Waiters |
| Kitchen (includes Pastry) | Head Chef | Chefs, pastry chefs |
| Barista | Head Barista | Baristas |
| Housekeeping | Housekeeping Head | Housekeepers, stewards |

## 4. Users and roles

Permissions are role-based. One person can hold several roles. Staff see only what their job needs. Sensitive actions such as voids, discounts, refunds, price changes and payroll need approval. Sensitive actions are recorded in an audit log.

| Role | Scope | Main devices | What they do |
|---|---|---|---|
| **Director** | All branches and departments | Laptop, phone | Full read and write access to everything. Sees all activity and all pending approvals. Can approve or do anything a branch manager can, at any time, without needing anyone else. Not limited or confined. |
| **Branch Manager** | Their branch | Tablet, laptop, phone | Runs all operations in the branch, makes sure things are going well, and manages the branch through the platform. |
| **HR Manager** | Company-wide | Laptop, phone | Hiring, contracts, leave, attendance, payroll inputs, disciplinary records, performance, training. |
| **Accountant** | Company-wide | Laptop, phone | Reconciliation (including uploading bank statements), expenses, supplier invoices and payments, payroll processing, tax, financial reports. |
| **Store Manager** | Central store, company-wide view | Laptop, phone | Purchasing from suppliers, receiving goods, stock control, dispatching to branches, supplier management, stock counts. |
| **Store Attendant** | Store | Tablet | Records deliveries, issues stock, does counts. Cannot change prices or approve purchases. |
| **Department Heads** (Head Waiter, Head Chef, Head Barista, Housekeeping Head) | Their department in their branch | Per department | Lead their department's work in that branch. |
| **Waiter** | Their branch | Phone or handheld | Takes orders, sends them to stations, manages tables, bills guests, takes payment. |
| **Chef / Pastry Chef** | Their branch | Kitchen display | Work order tickets, mark progress, flag out-of-stock items, log waste. |
| **Barista** | Their branch | Bar display | Works drink tickets, marks progress, logs waste. |
| **Housekeeper / Steward** | Their branch | Phone, shared tablet | Cleaning and hygiene tasks, checklists, issue reports, supply requests. |
| **System Admin** | Platform | Laptop | Technical custodian of the platform: users, roles and scope assignment, branch, department and device setup, integrations, settings, support, backups, system health and audit logs. Kept separate from business approvals and from payroll and financial data. |

There is no Cashier role; waiters take payment. There is no Guest or Customer role.

Directors need no second approver for their actions, and those actions are still recorded.

## 5. Core operating flows

### 5.1 Service flow

1. A guest arrives and the waiter takes their choices.
2. The waiter places the order in the system.
3. The order goes to the relevant station: kitchen, pastry or bar.
4. Kitchen staff or baristas prepare it, working from the display at their station.
5. When it is ready, the waiter is notified.
6. The waiter delivers it and the guest eats.
7. The waiter charges the guest based on the order.
8. If the guest pays by M-Pesa, the waiter sends an STK push to the guest's phone. The transaction is recorded.
9. Each order keeps full order details and payment details.
10. Throughout the day the system holds a record of all orders, including cancellations and returns.

### 5.2 End-of-day reconciliation

1. The accountant uploads documents such as bank statements.
2. The system (AI) checks each entry against the order records.
3. Every order is matched and any mismatch is flagged.
4. The outcome is that all orders are reconciled and everything is clean.

### 5.3 Inventory and supply

- A central store buys all items from suppliers.
- Each branch requisitions stock from the central store.
- The store dispatches to branches. Branches receive supplies daily, or less often for distant branches.
- The system tracks how each branch uses its stock.
- Everything is traceable, transparent and visible to the relevant people, and connects to payroll and payments where relevant.

### 5.4 Management and oversight

- Chefs work orders on kitchen display screens at their workstations.
- HR and branch managers manage the branch through the system.
- Directors see everything happening everywhere, see all pending approvals, and can act on them from their own account whenever they choose.

## 6. Feature catalogue

This is a broad list of features that could benefit the platform. It is a prospective list, not a build scope. Detail for each feature is given when that feature is planned.

**A. Orders and service**
Digital order management; tables and floor plan; modifiers, notes and allergens; courses; routing to kitchen, pastry or bar; split, merge and transfer bills; voids, discounts and comps with approval; cancellations and returns; takeaway and delivery orders; tips and service charge.

**B. Kitchen and bar displays**
Live tickets per station; preparation timers and late alerts; out-of-stock flags that reach waiters; order-ready notification to the waiter.

**C. Menu management**
Items, categories and prices per branch; recipes and costing; availability schedules; menu changes pushed to all branches.

**D. Payments and cashiering**
Cash and M-Pesa (STK push); receipts; shift and till reconciliation; tax receipts.

**E. Inventory and procurement**
Central store; supplier management; purchase orders; goods receiving; branch requisitions; dispatch to branches; usage tracking per branch; counts, variance and wastage; recipe-based stock deduction; reorder suggestions.

**F. Workforce**
Employee profiles and documents; shift scheduling; swaps and availability; clock in and out; leave; payroll inputs and payslips; performance, training and disciplinary records.

**G. Operations and tasks**
Opening and closing checklists; cleaning and hygiene tasks; maintenance and issue reporting; food safety logs.

**H. Finance**
Daily sales and cash-up; bank statement reconciliation; expenses and petty cash; supplier invoices and payables; profit and loss per branch; budgets and consolidated reporting.

**I. Reporting and analytics**
Live director dashboard; branch comparison; menu engineering; labour cost against sales; exportable reports.

**J. Approvals**
A shared approvals mechanism for discounts, voids, refunds, price changes, payroll, requisitions, leave and similar, with a pending-approvals view for directors.

**K. Platform foundations**
Roles and permissions; company, branch and department structure; audit log; notifications; offline resilience; settings, integrations and data import.

## 7. AI-native layer

AI is built into how each feature works, and each feature gets its AI capabilities as it is built.

- **Ask anything:** plain-language questions to the system about the business.
- **Daily briefings** for directors and managers.
- **Anomaly detection:** unusual voids, discounts, stock variances or cash shortages.
- **Forecasting** of sales and demand, and smart reorder suggestions.
- **Auto-scheduling:** draft rosters from demand and labour rules.
- **Document automation:** read supplier invoices and delivery notes, and draft purchase orders.
- **Reconciliation:** match uploaded bank statements to orders.
- **Task automation:** chase approvals, send reminders, escalate overdue items.

AI suggests and drafts. Humans approve anything involving money, people or compliance.

## 8. How the product will be built

- The platform is built feature by feature, a small set of features at a time, to keep the codebase clean. Not everything is built at once.
- Foundations come first: company, branch and department structure; users, roles and permissions; approvals; audit log; notifications.
- Each feature is then built end to end before the next begins.
- **Feature detail comes just before each feature is built.** The product owner will describe how each feature should work at the start of planning for that feature. This document gives the overall vision and the prospective feature list. It does not specify features in detail.
- The codebase must be very well structured, with clear boundaries between features.
- The data model supports multiple branches from the start and can be extended to multiple companies later.
