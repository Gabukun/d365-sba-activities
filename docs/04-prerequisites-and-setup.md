# 04 – Prerequisites and setup

Everything that must be true on the dev VM before discovery (doc 02 §4) and before testing (doc 07). Fill in the **Status** column as you go.

## 1. Checklist

| # | Item | Expected | Status |
|---|---|---|---|
| P-01 | VM version (*Help > About*: product, platform, *Installed models*) | Same as P-02, or lower | **Lower, OK** (09.11.2026, from the model descriptors in `PackagesLocalDirectory`): application models 10.40.3708.49257 (e.g. `SubscriptionBilling`, `ProjectOperations`), `ApplicationPlatform` 7.0.8199.32. hog-uat: 10.40.3825.34730 / 7.0.8199.55. Product version in *Help > About* not checked. See R-11. |
| P-02 | Honegger version | Screenshot 07.11.2026 (environment to confirm, OP-16): product **10.0.49 (10.0.2790.83)**, platform **Update73 (7.0.8199.55)**, application models 10.40.3825.34730. Rule: build on the **same or a lower** version than the target ([C-03 §3](conventions/d365-finops/03-alm-and-source-control.md#3-build-and-package)). Production must be confirmed separately. | Partly |
| P-09 | Honegger's own and ISV models on the VM: `HGExtension` (usr) and the *Axtension®* models (ISV; may need license files) | Present and building, so our code is tested against the same stack as hog-uat (D-12) | **Not present** (09.11.2026): no package matching `^(HG|Axtension|AXT)` and no non-Microsoft model on the VM. D-12 blocked. |
| P-03 | Feature states mirrored from hog-uat | Done through the *Feature management* entity (section 3) | **Done 09.11.2026.** 35 of 40 features enabled. 5 rejected on import: `BudgetControlTrackBFAOnlyFeature`, `VendInvoiceJourImportedTaxAmountFeature`, `TaxCalculateGSTBasedOnInvoiceAccountFeature`, `RetailReturnsTaxCalculationEnableFeature`, `ITMCostTypeCreditToLedgerFeature`. Judging by the names, none of them is a Subscription billing or project feature. |
| P-04 | Subscription billing features enabled | Section 2 | **Done** (09.11.2026) |
| P-05 | Test setup for Subscription billing with projects | Section 4 | |
| P-06 | Git for Windows, repo cloned, packages linked | Doc 06 §2 | |
| P-07 | Model `HOGsbaSubBillActivity` builds, BP checks on | Doc 06 §3 | |
| P-08 | Test packages present: `TestEssentials`, ATL | Folders exist in the packages folder (VERIFY exact names) | |

## 2. Features to enable (Feature management)

Microsoft: Subscription billing is enabled **in Feature management**, and it can't coexist with **Revenue recognition** ([Subscription billing overview](https://learn.microsoft.com/dynamics365/finance/accounts-receivable/subscription-billing-summary)). Microsoft documents no license configuration key for it, so maintenance mode should not be needed. VERIFY once in *System administration > Setup > License configuration*.

In this order:

| # | Feature | Action | Why |
|---|---|---|---|
| 1 | **Revenue recognition** | **Disable** | Mandatory before Subscription billing |
| 2 | **Subscription billing** | Enable | Base feature |
| 3 | **Recurring contract billing** | Enable | Billing schedules and *Generate invoice* |
| 4 | **Billing schedules with projects** (the Project Operations docs call it *Billing schedules and deferrals with projects*) | Enable, then **refresh the browser** | Project tab on the line, our FR-01 ([doc](https://learn.microsoft.com/dynamics365/finance/accounts-receivable/sb-bill-sched-project)) |
| 5 | Revenue and expense deferrals, Multiple element revenue allocation | **Only if hog-uat has them on** | Mirror the customer |
| 6 | Preview features (see below) | **Only if hog-uat has them on** | Microsoft: preview features "aren't meant for production use" ([preview features](https://learn.microsoft.com/power-platform/admin/what-are-preview-features-how-do-i-enable-them)) |

Preview features on the VM that concern Subscription billing (seen in the VM's list on 07.11.2026):

| Feature | What it does | Relevance |
|---|---|---|
| (Preview) Database Query Optimization and Reduced Chattiness | No Microsoft doc with this exact name. Probably related to the 2026 wave 1 item "Performance improvements when generating sales order invoice with subscription billing" ([release plan](https://learn.microsoft.com/dynamics365/release-plan/2026wave1/enterprise-resource-planning/dynamics365-finance/enhancements-subscription-billing)). | **May change the sales order creation path (our hook).** Run D-11 and TC-16 if hog-uat has it on. |
| (Preview) Enable project fee journals to support subscription billing for Project Operations for manufacturing | New invoice transaction type *Project fee journal*: *Generate invoice* creates project fee journals **instead of sales orders** ([doc](https://learn.microsoft.com/dynamics365/project-operations/prod-pma/subscription-bill-projects)). GA planned for September 2026. | That path bypasses `SalesLine`. Scope question OP-05. **On in hog-uat** (09.11.2026); the invoice transaction type in use is still to be checked (R-07). |
| (Preview) MERA termination revenue adjustment | New in 10.0.49. Affects complete termination from the header: MERA clearing journal or credit-note voucher ([doc](https://learn.microsoft.com/dynamics365/finance/accounts-receivable/multiple-element-revenue-allocation)). | Not on our path, unless termination credits create sales order lines (D-09) |

## 3. Mirror the customer's feature states

Microsoft provides the **Feature management** data entity to export feature states from one environment and import them into another. On import it applies the workspace rules, for example that mandatory features can't be disabled ([Feature management data entity](https://learn.microsoft.com/training/modules/feature-management-finance-operations/12-data-entities)).

1. hog-uat: *Data management > Export*, entity **Feature management**, format CSV or Excel → download.
2. VM: same export.
3. Put both files in `docs/discovery/feature-states/` (feature names and states only; no business data). Claude compares them.
4. Import the agreed rows into the VM with the same entity.

Features that use a configuration key appear in the list only after that key is on, followed by *Check for updates* ([features that use configuration keys](https://learn.microsoft.com/training/modules/feature-management-finance-operations/11-configuration-keys)).

The ERP MCP server can't read Feature management: the workspace form is explicitly excluded ([MCP known limitations](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/copilot/copilot-mcp#known-limitations)).

## 4. Test setup on the VM

Prefer Honegger's real setup (exported from hog-uat via DMF) over invented values (OP-11).

| # | Setup | Where | Source |
|---|---|---|---|
| 1 | Billing schedule group | Subscription billing > Recurring contract billing > Setup > Billing schedule groups | [Parameters](https://learn.microsoft.com/dynamics365/finance/accounts-receivable/sb-recur-bill-parameters) |
| 2 | Recurring contract billing parameters: default group, invoice transaction type **Sales order**, posting option, consolidation defaults, billing schedule number sequence | Recurring contract billing parameters | same |
| 3 | Project stage rule **Create billing schedule = Yes** for the stage used in tests | Project management and accounting parameters > Project stage | [Billing schedules with projects](https://learn.microsoft.com/dynamics365/finance/accounts-receivable/sb-bill-sched-project) |
| 4 | Time-and-material project with contract and customer funding source, in an invoiceable stage, **with at least two WBS activities** | Projects | same |
| 5 | Second project with its own activity (for TC-02) | Projects | – |
| 6 | Service or non-stock item (project lines allow only these) | Released products | same |
| 7 | Customer with a bill-to address (otherwise *Generate invoice* shows no lines) | Customers | same |
