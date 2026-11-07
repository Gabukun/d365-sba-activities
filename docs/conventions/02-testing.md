# C-02 – Testing conventions (FinOps extensions)

Reusable across Honegger (HOG) FinOps extension projects. Researched 2026-11-07. Tags as in [C-01](01-naming-and-extensions.md).

## 1. What Microsoft expects

- "Unit testing is required if you're customizing or extending your solution" ([Implementation guide – test types](https://learn.microsoft.com/dynamics365/guidance/implementation-guide/testing-strategy-test-types)). **[MS]**
- Developers unit test in a development environment, and regression tests should be automated ([Testing strategy checklist](https://learn.microsoft.com/dynamics365/guidance/implementation-guide/testing-strategy-checklist)). **[MS]**

Microsoft splits testing into these levels ([RSAT overview, "Intended usage"](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/perf-test/rsat/rsat-overview)):

| Level | Microsoft tool | Our use |
|---|---|---|
| Unit | SysTest framework | X++ test classes in a test package. Mandatory for every piece of logic we add. |
| Component | SysTest + **Acceptance Test Library (ATL)** | Where ATL covers the module (sales orders, items, customers). |
| Business cycle / UAT | RSAT + Task recorder + Azure DevOps Test Plans | **No new investment.** See section 4. |
| Data integration | Data task automation (DMF) | Only when a project touches entities in a relevant way. |

## 2. X++ tests (SysTest)

| Rule | Tag / source |
|---|---|
| A separate **test package** whose name contains `Test`, e.g. `HOGsbaSubBillActivityTest`. The default build discovers test modules by that word. | MS ([Testing and validations](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/perf-test/testing-validation)) |
| The test package is **excluded from the deployable package** (`PackagingExclusions` or the `!*Tests` pattern). | MS ([Exclude test packages](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/dev-tools/exclude-test-packages)) |
| Test classes extend `SysTestCase`. Methods carry `[SysTestMethod]`. | MS ([Testing support](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/perf-test/testing-support)) |
| Tag every test with `[SysTestCategory('HOG')]` plus the feature code (e.g. `'HOGsba'`), so a run can be filtered. | MS mechanism ([SysTest filtering](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/perf-test/systest-filtering)), Team values |
| Class name: `<Prefix><Subject>Test`, e.g. `HOGsbaScheduleLineActivityTest`. | Community / Microsoft examples |
| Method name: `when_given_then` style, e.g. `copyActivity_lineWithActivity_salesLineHasActivity`. | MS ([Public application tests](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/perf-test/public-application-tests)) |
| Data is rolled back after each test (`SysTestTransaction` defaults to AutoRollback). Never depend on demo data that may not exist. Create what the test needs. | MS |
| Test data through **ATL** where it exists: `AtlDataRootNode data`, entities `AtlEntity<Module><Entity>`, e.g. `data.sales().salesOrders().createDefault()`. | MS ([ATL](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/perf-test/acceptance-test-library), [entities](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/perf-test/concepts-entities)) |
| Where ATL has no entity (e.g. Subscription billing, VERIFY), write small builder methods in the test package (`HOGsbaTestDataBuilder`). Never call UI forms from tests. | Team |
| References of the test package: our package, `TestEssentials`, the ATL packages (exact folder names: VERIFY on the VM). | Community |

**Where tests run.** On the dev VM (Visual Studio > Test Explorer). Microsoft-hosted build agents do **not** run SysTest: "no support for X++ unit testing (SysTest), database synchronization" ([Hosted build automation](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/dev-tools/hosted-build-automation)). In a Unified Developer Environment (UDE), the pipeline task *Execute Unit Tests in Unified Environment* runs them against a cloud environment ([UDE pipeline test](https://learn.microsoft.com/power-platform/developer/unified-experience/finance-operations-pipeline-test)). **[MS]**

## 3. Functional test cases

| Rule | Tag |
|---|---|
| Test cases live in the repo in `docs/07-test-plan.md`: ID `TC-nn`, precondition, steps, expected result. Each FR maps to at least one TC. | Team (same format as d365-pma-indirectcosts) |
| Results are logged in the same document: date, tester, environment (VM / hog-uat), document numbers (billing schedule, sales order, invoice), result, evidence link. | Team |
| Evidence (screenshots, exported lines) goes to `docs/evidence/TC-nn/`. **No customer production data** in the repo. | Team |
| If the customer wants test cases in **Azure DevOps Test Plans**, convert the Markdown table to CSV and use *Import test cases from CSV/XLSX*. Required columns: ID, Work Item Type (`Test Case`), Title, Test Step, Step Action, Step Expected, Area Path, Assigned To, State. One row per step. | MS ([Bulk import or export test cases](https://learn.microsoft.com/azure/devops/test/bulk-import-export-test-cases)) |
| Authoring test plans in Azure DevOps needs **Basic + Test Plans** (or Visual Studio Enterprise / Test Professional). **Basic** can run tests and record outcomes. | MS ([Manual test access](https://learn.microsoft.com/azure/devops/test/manual-test-permissions)) |

## 4. RSAT and Task recorder

- **RSAT** "is marked for deprecation and won't be supported after May 15, 2027" ([RSAT overview](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/perf-test/rsat/rsat-overview)). It needs Azure DevOps Test Plans. **Don't build new RSAT suites.** **[MS]**
- **Task recorder:** "Starting in January 2026, you can only create, edit, or use Task Recordings in Sandbox and Production environments. Cloud-hosted environments no longer support this capability" ([Task recorder](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/user-interface/task-recorder)). Recordings for user documentation or training are made in the customer sandbox (hog-uat), not on our dev VM. **[MS]**

## 5. Feature flags and regression

- Features in Feature management change code paths. Run the functional cases with the **same feature states as the customer's target environment**. Mirror them from the customer environment with the **Feature management** data entity ([Feature management data entity](https://learn.microsoft.com/training/modules/feature-management-finance-operations/12-data-entities)).
- When a preview feature touches our hook (named in the project's doc 08), run the core TCs with the feature both on and off.
- On every service update: rebuild on the new version and run the core TCs (marked *Regression* in doc 07) before the update reaches production.
