# 07 – Test plan

Conventions: [C-02](conventions/d365-finops/02-testing.md). **R** in the last column = regression case, run again on every service update.

## 1. Test data (fill in on the VM)

| Item | Value |
|---|---|
| Legal entity | |
| Project P1 (T&M, invoiceable stage) | , activities A1 = , A2 = |
| Project P2 | , activity B1 = |
| Item (service / non-stock) | |
| Customer (with bill-to address) | |
| Billing schedule group | |
| Feature states | as hog-uat on (date) – file in `docs/discovery/feature-states/` |

## 2. Unit tests (SysTest, package `HOGsbaSubBillActivityTest`, category `HOGsba`)

| ID | Class / method (when_given_then) | Checks |
|---|---|---|
| TC-U01 | `HOGsbaScheduleLineActivityTest.validateActivity_activityOfLineProject_true` | FR-03 |
| TC-U02 | `HOGsbaScheduleLineActivityTest.validateActivity_activityOfOtherProject_false` | FR-03 |
| TC-U03 | `HOGsbaSalesLineActivityTest.apply_sameProject_activityCopied` | FR-05 |
| TC-U04 | `HOGsbaSalesLineActivityTest.apply_differentProject_notCopiedWarning` | Design choice 5 |
| TC-U05 | `HOGsbaSalesLineActivityTest.apply_noActivity_salesLineUnchanged` | FR-09 |
| TC-U06 | `HOGsbaScheduleLineActivityTest.modifyProject_activitySet_activityCleared` | FR-04 |

## 3. Functional test cases (VM, then hog-uat)

| ID | Scenario | Steps (short) | Expected | R |
|---|---|---|---|---|
| TC-01 | Enter activity | Billing schedule for P1, line with item, Project tab → Activity A1, save | Saved | |
| TC-02 | Activity of another project | Same line → Activity B1 | Error; not saved | |
| TC-03 | No project on the line | Line without Project ID → enter activity | Field not editable, or error | |
| TC-04 | Project changed | Line with A1 → change Project ID to P2 | Activity cleared | |
| TC-05 | Generate invoice, one line | Line with A1 → *Generate invoice*, *Create sales order* | SO line: Project P1, **Activity number A1** | R |
| TC-06 | Consolidation by item, different activities | Two lines, same item, A1 and A2, consolidate by item | **Two** SO lines, A1 and A2. Not merged. | R |
| TC-07 | Consolidate all periods | One line with A1, three open periods, consolidate all periods | One SO line with A1 | R |
| TC-08 | Batch | As TC-05, through the periodic *Generate invoice* batch job | As TC-05 | R |
| TC-09 | Through to posting | TC-05 with *Post invoice automatically* | Project invoice posted; posted project transaction shows activity A1 (only if D-07 confirms standard carries it) | R |
| TC-10 | Line without activity | Line with project, no activity → generate | SO line created, Activity number empty, no message | R |
| TC-11 | Schedule without project | Billing schedule without project → generate | Behaves exactly as before the extension | R |
| TC-12 | Copy schedule | Copy the TC-01 schedule | New line has A1 (per FR-08) | |
| TC-13 | Data entity | Export lines (activity column present); import a line with B1 for project P1 | Export OK; import rejected with the validation error | |
| TC-14 | Customer split | Schedule with customer split, line with A1 → generate | Every SO line has A1 | |
| TC-15 | Termination with *Issue credit* | Terminate the TC-05 schedule after one invoiced period, *Issue credit* | Credit line(s) carry A1 (FR-13) | R |
| TC-15b | Termination with *Credit adjustment* | As TC-15 with *Credit adjustment* | Adjusted line(s) carry A1 (FR-13) | |
| TC-18 | Renewal | Line with A1 and *Renew automatically*, run the renewal | Renewed line has A1 (FR-11) | R |
| TC-19 | Add renewal term / lines per renewal | Line with A1 → *Add renewal term* | New or extended line has A1 (FR-11) | |
| TC-20 | Support and renewal | SO line for P1 with A1, renewal item → post invoice | Created billing schedule line has P1 / A1 (FR-12) | |
| TC-21 | Coexistence with Honegger models | Run TC-05, TC-06 and TC-15 on a VM with `HGExtension` and the Axtension models deployed | Same results; no conflict with existing extensions (D-12) | R |
| TC-16 | Preview feature on and off | Repeat TC-05 and TC-06 with *(Preview) Database Query Optimization and Reduced Chattiness* on, then off | Same result both times (only if hog-uat uses the feature, D-11) | R |
| TC-17 | Security | User with view-only access to billing schedules | Can see the activity, can't change it | |

## 4. Results log

| ID | Date | Tester | Env | Billing schedule / SO / invoice | Result | Evidence |
|---|---|---|---|---|---|---|
| TC-01 | | | VM | | | |
| … | | | | | | |
