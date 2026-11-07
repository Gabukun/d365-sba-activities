# 01 – Functional design

**Status:** Draft v0.1 – 07.11.2026. Open points: [doc 08](08-decisions-and-open-points.md).

## 1. Background

- Honegger bills recurring charges with **Subscription billing → Recurring contract billing** (billing schedules).
- With the standard feature **Billing schedules with projects**, a billing schedule header can carry a project. Its lines then show a **Project** tab with *Project ID, Category, Line property, Sales tax*. **There is no activity field** ([Billing schedules with projects](https://learn.microsoft.com/dynamics365/finance/accounts-receivable/sb-bill-sched-project)).
  - When such a line is invoiced with **Generate invoice**, a **sales order** is created, and it is invoiced through a **project invoice proposal** (same source).
- The sales order line has a standard **Activity number** field, `SalesLine.ActivityNumber`, EDT `ProjActivityNumber` ([CDM: SalesLine](https://learn.microsoft.com/common-data-model/schema/core/operationscommon/tables/supplychain/salesandmarketing/worksheetline/salesline)). Sales orders generated from billing schedules leave it empty, because the source has no activity.
- **At Honegger every billing schedule is linked to a project** (confirmed 07.11.2026). Example: billing schedule 100-000098, line project 100-000522.1.1. The sales order line for that project shows an empty *Aktivitätsnummer* (form `SalesTable`, control `Project_ActivityNumber`).

## 2. Goal

The user enters the **project activity** on the billing schedule line. *Generate invoice* copies it to the **Activity number** of the sales order line. From there, standard carries it into the project invoice and the posted project transactions.

> The last step must be proven before we build (discovery **D-07**). If standard doesn't carry `SalesLine.ActivityNumber` into the posted project transactions, the business value depends on what Honegger reports from (OP-01).

## 3. Scope

**In scope (proposal)**

| ID | Requirement |
|---|---|
| FR-01 | New field **Activity number** on the billing schedule line, on the **Project** tab next to *Project ID* |
| FR-02 | Lookup shows the activities of the line's project. It behaves like the Activity number lookup on the sales order line. |
| FR-03 | Validation: the activity must belong to the line's project. The field can only be filled when the line has a project. |
| FR-04 | Changing or clearing the line's project clears the activity |
| FR-05 | *Generate invoice* (from the schedule and as the batch job) copies the activity to `SalesLine.ActivityNumber` |
| FR-06 | Consolidation never merges billing schedule lines with **different** activities into one sales order line |
| FR-07 | The billing schedule line data entity exports and imports the activity, with the same validation as FR-03 |
| FR-08 | *Copy schedule* copies the activity together with the line |
| FR-09 | Lines without an activity behave exactly as today (no error, no change) |
| FR-10 | Works with Honegger's feature states. Preview features that touch sales order creation are tested on and off (doc 07). |
| FR-11 | **Renewal** (*Renew automatically*, *Add renewal term*, *Lines to add per renewal*): new or extended lines keep the activity of the line they come from |
| FR-12 | **Support and renewal** (posting a sales order invoice creates or extends a billing schedule): the new billing schedule line takes `SalesLine.ActivityNumber` if the project matches. VERIFY with Honegger that "renewals: in" includes this direction. |
| FR-13 | **Termination credits** (*Issue credit*, *Credit adjustment*): every sales order or credit line created for the terminated line carries its activity (path: D-09) |

Scope decisions of 07.11.2026 (DEC-09): renewals **in**, copy schedule **in**, termination credits **in**, project fee journal invoice type **out**.

**Still open (OP-05, OP-06):**
- Revenue split child lines: inherit the parent's activity?
- Default from the header? (The header has no activity today; proposal: no.)
- ~~Backfill of existing lines~~ **Not needed** (07.11.2026): Honegger is setting up Subscription billing now and has no productive billing schedules yet. Condition: the extension must be in production **before** Honegger creates productive schedules, otherwise those lines need the activity filled afterwards (OP-17).

**Out of scope:**
- The **Project fee journal** invoice type (GA September 2026; it creates project fee journals instead of sales orders).
- Deferral schedules, multiple element revenue allocation (MERA), revenue recognition journals.
- Billing schedules without a project. They don't occur at Honegger. The field stays empty and nothing breaks (FR-09).

## 4. Process (to-be)

```
Project (T&M, activities in the WBS)
  └─ New > Subscription billing > Billing schedule          (standard)
       └─ Line: item, quantity, period
            └─ Project tab: Project ID (default), Activity number  ← NEW (FR-01..04)
                 └─ Generate invoice → Create sales order            (standard)
                      └─ Sales order line: Activity number filled    ← NEW (FR-05, FR-06)
                           └─ Project invoice proposal → post        (standard, D-07)
```

## 5. Roles and security

- No new roles or privileges. The field belongs to the existing billing schedule line, so whoever may edit the line may edit the activity.
- The data entity extension inherits the entity's privileges. VERIFY that the best-practice check agrees.
