# 02 – Technical design

> **Status:** draft. Static discovery on the dev VM done 09.11.2026 (D-01, and the source-reading part of D-03 to D-06, D-09 to D-11, D-13; see §4.1). Names in §1 marked **Confirmed** were read in `PackagesLocalDirectory` on the VM. Names still marked **VERIFY** are not yet confirmed. X++ in this document is a **draft and has not been compiled**.
>
> The VM runs application **10.40.3708.49257** (platform 7.0.8199.32). hog-uat runs 10.40.3825.34730 (doc 04 P-02). The code below was read on the **older** VM build. Re-check the hook methods when the VM is updated (R-11).

## 1. Standard objects – what we know

Source for every row marked *VM*: metadata in `C:\AosService\PackagesLocalDirectory`, read 09.11.2026. Full object list: [`discovery/D-01-subbill-objects.csv`](discovery/D-01-subbill-objects.csv) (753 `SubBill*` tables, classes, forms and entities; names only).

| Concept | Name | Source | Status |
|---|---|---|---|
| Subscription billing package/model | Package and model **`SubscriptionBilling`** (folder `SubscriptionBilling\SubscriptionBilling`), display name *Subscription Billing*, references `ApplicationSuite`, `ApplicationFoundation`, `ApplicationPlatform`, `Project`, `Dimensions`, … | VM: `SubscriptionBilling\Descriptor\SubscriptionBilling.xml` | **Confirmed (D-01)** |
| Other packages with `SubBill*` objects | `ProjectOperations` (5 CoC classes, e.g. `SubBillScheduleLine_ProjOps_Extension`), `BusinessIntelligence` (9 BI entities) | VM, D-01 CSV | Confirmed |
| Billing schedule header | Table `SubBillScheduleTable` | VM | Confirmed |
| Billing schedule line | Table **`SubBillScheduleLine`**, 78 fields. Primary, clustered and replacement key: index `LineNum` = `SubBillBillingScheduleNumber` + `LineNum` | VM: `AxTable\SubBillScheduleLine.xml` | **Confirmed (D-01)** |
| Project fields on the line | `ProjId` (EDT `ProjId`), `ProjCategoryId` (EDT `ProjCategoryId`), `ProjLinePropertyId` (EDT `ProjLinePropertyId`), `ProjTaxGroup` (EDT `TaxGroup`), `ProjTaxItemGroup` (EDT `TaxItemGroup`). All five: `FeatureClass` = `SubBillProjectEnableFeature`, `IgnoreEDTRelation` = Yes, explicit table relations `ProjTable`, `ProjCategory`, `ProjLineProperty`, `ProjTaxGroup`, `ProjTaxItemGroup`. | VM | **Confirmed (D-01)** |
| Field groups of the line | **No project field group.** Groups: `AutoReport` (contains the project fields), `AutoLookup`, `AutoIdentification`, `AutoSummary`, `AutoBrowse`, `SubBillRetailVariantId`. | VM | **Confirmed (D-01)** |
| Project tab on the form | Form `SubBillScheduleTable` → tab `TabLineDetails` → tab page **`TabProject`** → group **`Project`** with controls `SubBillScheduleLine_ProjId`, `SubBillScheduleLine_ProjCategoryId`, `SubBillScheduleLine_ProjLinePropertyId` | VM: `AxForm\SubBillScheduleTable.xml` | **Confirmed (D-02 remainder)** |
| Line: reaction to a project change | `SubBillScheduleLine.modifiedFieldValue(FieldName, int = 1)`, public, case `ProjId`: `initProjFields` (internal), `mergeProjectDefaultDimension`, `rolldownProjFields` (private; revenue split parents and milestone headers). `modifiedField` has no `ProjId` case. Form data field `SubBillScheduleLine.ProjId` → `modified()` and `lookup()` are `[Hookable(false)]`. | VM | Confirmed |
| Line: project validation | `SubBillScheduleLine.validateField`, case `ProjId` → `SubBillScheduleTable::validateProjectId(ProjId, CustAccount)` | VM | Confirmed |
| Line: other CoC on the same table | `SubBillScheduleLine_ProjOps_Extension` (package `ProjectOperations`) wraps `insert`, `validateDelete`, `lastProtectedDetailLineForRecreation`, `isDetailLineUpdatableDuringRecreation`. Only active with Project Operations integration. | VM | Confirmed; no conflict with our hooks |
| Header project change | The header `ProjId` is **not** pushed to existing lines. Only form `SubBillScheduleTable`, data field `ItemId.modified()` copies the header project to the line. | VM | Confirmed |
| Generate invoice working table | `SubBillInvoiceParmLines` | [What's new 10.0.47](https://learn.microsoft.com/dynamics365/finance/get-started/whats-new-changed-10-0-47) | Confirmed name |
| Generate invoice service | `SubBillGenerateInvoice` (public, extends `SysOperationServiceBase`): `runProcess`, `generateSalesOrderOrFreeText` (protected), `consolidateInvoiceProcessingData` (private) | VM | Confirmed |
| Sales order creation | Class **`SubBillCreateSalesOrder`**: `public final class`, **all methods static**. Method list and signatures in §4.1. | VM | **Confirmed (D-03 static)** |
| Consolidation buffer | Table **`SubBillSalesLineConsolidated`**: regular table (group *Transaction*), 95 fields, has `ProjId`, `ProjCategoryId`, `ProjLinePropertyId`, `ProjTaxGroup`, `ProjTaxItemGroup`, `ScheduleLineRecId`, `LineNum`, `SubBillBillingScheduleNumber`, `ParmId`. No activity field. | VM | **Confirmed (D-05)** |
| Termination credits | Class `SubBillMassTermination` (public final, static). Its own `addScheduleLineSelectionFields`, `addScheduleLineGroupByFields`, `initializeSubBillSalesLineConsolidated`, then the same `SubBillCreateSalesOrder` methods. | VM | Confirmed (D-09 static) |
| Customer split | Class `SubBillCreateSalesOrderCustomerSplit` (public final, static). Project comes from **`SubBillCustomerSplitLine.ProjId`**, not from the schedule line (`assignSalesLineConsolidatedValues`, protected static). | VM | Confirmed (D-09 static); design open (OP-19) |
| Back-references on SalesLine | `SubBillDetailLineRecId`, `SubBillTermScheduleLineRecId`, … | Community SQL schema export | VERIFY (not needed so far) |
| SO line activity | `SalesLine.ActivityNumber`, EDT `ProjActivityNumber`, `IgnoreEDTRelation` = Yes, table relation `smmActivities` (`ActivityNumber` → `smmActivities.ActivityNumber`, OnDelete *Restricted*, index `ActivityNumberIdx`) | VM: `ApplicationSuite\Foundation\AxTable\SalesLine.xml` | **Confirmed (D-10)** |
| EDT chain | `ProjActivityNumber` (package `ApplicationSuite`, configuration key **`Project`**, no own relation) extends `smmActivityNumber` (package `CaseManagement`, FormHelp `smmActivityIdLookup`, relation `smmActivities.ActivityNumber`) extends `smmActivityId` (`CaseManagement`, label `@SYS81290` *Activity number* / *Aktivitätsnummer*, **string size 50**). No EDT extensions on the VM. | VM | **Confirmed (D-10)** |
| Activity master | `smmActivities` (package `CaseManagement`); WBS activity `ProjActivity` (`ApplicationSuite`) | VM | **Confirmed (D-10)** |
| Standard activity check | `ProjActivityHelper::validateProjActivityParentLinkV2(ProjId _projId, smmActivityNumber _activityNumber, boolean _enableClosedCheck = true)`: public static, class not final (`ApplicationSuite`). Checks: `ProjActivity` exists; not closed (if `ProjParameters.EnableActivityIdCheck`); linked to the project in `smmActivityParentLinkTable`. With an empty project it fails with `@Proj:ActivityNumberNotAssociatedToProject`. | VM | **Confirmed (D-04)** |
| Where SalesLine checks the activity | `SalesLine.checkActivity()` (public) → the check above. Called only in `SalesLine.validateWrite_server`. `SalesLine.validateField` has **no** `ActivityNumber` case. `SalesLine.checkReqActivity()` uses `ProjTable::checkRequireActivity`. | VM | **Confirmed (D-04)** |
| Side effects on SalesLine | `SalesLine.modifiedField` → `modifyField(FieldId, boolean = true, boolean = true)`, case `ActivityNumber` → `initFromProjActivity(ProjActivity)`: sets `ProjCategoryId` to the activity's default category if that category is of type *Item*. `taxRecalculationNeeded` compares `ActivityNumber`. | VM | **Confirmed (D-04)** |
| Standard lookup | Form `SalesTable`, data source `SalesLine`, data field `ActivityNumber` → `lookup()` → `HierarchyTreeTable::lookupHierarchyTree(FormStringControl _ctrl, Common _common, NoYes _showChildOnly = NoYes::No)` with `ProjTable::find(salesLine.ProjId)`. `HierarchyTreeTable` is in package `CaseManagement`. Control `Project_ActivityNumber` has no own methods. | VM | **Confirmed (D-04)** |
| Billing schedule line entity | `SubBillScheduleLineEntity` exists (D-01 CSV); it calls `clearProjFields`. Public name `SubBillBillingScheduleLine` (community), staging table not checked. | VM, community | VERIFY D-08 |
| Preview performance feature | `SubBillQueryPerfImprovementFeature` = *(Preview) Database Query Optimization and Reduced Chattiness* (label `@SubBill:SubBillQueryPerfImprovementLabel`). In `SubBillCreateSalesOrder` it is used only in `updateBillingDetails_NoConsolidation`, after the sales line is inserted. | VM | Confirmed (D-11 static) |
| Honegger extension models | `HGExtension` (usr, 2025.3.6.4); ISV *Axtension®* Platform / Invoice Processing / Content Gate. **Not on the VM**: no package matches `^(HG|Axtension|AXT)`, and the VM has no non-Microsoft model at all. | *About > Installed models* on hog-uat; VM | **D-12 blocked (P-09)** |

## 2. Objects to create

Package **`HOGsbaSubBillActivity`**, solution prefix **`HOGsba`** (DEC-02). Naming per [C-01](conventions/d365-finops/01-naming-and-extensions.md). Hook points updated after static discovery (§4.1); still to be compiled.

| Type | Name | Purpose | Depends on |
|---|---|---|---|
| Descriptor | `HOGsbaSubBillActivity` | Model, layer usr. References: `SubscriptionBilling`, `ApplicationSuite` (`ProjActivityNumber`, `ProjActivityHelper`, `SalesLine`), `CaseManagement` (`HierarchyTreeTable`, `smmActivities`), `ApplicationFoundation`, `ApplicationPlatform`. Final list after the first build. | D-01 done |
| Table ext. | `SubBillScheduleLine.HOGsba` | Field **`HOGsbaActivityNumber`** (EDT `ProjActivityNumber`, standard label). Add to `AutoReport` like the standard project fields (there is no project field group). Relation to `smmActivities` like `SalesLine`. `FeatureClass` `SubBillProjectEnableFeature` like the standard project fields, if the property is allowed on an extension field (VERIFY at build). | D-01 done |
| Table ext. | `SubBillSalesLineConsolidated.HOGsba` | Field `HOGsbaActivityNumber`. Carries the activity from the schedule line to the sales line, like standard carries `ProjId`. | D-05 done |
| Class (CoC) | `SubBillScheduleLine_HOGsba_Extension` | `validateField`, `validateWrite` (FR-03, also for imports), `modifiedField` (FR-04) | D-04 done |
| Form ext. | `SubBillScheduleTable.HOGsba` | Control `SubBillScheduleLine_HOGsbaActivityNumber` in group `Project` on tab page `TabProject`, after `SubBillScheduleLine_ProjId` | D-02 done |
| Lookup | form data field or control event handler, name TBD | Lookup (FR-02) via `HierarchyTreeTable::lookupHierarchyTree` with the line's project. The standard data field methods on this form are `[Hookable(false)]`, so the lookup goes on **our** field/control. Mechanism (data field CoC vs `OnLookup` handler): decide at build. | D-04 done |
| Class (CoC) | `SubBillCreateSalesOrder_HOGsba_Extension` | Wraps 4 public static methods: `addScheduleLineSelectionFields`, `addScheduleLineGroupByFields` (FR-06), `initializeSubBillSalesLineConsolidated`, `createSalesLine` (FR-05) | D-03, D-05 static done |
| Class (CoC) | `SubBillMassTermination_HOGsba_Extension` | Same three query/initialise hooks for termination credits (FR-13) | D-09 static done |
| Class (CoC) | customer split hook, name TBD | `SubBillCreateSalesOrderCustomerSplit::assignSalesLineConsolidatedValues`, only if OP-19 says the activity applies to split lines | OP-19 |
| Class | `HOGsbaSalesLineActivity` | Copy rule: consolidated buffer → `SalesLine.ActivityNumber` (design choices 4, 5) | D-04 done |
| Class (CoC) | copy-schedule hook | **Not needed**: *Copy schedule* uses `buf2Buf` (FR-08) | D-06 done |
| Class (CoC) | renewal hooks | **Not needed** for auto-renewal (same line is extended) and milestone auto-renewal (`buf2Buf`). Support and renewal: see OP-18. | D-13 static done |
| Entity ext. | `SubBillScheduleLineEntity.HOGsba` + staging table ext. | Field `HOGsbaActivityNumber` (FR-07) | D-08 |
| Test package | `HOGsbaSubBillActivityTest` | SysTest classes (doc 07 §3) | – |
| Labels | probably one | Validation reuses the standard `@Proj:` messages. The warning in design choice 5 needs a text; check for a fitting standard label first, otherwise create label file `HOGsba`. | – |

## 3. Design choices

1. **Field name `HOGsbaActivityNumber`.** This is the standard name `ActivityNumber` with the mandatory extension prefix ([C-01 §2](conventions/d365-finops/01-naming-and-extensions.md#2-prefixes)). Microsoft is actively extending Subscription billing towards projects (project fee journals, project deferrals). If it adds `ActivityNumber` to the line one day, an unprefixed field would break the build.
2. **Same EDT as the sales order line** (`ProjActivityNumber`). Length, label, help text and relation are then identical, and the copy is a plain assignment.
3. **Reuse the standard lookup and validation** that the sales order line uses for its Activity number (D-04). We don't write our own query over the WBS. This keeps behaviour identical (closed activities, project hierarchy) and survives service updates.
4. **Copy as late and as narrowly as possible.** Prefer a hook where the sales order line is initialised **before insert**. Avoid updating an already inserted line (an extra database round trip, and it re-runs `SalesLine.update` logic). Final choice after D-03.
5. **Copy only when the project matches.** Set `SalesLine.ActivityNumber` only if the sales order line has the same `ProjId` as the billing schedule line. Otherwise do nothing and log a warning. Never fail invoice generation because of the activity.
6. **Consolidation.** If *Consolidate by item* or *Consolidate all periods* can merge several billing schedule lines into one sales order line, the activity must be part of the comparison, just like Project already is ([sb-bill-sched-project](https://learn.microsoft.com/dynamics365/finance/accounts-receivable/sb-bill-sched-project): consolidation compares Project). D-05 shows where that comparison happens.
7. **Standard side effects.** If standard runs logic when `SalesLine.ActivityNumber` changes (for example defaulting dimensions), our copy must trigger it the same way standard does when it sets the field (D-04).

**After static discovery (09.11.2026):**
- To 4: `SubBillCreateSalesOrder::createSalesLine` fills the buffer and returns it; `createSalesOrder` inserts it afterwards with `salesLine.createLine(false, false, false, true, true, false)`. A post-`next` assignment in `createSalesLine` is **before insert**. Validation is off in that `createLine` call, so `SalesLine.checkActivity` does not run there.
- To 5: `createSalesLine` takes `ProjId` from `SubBillSalesLineConsolidated`, not from the schedule line. We carry the activity on the same buffer, so the comparison is consolidated `ProjId` vs `SalesLine.ProjId`.
- To 7: The only standard side effect is `initFromProjActivity`, which can **overwrite `ProjCategoryId`** with the activity's default category. On a billing-schedule-generated line the category comes from the billing schedule line and must stay. **Proposal: plain assignment, don't call `modifiedField`.** No dimension defaulting is tied to `ActivityNumber`.
- To 6: see R-10. Standard links billed detail lines back to the sales order line with `insert_recordset`/`update_recordset` joins that compare the project fields. With *Consolidate by item*, those joins are in a **private** method and a public method we can't change. If the activity becomes part of the grouping but not of the link-back join, two sales order lines that differ only by activity can be linked to the wrong detail lines.

## 4. Discovery (on the dev VM, before coding)

Send the result as text (call stacks, short source excerpts). **Don't paste whole Microsoft source files into the repo.**

| ID | Question | How |
|---|---|---|
| **D-01** | Package, tables and fields of Subscription billing | **Done 09.11.2026** (§1, §4.1). PowerShell below; CSV in `docs/discovery/`. |
| **D-02** | Billing schedule form, line data source, Project tab group | **Done:** form `SubBillScheduleTable`, control `SubBillScheduleLine_ProjId` (07.11.2026); tab page `TabProject`, group `Project` (09.11.2026). |
| **D-03** | Where is `SalesLine` created and inserted by *Generate invoice*? | Static part **done** (§4.1). Remaining: confirm with the debugger that this path runs (breakpoint in `SalesLine.insert()` from `w3wp.exe`, *Generate invoice* → *Create sales order*, one line). |
| **D-04** | Standard lookup, validation and side effects of `SalesLine.ActivityNumber` | **Done 09.11.2026** (§1). |
| **D-05** | Consolidation: where are lines grouped into one sales order line, and which fields are compared? | Static part **done** (§4.1). Remaining: debugger run with *Consolidate by item* = Yes, two lines of the same item and project with different activities (TC-06). Checks R-10. |
| **D-06** | How does *Copy schedule* copy lines? | **Done 09.11.2026:** `buf2Buf` (§4.1). |
| **D-07** | **Business value check, no code needed:** does a manually entered `SalesLine.ActivityNumber` reach the posted project transactions? | Billing schedule with project → *Generate invoice* (Create sales order) → set Activity number on the SO line by hand → project invoice proposal → post. Check the posted project transactions (and the voucher dimensions) for the activity. **Do this first.** |
| **D-08** | Billing schedule line data entity: AOT name and staging table | Entity `SubBillScheduleLineEntity` found (D-01). Remaining: staging table, public name. |
| **D-09** | Other paths that create sales order lines: customer split, termination *Issue credit* | Static part **done** (§4.1). Remaining: debugger for both, and OP-19. |
| **D-10** | `ProjActivityNumber` in D365: relation, string size, configuration key | **Done 09.11.2026** (§1). |
| **D-11** | Does the preview feature *Database Query Optimization and Reduced Chattiness* change the D-03 path? | Static part **done**: no (§1, §4.1). Remaining only if hog-uat has it on: repeat D-03 with the feature on. |
| **D-12** | Does Honegger's `HGExtension` (or an Axtension model) already extend `SubBillScheduleLine`, `SubBillCreateSalesOrder` or `SalesLine.ActivityNumber`? Which prefix do HGExtension objects use? | **Blocked 09.11.2026:** Honegger's models are not on the VM (P-09). Run the PowerShell below once they are, or on a machine that has them. |
| **D-13** | Renewal paths: *Renew automatically*, *Add renewal term*, *Lines to add per renewal*, and support and renewal from a posted SO invoice. Which methods create or copy the lines? | Static part **done** (§4.1). Remaining: debugger run for auto-renewal; OP-18. |

### 4.1 Static discovery results (09.11.2026)

Read on the VM in `PackagesLocalDirectory` (package `SubscriptionBilling` 10.40.3708.49257 unless stated). Class, method, signature and attributes are quoted from the metadata. **No method on the path below carries `[Hookable]`, `[Wrappable]` or `[Replaceable]`, and none is declared `final`.** Several *classes* are `final`; CoC on their public and protected methods is expected to work but is only proven by the first build.

**D-03 – Generate invoice (sales order), call chain**

| # | Class | Method (signature, shortened) | Visibility | What happens | Hook? |
|---|---|---|---|---|---|
| 1 | `SubBillGenerateInvoice` | `runProcess(SubBillGenerateInvoiceContract)` | public | Entry from the dialog and the batch job | no |
| 2 | `SubBillGenerateInvoice` | `consolidateInvoiceProcessingData(SubBillInvoiceParmUpdate, ParmId, SysInstrumentationActivityContext)` | private | Calls #3 for invoice type *Sales order*. Type *Project fee journal* goes to `ProjSubBillInvoiceConsolidationHelper` instead (out of scope). | no |
| 3 | `SubBillCreateSalesOrder` | `static void consolidateInvoiceProcessingData(boolean _customerConsolidate, …)` | public | Builds the query (#4), loops it, calls #6 per row, `RecordInsertList` into `SubBillSalesLineConsolidated`. Also calls `SubBillCreateSalesOrderCustomerSplit::addCustomerSplitSalesLineTmp`. | no |
| 4 | `SubBillCreateSalesOrder` | `static Query buildConsolidationQuery(…)` → `addScheduleLineSelectionFields(QueryBuildDataSource, SubBillInvTransType, boolean _itemConsolidate, boolean _consolidateByBillingCode)` and `addScheduleLineGroupByFields(` same parameters `)` | public static | Selection and group-by on `SubBillScheduleLine`. Project fields: selected if `SubBillProjectEnableFeature::isEnabled()`; grouped only if also `_invoiceTransType == SalesOrder`. `RecId` and `LineNum` are selected/grouped only **without** item or billing-code consolidation. | **yes** (add our field, same conditions) |
| 5 | `SubBillCreateSalesOrder` | `static SubBillSalesLineConsolidated initializeSubBillSalesLineConsolidated(SubBillSalesLineConsolidated, SubBillScheduleTable, SubBillScheduleLine _scheduleLineTable, SubBillScheduleLine _parentScheduleLine, SubBillDetailLine, SubBillSalesLine, boolean _itemConsolidate, boolean _consolidateByBillingCode, SubBillInvTransType, ParmId, boolean _doConsolidateAllPeriods, QueryRun)` | public static | Copies the project fields from `_scheduleLineTable` to the consolidated buffer (`if (SubBillProjectEnableFeature::isEnabled() && _scheduleLineTable.ProjId)`) | **yes** (copy our field) |
| 6 | `SubBillCreateSalesOrder` | `groupConsolidatedLinesBySalesId`, `generateSalesOrderFromConsolidatedData(…)` | public static | New SO header when, among others, `ProjId` changes (`createSalesHeaderCheck`). Calls #7 per consolidated line. | no |
| 7 | `SubBillCreateSalesOrder` | `static container createSalesOrder(boolean _createSalesHeader, SalesId, SubBillSalesLineConsolidated, …, ParmId, boolean _consolidateAllPeriods = false)` | public static | Header: `initSalesTable` + `salesTable.insert()`. Line: #8, then **`salesLine.createLine(false, false, false, true, true, false)` = insert**, then `updateBillingTable`. | no (too wide) |
| 8 | `SubBillCreateSalesOrder` | `static SalesLine createSalesLine(SalesLine _salesLine, SubBillSalesLineConsolidated _salesLineConsolidated, SalesId _salesIdCon, boolean _allocEnabled, boolean _splitByItemGroup, boolean _isInvoiceCreator, boolean _includeBillingDatesToItem)` | public static | **Initialises `SalesLine` before insert.** Sets `ProjId`, `ProjCategoryId`, `ProjLinePropertyId`, `TaxItemGroup`, `TaxGroup` from `_salesLineConsolidated` if the project feature is on and `ProjId` is set. Returns the buffer; no insert. | **yes** (FR-05) |
| 9 | `SubBillCreateSalesOrder` | `updateBillingTable(…)` → `updateBillingDetails_NoConsolidation` (private) / `updateBillingDetails_ConsolidationByItem` (public static) | mixed | After insert: links detail lines to the new SO line. By-item: `update_recordset`/`insert_recordset` joins that compare `ItemId`, `ProjId`, `ProjCategoryId`, `ProjLinePropertyId`, `ProjTaxItemGroup`, `ProjTaxGroup`, main account, dimension, … (also in private `insertBillingConsolidationDetails_ConsolidationByItem`). | can't add a join condition (R-10) |

The community signature for `createSalesLine` (doc version 0.2) is **correct**.

**D-05 – Consolidation**

- Grouping happens in the **query** (#4): one `SubBillSalesLineConsolidated` row per group. Without item consolidation the group contains the schedule line `RecId`, so one SO line = one schedule line. With *Consolidate by item* or by billing code, lines with the same item, project fields, address, frequency, main account, dimension, text, external item, customer reference, price unit, revenue split fields … are merged.
- *Consolidate all periods* adds group-by on the temp data source (`addScheduleTempGroupByFields`); it doesn't change the schedule line grouping.
- To keep lines with different activities apart (FR-06) we add `HOGsbaActivityNumber` to the group-by (#4). The link-back after insert (#9) can't see our field. **Needs a debugger/test run before we rely on it** (TC-06, R-10).

**D-06 – Copy schedule**

Form `SubBillCopySchedule`, form method `public SubBillScheduleLine copyScheduleLine(SubBillCopyScheduleLineTmp, SubBillScheduleLine _srcScheduleLine, …)`: `buf2Buf(_srcScheduleLine, destScheduleLine)`, then overrides dates, quantities, amounts, status, text. `ProjId` is not overridden, so the activity is copied together with its project. **No hook needed (FR-08).** A different project chosen in the copy dialog goes to the new header only.

**D-09 – Termination credits and customer split**

| Path | Chain | Activity hook |
|---|---|---|
| *Terminate* (single line, form `SubBillTerminateDialog`) and mass termination | `SubBillMassTerminationParmUpdate.massTerminationProcess()` → `SubBillMassTermination::massTerminationProcess(SubBillMassTerminationParmUpdate, SubBillScheduleLineTmp)` (public static) → `SubBillMassTermination::consolidateInvoiceProcessingData(boolean, ParmId, SubBillInvTransType = SalesOrder)` → its own `addScheduleLineSelectionFields(QueryBuildDataSource, SubBillInvTransType)`, `addScheduleLineGroupByFields(` same `)`, `initializeSubBillSalesLineConsolidated(SubBillSalesLineConsolidated, SubBillScheduleTable, SubBillScheduleLine _scheduleLineTable, SubBillScheduleLine, SubBillDetailLine, SubBillSalesLine, boolean _customerConsolidate, SubBillInvTransType, ParmId)` (all public static) → `SubBillCreateSalesOrder::generateSalesOrderFromConsolidatedData` → #7 → #8. Project fields selected and grouped only for invoice type *Sales order* with the project feature on; `RecId` and `LineNum` always selected and grouped, so **one SO line per schedule line** (no merging). | Wrap the three `SubBillMassTermination` methods; #8 is shared. |
| *Credit adjustment* | Credit detail lines are billed by the normal *Generate invoice* path (`SubBillTerminationCreditLine` on the consolidated buffer). | Covered by D-03 hooks |
| Customer split | `SubBillCreateSalesOrderCustomerSplit` (public final): `assignSalesLineConsolidatedValues(SubBillCustomerSplitLine, SubBillSalesLineConsolidated)` (protected static) sets `ProjId` etc. from **`SubBillCustomerSplitLine`**. Termination with customer split and prorate (`SubBillMassTermination::subBillTerminationProcessingData`, private) sets `ProjId` from the **original sales order header**. | OP-19. Debugger needed. |

**D-11 – Preview feature *Database Query Optimization and Reduced Chattiness***

Feature class `SubBillQueryPerfImprovementFeature`. Usages: `SubBillCreateSalesOrder.updateBillingDetails_NoConsolidation` (after insert), `SubBillScheduleLine`, `SubBillDetailLine`, `SubBillRenewalLine`, `SalesFormletterProviderSubBill_Extension`. **Not** used in #3 to #8, so static reading says our hooks are not affected. Repeat D-03 with the feature on only if hog-uat enables it.

**D-13 – Renewals**

| Path | Method | Visibility | How lines are created | Activity |
|---|---|---|---|---|
| *Renew automatically* (from *Generate invoice*) | `SubBillCreateSalesOrder::processRenewal` (internal) → `SubBillScheduleLine::createAutoRenewalLines(RecId, boolean, boolean _renewalUtility = false)` | public static | **Same schedule line**: end date extended, `SubBillRenewalLine` record created. No new schedule line. | kept automatically |
| Milestone auto-renewal | `SubBillScheduleLine.applyMilestoneAutoRenewal` → `createAutoRenewedMilestoneLines` → `copyMilestoneScheduleLine` | internal | `buf2Buf` | copied automatically |
| *Add renewal term*, *Lines to add per renewal* | `SubBillRenewalLine` records (renewal terms) on the existing line | – | no new schedule line | kept automatically |
| Revenue split children | `SubBillScheduleLine.createRevenueSplitChild` (internal), `rolldownProjFields` (private) | internal/private | Children get the parent's `ProjId` | **Not inherited** unless we add it (OP-05) |
| Support and renewal (posting a sales invoice) | `SalesInvoiceJournalPostSubBill_Extension.run` → `SubBillScheduleTable::create(SalesTable, SubBillSupportRenewalProcessTable)` and **`SubBillScheduleLine::createRenewalLine(…)` – `internal static`** | internal | Field-by-field from the renewal process line and the `SalesLine`. **Neither header nor line gets a `ProjId`.** | FR-12 has no project to match: OP-18 |

Needs the debugger: D-03 (path really runs), D-05 (by-item link-back, R-10), D-09 (customer split, termination with split), D-13 (auto-renewal end to end).

### 4.2 PowerShell

PowerShell for D-01 (run on the VM; read-only):

```powershell
# Locate the packages folder (VHD: C:, cloud-hosted: K:)
$pld = @('C:\AosService\PackagesLocalDirectory', 'K:\AosService\PackagesLocalDirectory') |
    Where-Object { Test-Path $_ } | Select-Object -First 1
"Packages folder: $pld"

# Which package contains Subscription billing?
Get-ChildItem $pld -Directory | Where-Object Name -like '*Subscription*' | Select-Object Name

# Standard SubBill* objects of the relevant types, by package
Get-ChildItem $pld -Directory | ForEach-Object {
    $pkg = $_.Name
    Get-ChildItem $_.FullName -Directory -ErrorAction SilentlyContinue | ForEach-Object {
        foreach ($type in 'AxTable', 'AxClass', 'AxForm', 'AxDataEntityView') {
            $dir = Join-Path $_.FullName $type
            if (Test-Path $dir) {
                Get-ChildItem $dir -Filter 'SubBill*.xml' -File |
                    Select-Object @{n='Package';e={$pkg}}, @{n='Type';e={$type}}, @{n='Name';e={$_.BaseName}}
            }
        }
    }
} | Sort-Object Type, Name | Export-Csv 'docs\discovery\D-01-subbill-objects.csv' -NoTypeInformation -Encoding UTF8
```

The CSV contains object names only (no source code). Run from the repo root.

PowerShell for D-12 (read-only; lists Honegger extension objects that touch our area):

```powershell
$pld = @('C:\AosService\PackagesLocalDirectory', 'K:\AosService\PackagesLocalDirectory') |
    Where-Object { Test-Path $_ } | Select-Object -First 1

# Package folders of Honegger's and the ISV's models (names VERIFY: shown here by pattern)
$packages = Get-ChildItem $pld -Directory | Where-Object Name -match '^(HG|Axtension|AXT)'
$packages | Select-Object Name

# Extension objects in those packages that mention SubBill or ActivityNumber (file names only)
$packages | ForEach-Object {
    Get-ChildItem $_.FullName -Recurse -File -Filter '*.xml' |
        Select-String -Pattern 'SubBill', 'ActivityNumber' -List |
        Select-Object @{n='Package';e={$_.Path.Substring($pld.Length + 1).Split('\')[0]}},
                      @{n='Type';e={Split-Path (Split-Path $_.Path -Parent) -Leaf}},
                      @{n='Object';e={[IO.Path]::GetFileNameWithoutExtension($_.Path)}}
} | Sort-Object Package, Type, Object | Format-Table -AutoSize

# Prefix check: first 30 class names in Honegger's package(s)
$packages | Where-Object Name -like 'HG*' | ForEach-Object {
    Get-ChildItem $_.FullName -Recurse -Directory -Filter AxClass
} | Get-ChildItem -File | Select-Object -First 30 -ExpandProperty BaseName
```

## 5. X++ drafts

All drafts: **not compiled**. Standard names in them are confirmed on the VM (§1, §4.1) unless marked VERIFY.

### 5.1 Billing schedule line: validation and reset

```xpp
/// <summary>
/// Activity number on billing schedule lines: validation and reset.
/// DRAFT – not compiled. Standard names confirmed 09.11.2026 (D-01, D-04).
/// </summary>
[ExtensionOf(tableStr(SubBillScheduleLine))]
final class SubBillScheduleLine_HOGsba_Extension
{
    public boolean validateField(FieldId _fieldIdToCheck)
    {
        boolean ret = next validateField(_fieldIdToCheck);

        if (ret && _fieldIdToCheck == fieldNum(SubBillScheduleLine, HOGsbaActivityNumber))
        {
            ret = this.hogsbaValidateActivity();
        }
        return ret;
    }

    public boolean validateWrite()
    {
        boolean ret = next validateWrite();

        // Also covers data entity imports (FR-07), which may not call validateField.
        return ret && this.hogsbaValidateActivity();
    }

    public void modifiedField(FieldId _fieldId)
    {
        next modifiedField(_fieldId);

        // Standard handles ProjId in modifiedFieldValue (initProjFields). VERIFY at test that
        // the form's modifiedFieldValue -> modifiedField chain reaches this code (TC-04);
        // otherwise wrap modifiedFieldValue(FieldName, int) instead.
        if (_fieldId == fieldNum(SubBillScheduleLine, ProjId))
        {
            this.HOGsbaActivityNumber = '';                          // FR-04
        }
    }

    /// <summary>Activity must be empty or belong to the line's project (FR-03).</summary>
    public boolean hogsbaValidateActivity()
    {
        if (!this.HOGsbaActivityNumber)
        {
            return true;
        }
        // Same check as SalesLine.checkActivity(). Fails with a standard message
        // also when the line has no project.
        return ProjActivityHelper::validateProjActivityParentLinkV2(this.ProjId, this.HOGsbaActivityNumber);
    }
}
```

### 5.2 Carry the activity through consolidation

```xpp
/// <summary>
/// Carries the project activity from the billing schedule line into
/// SubBillSalesLineConsolidated and from there to the sales order line.
/// DRAFT – not compiled. Signatures copied from the VM (10.40.3708.49257), 09.11.2026.
/// Wrapped static methods need the static keyword and no default values (C-01 §5).
/// </summary>
[ExtensionOf(classStr(SubBillCreateSalesOrder))]
final class SubBillCreateSalesOrder_HOGsba_Extension
{
    public static QueryBuildDataSource addScheduleLineSelectionFields(
        QueryBuildDataSource    _scheduleLine,
        SubBillInvTransType     _invoiceTransType,
        boolean                 _itemConsolidate,
        boolean                 _consolidateByBillingCode)
    {
        QueryBuildDataSource qbds = next addScheduleLineSelectionFields(
            _scheduleLine, _invoiceTransType, _itemConsolidate, _consolidateByBillingCode);

        if (SubBillProjectEnableFeature::isEnabled())                 // same condition as ProjId
        {
            qbds.addSelectionField(fieldNum(SubBillScheduleLine, HOGsbaActivityNumber));
        }
        return qbds;
    }

    public static QueryBuildDataSource addScheduleLineGroupByFields(
        QueryBuildDataSource    _scheduleLine,
        SubBillInvTransType     _invoiceTransType,
        boolean                 _itemConsolidate,
        boolean                 _consolidateByBillingCode)
    {
        QueryBuildDataSource qbds = next addScheduleLineGroupByFields(
            _scheduleLine, _invoiceTransType, _itemConsolidate, _consolidateByBillingCode);

        // FR-06. Same condition as the standard ProjId group-by. See R-10 before relying on it.
        if (_invoiceTransType == SubBillInvTransType::SalesOrder && SubBillProjectEnableFeature::isEnabled())
        {
            qbds.addGroupByField(fieldNum(SubBillScheduleLine, HOGsbaActivityNumber));
        }
        return qbds;
    }

    public static SubBillSalesLineConsolidated initializeSubBillSalesLineConsolidated(
        SubBillSalesLineConsolidated    _subBillSalesLineConsolidated,
        SubBillScheduleTable            _scheduleHeaderTable,
        SubBillScheduleLine             _scheduleLineTable,
        SubBillScheduleLine             _parentScheduleLine,
        SubBillDetailLine               _detailLine,
        SubBillSalesLine                _subBillSalesLine,
        boolean                         _itemConsolidate,
        boolean                         _consolidateByBillingCode,
        SubBillInvTransType             _invoiceTransType,
        ParmId                          _curParmId,
        boolean                         _doConsolidateAllPeriods,
        QueryRun                        _queryRun)
    {
        SubBillSalesLineConsolidated ret = next initializeSubBillSalesLineConsolidated(
            _subBillSalesLineConsolidated, _scheduleHeaderTable, _scheduleLineTable, _parentScheduleLine,
            _detailLine, _subBillSalesLine, _itemConsolidate, _consolidateByBillingCode,
            _invoiceTransType, _curParmId, _doConsolidateAllPeriods, _queryRun);

        if (ret.ProjId)                                             // standard copied the project
        {
            ret.HOGsbaActivityNumber = _scheduleLineTable.HOGsbaActivityNumber;
        }
        return ret;
    }

    public static SalesLine createSalesLine(
        SalesLine                       _salesLine,
        SubBillSalesLineConsolidated    _salesLineConsolidated,
        SalesId                         _salesIdCon,
        boolean                         _allocEnabled,
        boolean                         _splitByItemGroup,
        boolean                         _isInvoiceCreator,
        boolean                         _includeBillingDatesToItem)
    {
        SalesLine salesLine = next createSalesLine(
            _salesLine, _salesLineConsolidated, _salesIdCon,
            _allocEnabled, _splitByItemGroup, _isInvoiceCreator, _includeBillingDatesToItem);

        // Not inserted yet: createSalesOrder calls salesLine.createLine(...) afterwards.
        HOGsbaSalesLineActivity::apply(salesLine, _salesLineConsolidated);
        return salesLine;
    }
}
```

```xpp
/// <summary>
/// Termination credits (FR-13): same three hooks on SubBillMassTermination.
/// DRAFT – not compiled. Signatures copied from the VM, 09.11.2026.
/// </summary>
[ExtensionOf(classStr(SubBillMassTermination))]
final class SubBillMassTermination_HOGsba_Extension
{
    public static QueryBuildDataSource addScheduleLineSelectionFields(QueryBuildDataSource _scheduleLine, SubBillInvTransType _invoiceTransType)
    {
        QueryBuildDataSource qbds = next addScheduleLineSelectionFields(_scheduleLine, _invoiceTransType);

        if (_invoiceTransType == SubBillInvTransType::SalesOrder && SubBillProjectEnableFeature::isEnabled())
        {
            qbds.addSelectionField(fieldNum(SubBillScheduleLine, HOGsbaActivityNumber));
        }
        return qbds;
    }

    public static QueryBuildDataSource addScheduleLineGroupByFields(QueryBuildDataSource _scheduleLine, SubBillInvTransType _invoiceTransType)
    {
        QueryBuildDataSource qbds = next addScheduleLineGroupByFields(_scheduleLine, _invoiceTransType);

        // Same condition as the standard ProjId group-by in this method (read 09.11.2026).
        // Termination always groups by schedule line RecId, so this never merges lines.
        if (_invoiceTransType == SubBillInvTransType::SalesOrder && SubBillProjectEnableFeature::isEnabled())
        {
            qbds.addGroupByField(fieldNum(SubBillScheduleLine, HOGsbaActivityNumber));
        }
        return qbds;
    }

    public static SubBillSalesLineConsolidated initializeSubBillSalesLineConsolidated(
        SubBillSalesLineConsolidated    _subBillSalesLineConsolidated,
        SubBillScheduleTable            _scheduleHeaderTable,
        SubBillScheduleLine             _scheduleLineTable,
        SubBillScheduleLine             _parentScheduleLine,
        SubBillDetailLine               _detailLine,
        SubBillSalesLine                _subBillSalesLine,
        boolean                         _customerConsolidate,
        SubBillInvTransType             _invoiceTransType,
        ParmId                          _curParmId)
    {
        SubBillSalesLineConsolidated ret = next initializeSubBillSalesLineConsolidated(
            _subBillSalesLineConsolidated, _scheduleHeaderTable, _scheduleLineTable, _parentScheduleLine,
            _detailLine, _subBillSalesLine, _customerConsolidate, _invoiceTransType, _curParmId);

        if (ret.ProjId && ret.ProjId == _scheduleLineTable.ProjId)
        {
            ret.HOGsbaActivityNumber = _scheduleLineTable.HOGsbaActivityNumber;
        }
        return ret;
    }
}
```

### 5.3 Copy rule

```xpp
/// <summary>
/// Copies the activity from the consolidated billing buffer to a sales order line.
/// Plain class (not an extension class) so it can be unit tested without the generate-invoice process.
/// DRAFT – not compiled.
/// </summary>
final class HOGsbaSalesLineActivity
{
    /// <summary>Sets the activity on the sales order line if the project matches (design choice 5).</summary>
    public static void apply(SalesLine _salesLine, SubBillSalesLineConsolidated _consolidated)
    {
        if (!_consolidated.HOGsbaActivityNumber
            || _salesLine.ActivityNumber == _consolidated.HOGsbaActivityNumber)
        {
            return;
        }
        if (_salesLine.ProjId != _consolidated.ProjId)
        {
            warning(strFmt("@HOGsba:...", _consolidated.HOGsbaActivityNumber)); // VERIFY label (§2)
            return;
        }
        // Plain assignment on purpose: SalesLine.modifyField(ActivityNumber) would call
        // initFromProjActivity and could replace ProjCategoryId, which must stay as on the
        // billing schedule line (§3, after static discovery, point 7).
        _salesLine.ActivityNumber = _consolidated.HOGsbaActivityNumber;
    }
}
```

`HOGsbaSalesLineActivity::apply` is the unit under test in doc 07 (TC-U03 to TC-U05). It doesn't depend on the generate-invoice process. **Doc 07 still describes the older signature with `SubBillScheduleLine`; update it when the code is written.**

## 6. Performance

- No extra reads: the activity travels in the consolidation query and the consolidated buffer, like `ProjId`. The `SubBillScheduleLine::find` of the earlier draft is gone.
- One more group-by column in the consolidation query. One more column on `SubBillSalesLineConsolidated` (a working table, rows per run).
- No new loops, no set-based operations, no new indexes on `SalesLine`.

## 7. Error handling

- Validation errors (FR-03) stop saving the billing schedule line, as standard field validation does.
- During invoice generation, a mismatch (design choice 5) **never** stops generation. It produces a warning in the infolog/batch log, and the sales order line is created without an activity.
