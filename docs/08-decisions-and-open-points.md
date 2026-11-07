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
| OP-16 | Environment | Which environment are the screenshots of 07.11.2026 from (hog-uat or the VM)? If the VM holds a copy of Honegger's data, Honegger must agree that Claude on the VM may read it, or we use a data-free test company. | Gabriel | Discovery |

## 3. Discovery items (dev VM)

D-01 to D-13, see [doc 02 §4](02-technical-design.md#4-discovery-on-the-dev-vm-before-coding). Status:

| ID | Status | Result |
|---|---|---|
| D-01 | open | |
| D-02 | **done** | Form `SubBillScheduleTable`, data source `SubBillScheduleLine`, field `ProjId` |
| D-03 | open | |
| D-04 | open | |
| D-05 | open | |
| D-06 | open | |
| D-07 | open – **do first** | |
| D-08 | open | |
| D-09 | open | |
| D-10 | open | |
| D-11 | open (only if relevant) | |
| D-12 | open | |
| D-13 | open | |

## 4. Risks

| ID | Risk | Impact | Mitigation |
|---|---|---|---|
| R-01 | Standard names differ from the community sources | Rework of hooks | Discovery D-01 to D-06 before coding |
| R-02 | Microsoft adds an activity to billing schedule lines later | Duplicate field | Prefixed name, so no compile conflict; migrate data to the standard field then |
| R-03 | The preview performance feature changes the SO creation path | Activity silently missing | D-11, TC-16; narrow hook |
| R-04 | Consolidation merges lines with different activities | Wrong activity on revenue | DEC-04, TC-06 |
| R-05 | Setting `ActivityNumber` by code skips standard side effects | Inconsistent SO line (e.g. dimensions) | D-04, D-07 |
| R-06 | VM version newer than Honegger's | Package doesn't deploy, or behaves differently | OP-03 |
| R-07 | The project fee journal invoice type is used | Extension never runs | OP-05 |
| R-08 | Customer rejects AI-assisted code or data access | No deployment / no MCP | Code reviewed and owned by AlfaPeople; full test evidence; MCP read-only with written approval |
| R-09 | `HGExtension` or an ISV model already wraps the same methods, or already adds an activity | Conflicting CoC behaviour or a duplicate field | D-12 before coding; TC-21 |
