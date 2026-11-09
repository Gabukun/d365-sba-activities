# 08 – Decisions and open points

## 1. Decision log

| ID | Date | Decision | Reason | Status |
|---|---|---|---|---|
| DEC-01 | 07.11.2026 | Field `HOGsbaActivityNumber` on the billing schedule line, EDT `ProjActivityNumber` (as `SalesLine.ActivityNumber`), standard label *Activity number / Aktivitätsnummer*. Control `SubBillScheduleLine_HOGsbaActivityNumber`. | As close to standard as possible. Microsoft requires the prefix at the beginning of fields added in extensions ([C-01 §2](conventions/d365-finops/01-naming-and-extensions.md#2-prefixes)). `SubBillScheduleLine_ActivityNumber` and `Project_ActivityNumber` are control names, not field names. | **Agreed** |
| DEC-02 | 07.11.2026 | Own package `HOGsbaSubBillActivity`, solution prefix `HOGsba` (SBA = official acronym of the module), test package `HOGsbaSubBillActivityTest` | Can be deployed and removed independently | **Agreed** |
| DEC-03 | 07.11.2026 | Activity only on lines that have a project, validated against that project with the standard check | `SalesLine.ActivityNumber` belongs to the project of the SO line | Proposed |
| DEC-04 | 07.11.2026 | Activity is part of the consolidation key | Otherwise one SO line could represent two activities | Proposed (after D-05) |
| DEC-05 | 07.11.2026 | Copy in the narrowest hook before `SalesLine` insert. A mismatch never stops invoice generation. | Robust against Microsoft refactoring; no batch failures | Proposed (after D-03) |
| DEC-06 | 07.11.2026 | GitHub until handover; build and test on the VM; Honegger's Azure DevOps afterwards | [ADR-001](adr/ADR-001-github-now-azure-devops-at-handover.md) | Proposed |
| DEC-07 | 07.11.2026 | The VM mirrors hog-uat feature states. Preview features stay off unless hog-uat has them on. | Tests must run the customer's code paths | Proposed |
| DEC-08 | 07.11.2026 | No RSAT. SysTest plus Markdown test cases; CSV import into Azure DevOps Test Plans on request. | RSAT unsupported after 15.05.2027; Task recorder no longer works on cloud-hosted dev boxes | Proposed |
| DEC-09 | 07.11.2026 | Scope: renewals **in**, copy schedule **in**, termination credits **in**, project fee journal invoice type **out** | Answer from Gabriel | **Agreed** |
| DEC-10 | 07.11.2026 | Honegger's Azure DevOps uses **Git**. Handover = *Import repository* with full history. | Answer from Gabriel | **Agreed** |
| DEC-11 | 07.11.2026 | Reusable conventions are mastered in the central repository [youniform](https://github.com/youserpath/youniform) (`conventions/`). This repo keeps a **snapshot** in `docs/conventions/` so it stays complete when handed over to Honegger's Azure DevOps. | Answer from Gabriel | **Agreed** |
| DEC-12 | 07.11.2026 | No backfill: Subscription billing goes live at Honegger without existing schedules | Answer from Gabriel | **Agreed** (see OP-17) |
| DEC-13 | 07.11.2026 | Discovery and builds run in the Claude session **on the dev VM**. The cloud session does research, docs and review. | Claude installed on the VM | **Agreed** |
| DEC-14 | 09.11.2026 | **Closed activities are not validated at invoice time**, as in standard. The activity is validated when it is entered on the billing schedule line (FR-03). *Generate invoice* copies it without checking again whether the activity has been closed since. | Same as standard: generate invoice creates the sales order line with validation off, so standard doesn't check the activity there either (`SalesLine.checkActivity` runs only in `validateWrite`). | **Agreed** |
| DEC-15 | 09.11.2026 | **Consolidation merges only lines with the same item and the same activity** (FR-06). Lines that differ only by activity become separate sales order lines. Answers OP-20 in part. | One sales order line must never stand for two activities (DEC-04, R-04). | **Agreed** |

## 2. Open points

| ID | Topic | Question | Owner | Needed before |
|---|---|---|---|---|
| OP-01 | Purpose | What is the activity on the SO line used for: project reporting per activity, revenue per WBS element, dimensions, Power BI? Which report proves success? Volumes (schedules, lines per run)? | Honegger | Design sign-off |
| OP-02 | Usage | ~~Does Honegger use Billing schedules with projects?~~ **Closed 07.11.2026: yes, strictly; every billing schedule has a project.** | – | – |
| OP-03 | Versions | Honegger: 10.0.49 (10.0.2790.83), PU73 (see OP-16 for which environment). Open: production version, and the VM's exact build (must be the same or lower). | Gabriel | Build |
| OP-04 | Azure DevOps | Git (DEC-10). Still open: is there a build pipeline for `HGExtension`? Who builds, packages and deploys? LCS or PPAC? | Honegger IT / AlfaPeople | Handover |
| OP-05 | Scope | Decided in DEC-09. Still open: revenue split child lines (inherit the parent's activity?), default from the header (proposal: no), and whether "renewals" includes support and renewal from a sales order (FR-12). | Honegger | Build |
| OP-06 | Backfill | ~~Fill existing lines?~~ **Closed: no existing schedules (DEC-12).** | – | – |
| OP-07 | Naming | **Closed:** DEC-01, DEC-02 agreed. | – | – |
| OP-08 | Conventions | ~~Where do the conventions live?~~ **Closed: youniform (DEC-11).** Open: the prefix casing rule (`HOGsba` vs `HOGSoc` in d365-pma-indirectcosts) is decided there. | Gabriel | youniform |
| OP-09 | GitHub | GitHub plan of the account: protected branches on private repos need Pro/Team | Gabriel | First PR |
| OP-10 | MCP | ERP MCP on **hog-uat** (not our VM): version ≥ 10.0.47 and Tier 2; Entra app registered in **Honegger's** tenant; client ID under *Allowed MCP clients*; dedicated read-only user; written approval for data access by Claude. Claude's connector sign-in with this server is undocumented, so test it first. | Honegger IT / Gabriel | Optional |
| OP-11 | Test data | Copy of the hog-uat database on the VM, or a hand-made test company per doc 04 §4? | Gabriel / Honegger | Testing |
| OP-12 | Acceptance | Who tests in hog-uat and signs off? | Honegger | UAT |
| OP-13 | Alternative | *Financial tags for Subscription billing* (preview in 10.0.50) carries extra context from billing schedule lines downstream. It is not a replacement: it doesn't fill `SalesLine.ActivityNumber` (inference). Revisit if OP-01 is about reporting only. | Gabriel | Design sign-off |
| OP-14 | Claude on the VM | **Closed: installed (DEC-13).** Data access still depends on OP-16. | – | – |
| OP-15 | Customer prefix | Honegger's existing model is `HGExtension`. Do its objects use `HG…`? If so, should new packages use `HG` instead of `HOG`? (D-12) | Gabriel / Honegger | Build |
| OP-17 | Timing | When does Honegger create the first **productive** billing schedules? The extension must be in production before that, or those lines need the activity filled afterwards. While Honegger is still setting up the module, feature states and parameters can change: mirror them again before testing (doc 04 P-03). | Honegger / Gabriel | Plan |
| OP-18 | Scope | FR-12 (support and renewal from a posted SO invoice): standard creates the billing schedule header and line **without a project** (`SubBillScheduleTable::create`, `SubBillScheduleLine::createRenewalLine`, internal; D-13). There is nothing to match the SO line's activity against. Drop FR-12, or also copy `ProjId` from the SO line (a new feature, and `createRenewalLine` is internal, so only a hook on `SubBillScheduleLine.insert` or later would work)? | Honegger / Gabriel | Build |
| OP-19 | Scope | **Customer split**: split lines (`SubBillCustomerSplitLine`) carry their **own** project, and generate invoice takes the SO line project from the split line (D-09). Should the activity apply to split lines? If the split line's project equals the schedule line's, copy the activity; otherwise leave it empty? Does Honegger use customer split at all? | Honegger | Build |
| OP-20 | Consolidation | Does Honegger use **Consolidate by item** or consolidation by billing code in *Generate invoice*? If not, R-10 doesn't apply to them, and FR-06 can be limited to a guard. | Honegger | Build |
| OP-16 | Environment | Which environment are the screenshots of 07.11.2026 from (hog-uat or the VM)? If the VM holds a copy of Honegger's data, Honegger must agree that Claude on the VM may read it, or we use a data-free test company. | Gabriel | Discovery |

## 3. Discovery items (dev VM)

D-01 to D-13, see [doc 02 §4](02-technical-design.md#4-discovery-on-the-dev-vm-before-coding). Status:

Static discovery on the VM, 09.11.2026 (Claude on the VM, read-only on `PackagesLocalDirectory`, no database). Details in doc 02 §1 and §4.1. "Static" means read in the source; the runtime path still needs one debugger run.

| ID | Status | Result |
|---|---|---|
| D-01 | **done** 09.11.2026 | Package/model `SubscriptionBilling` (VM 10.40.3708.49257). `SubBillScheduleLine`: key `SubBillBillingScheduleNumber` + `LineNum`; project fields `ProjId`, `ProjCategoryId`, `ProjLinePropertyId`, `ProjTaxGroup`, `ProjTaxItemGroup` (feature class `SubBillProjectEnableFeature`). **No project field group** (only `AutoReport`). CSV: `docs/discovery/D-01-subbill-objects.csv` (753 objects). |
| D-02 | **done** | Form `SubBillScheduleTable`, data source `SubBillScheduleLine`, field `ProjId`; tab page `TabProject`, group `Project` (09.11.2026) |
| D-03 | static done; debugger open | `SubBillCreateSalesOrder::createSalesLine` (public static, class public final) fills `SalesLine` from `SubBillSalesLineConsolidated`, **before** `createSalesOrder` inserts it with `salesLine.createLine(false, …)`. Community signature confirmed. The activity travels via 3 more public static hooks (query selection, group-by, `initializeSubBillSalesLineConsolidated`). |
| D-04 | **done** 09.11.2026 | Check: `ProjActivityHelper::validateProjActivityParentLinkV2` (via `SalesLine.checkActivity` ← `validateWrite_server`; no `validateField` case). Lookup: `HierarchyTreeTable::lookupHierarchyTree(ctrl, ProjTable)`. Side effect: `modifyField` → `initFromProjActivity` may replace `ProjCategoryId` → we assign without `modifiedField`. |
| D-05 | static done; debugger open | Grouping in the consolidation query (`addScheduleLineGroupByFields`). Link-back after insert compares project fields in private/unchangeable joins → R-10, OP-20, TC-06. |
| D-06 | **done** 09.11.2026 | Form `SubBillCopySchedule.copyScheduleLine` uses `buf2Buf`; `ProjId` kept. No hook needed (FR-08). |
| D-07 | **Closed** 09.11.2026 | Stated by Gabriel from earlier tests; **not rerun** on the VM. A manually entered `SalesLine.ActivityNumber` reaches the posted project transactions. |
| D-08 | open | Entity `SubBillScheduleLineEntity` exists (D-01). Staging table still to check. |
| D-09 | static done; debugger open | Termination (*Terminate* dialog and mass termination) → `SubBillMassTermination` (3 public static hooks), then the D-03 methods; one SO line per schedule line. *Credit adjustment* is billed through D-03. Customer split takes the project from `SubBillCustomerSplitLine` → OP-19. |
| D-10 | **done** 09.11.2026 | `ProjActivityNumber` (ApplicationSuite, config key `Project`) → `smmActivityNumber` → `smmActivityId` (CaseManagement, size 50, label `@SYS81290`); relation to `smmActivities.ActivityNumber`. |
| D-11 | static done | Feature class `SubBillQueryPerfImprovementFeature`; not used on our hook path (only after insert). Runtime check only if hog-uat turns it on. |
| D-12 | **blocked** 09.11.2026 | No package matches `^(HG|Axtension|AXT)`; the VM has no non-Microsoft model. Honegger's models are not on the VM (P-09). |
| D-13 | static done; debugger open | Auto-renewal extends the same line; milestone auto-renewal uses `buf2Buf`; renewal terms are `SubBillRenewalLine` → activity kept, no hook. Support and renewal creates lines without a project → OP-18. Revenue split children don't inherit it → OP-05. |

## 4. Risks

| ID | Risk | Impact | Mitigation |
|---|---|---|---|
| R-01 | Standard names differ from the community sources | Rework of hooks | Discovery D-01 to D-06 before coding |
| R-02 | Microsoft adds an activity to billing schedule lines later | Duplicate field | Prefixed name, so no compile conflict; migrate data to the standard field then |
| R-03 | The preview performance feature changes the SO creation path | Activity silently missing | D-11, TC-16; narrow hook |
| R-04 | Consolidation merges lines with different activities | Wrong activity on revenue | DEC-04, TC-06 |
| R-05 | Setting `ActivityNumber` by code skips standard side effects | Inconsistent SO line (e.g. dimensions) | D-04, D-07 |
| R-06 | VM version newer than Honegger's | Package doesn't deploy, or behaves differently | OP-03 |
| R-07 | The project fee journal invoice type is used | Extension never runs | OP-05. **Closed 09.11.2026:** at go-live Honegger generates **sales order lines only**, because accruals run on sales order lines. Condition: check `SubBillInvTransType` on all productive billing schedules. |
| R-08 | Customer rejects AI-assisted code or data access | No deployment / no MCP | Code reviewed and owned by AlfaPeople; full test evidence; MCP read-only with written approval |
| R-09 | `HGExtension` or an ISV model already wraps the same methods, or already adds an activity | Conflicting CoC behaviour or a duplicate field | D-12 blocked, covered by TC-21 in hog-uat before handover. **09.11.2026:** D-12 can't run on the VM (models missing, P-09). **Note 09.11.2026:** the Subscription billing module is believed to be unextended at Honegger. Evidence limit: nothing extending it has been seen since the module was activated; access to Honegger's repository is still open, so this is not verified. |
| R-10 | With *Consolidate by item*, standard links billed detail lines to the new SO line with joins on item and project fields (`insertBillingConsolidationDetails_ConsolidationByItem` private, `updateBillingDetails_ConsolidationByItem` public static, both with `insert_recordset`/`update_recordset`). We can add the activity to the grouping, but not to these joins. | Two SO lines that differ only by activity may be linked to each other's detail lines: wrong billed status or amounts | Debugger/test before relying on it (D-05, TC-06). Ask OP-20. Fallback: don't group by activity; instead stop *Generate invoice* for a merge group with mixed activities, or use one SO line per schedule line when activities are set. |
| R-11 | Code read on the VM build (10.40.3708.49257) differs from hog-uat (10.40.3825.34730) | A hook signature or the path changed | Recheck the 7 hook methods after every VM update; build against the target version before handover (OP-03). |
