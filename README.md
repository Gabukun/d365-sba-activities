# HOG – Activity number on Subscription billing (SBA) schedule lines

| | |
|---|---|
| **Customer** | Honegger AG |
| **Product** | Dynamics 365 Finance & Operations: Subscription billing (Recurring contract billing) with Project management and accounting |
| **Dev environment** | VHD dev VM `dvhd10049vm` (F&O `dvhdm100049dm`, 10.0.49) |
| **Status** | Draft v0.2 – 07.11.2026. Scope and naming agreed (DEC-02, DEC-09). Discovery: D-01, D-02, D-04, D-06, D-10 done; static part of D-03, D-05, D-09, D-11, D-13 done (09.11.2026); D-12 blocked (Honegger models not on the VM). No X++ compiled yet. |
| **Owner** | Gabriel Burgener, AlfaPeople Switzerland |

## Purpose

Billing schedule lines that are linked to a project have no **activity**. The sales order line generated from them has the standard **Activity number** field, which today stays empty. This extension:
- adds the activity to the billing schedule line, validated against the line's project;
- copies it to `SalesLine.ActivityNumber` when *Generate invoice* creates the sales order.

## Documents

| # | Document | Content |
|---|---|---|
| 01 | [Functional design](docs/01-functional-design.md) | Background, scope (FR-01…10), process |
| 02 | [Technical design](docs/02-technical-design.md) | Standard objects (with confidence), objects to create, discovery D-01…11, X++ drafts |
| 04 | [Prerequisites and setup](docs/04-prerequisites-and-setup.md) | Versions, features to enable, mirroring Honegger's feature states, test setup |
| 06 | [Dev environment and working model](docs/06-dev-environment-and-workflow.md) | Who does what where, Git on the VM (PowerShell), model creation, Claude access options (MCP, VM) |
| 07 | [Test plan](docs/07-test-plan.md) | SysTest unit tests, functional test cases, results log |
| 08 | [Decisions and open points](docs/08-decisions-and-open-points.md) | DEC, OP, discovery status, risks |
| ADR | [ADR-001 GitHub now, Azure DevOps at handover](docs/adr/ADR-001-github-now-azure-devops-at-handover.md) | Source control and ALM decision |

Doc numbers follow the HOG project template (d365-pma-indirectcosts). 03 (data model) is part of 02, and 05 (integration) doesn't apply.

## Conventions

Master: [youniform](https://github.com/youserpath/youniform) `conventions/`. This repo keeps a snapshot in `docs/conventions/` (DEC-11) so it is complete at handover.

| # | Document | Content |
|---|---|---|
| C-00 | [Documentation](docs/conventions/documentation.md) | Document set, IDs, markers, ADRs |
| C-01 | [Naming and extensions](docs/conventions/d365-finops/01-naming-and-extensions.md) | Packages, prefixes, extension naming, CoC rules, data entities, BP checks, repo layout, doc markers |
| C-02 | [Testing](docs/conventions/d365-finops/02-testing.md) | SysTest, ATL, test packages, functional test cases, Azure DevOps Test Plans import, RSAT/Task recorder status |
| C-03 | [ALM and source control](docs/conventions/d365-finops/03-alm-and-source-control.md) | Git vs TFVC, branching, build/package, platform status, GitHub vs Azure DevOps parity and migration |

## Repository layout

```
d365-sba-activities/
├── README.md
├── CLAUDE.md                     (instructions for AI-assisted sessions)
├── docs/
│   ├── 01 … 08                   (project documents)
│   ├── adr/                      (decision records)
│   ├── conventions/              (snapshot of youniform conventions, C-00 … C-03)
│   └── discovery/                (results from the VM: object lists, feature states – created when needed)
├── Metadata/                     (X++ packages, linked into PackagesLocalDirectory – see doc 06)
│   ├── HOGsbaSubBillActivity/       (after discovery)
│   └── HOGsbaSubBillActivityTest/
└── Projects/                     (Visual Studio projects)
```

## Next steps

1. Confirm the field name `HOGsbaActivityNumber` (DEC-01) and answer OP-15 (`HG` or `HOG`) and OP-16 (which environment and data the VM holds).
2. Mirror Honegger's feature states on the VM (doc 04 §3) and enable Subscription billing (doc 04 §2).
3. Run **D-07 first** (does a manual Activity number on the SO line reach the posted project transactions?), then D-12 (overlap with `HGExtension`), D-01, D-03 to D-06, D-13.
4. Decide whether Claude runs on the VM (OP-14, doc 06 §4a). That changes who does the discovery steps.
5. Create the models (doc 06 §3); rewrite the drafts in doc 02 against the confirmed names.
6. Build, test (doc 07), PR.
