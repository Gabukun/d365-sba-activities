# C-00 – Documentation conventions

> **Snapshot.** Master copy: [youniform `conventions/documentation.md`](https://github.com/youserpath/youniform/blob/main/conventions/documentation.md), snapshot of 07.11.2026. Don't edit here; change youniform and refresh this copy (DEC-11).

Applies to every project repository. Team decisions of 07.11.2026, based on d365-pma-indirectcosts and d365-sba-activities.

## 1. Standard document set (D365 FinOps extension)

| # | File | Content | When |
|---|---|---|---|
| – | `README.md` | Purpose, status, document index, layout, next steps | Always |
| – | `CLAUDE.md` | Rules for AI-assisted sessions; points to youniform | Always |
| 01 | `docs/01-functional-design.md` | Background, scope (FR-nn), process, roles | Always |
| 02 | `docs/02-technical-design.md` | Standard objects with confidence, objects to create, design choices, discovery (D-nn), X++ drafts | Always |
| 03 | `docs/03-data-model.md` | New tables, EDTs, enums, table extensions | When new tables exist; otherwise part of 02 |
| 04 | `docs/04-…` | Setup guide, or prerequisites and setup | Always |
| 05 | `docs/05-integration-contract.md` | What external systems send and must not send | Only with integrations |
| 06 | `docs/06-dev-environment-and-….md` | Working model, VM, Git, build, package, deploy | Always |
| 07 | `docs/07-test-plan.md` | Test data, unit tests, functional test cases, results log | Always |
| 08 | `docs/08-decisions-and-open-points.md` | DEC, OP, discovery status, risks | Always |
| – | `docs/adr/ADR-nnn-<topic>.md` | Architecture and ALM decisions with alternatives | Per decision |
| – | `docs/conventions/` | Snapshot of youniform conventions | Always |
| – | `docs/discovery/` | Results from the dev VM: object lists, feature states (no business data) | As needed |
| – | `docs/evidence/TC-nn/` | Test evidence (no customer production data) | As needed |

Numbers stay fixed even when a document isn't needed, so "doc 07" means the test plan in every repository.

## 2. IDs and markers

| ID / marker | Meaning | Lives in |
|---|---|---|
| `FR-nn` | Functional requirement | 01 |
| `D-nn` | Discovery item: a step on the dev VM with expected evidence | 02 (status in 08) |
| `VERIFY` | Standard name or fact not yet confirmed on the dev VM or in Microsoft docs | anywhere |
| `TC-nn`, `TC-Unn` | Functional test case, unit test | 07 |
| `DEC-nn` | Decision, with date, reason and status (Proposed / Agreed) | 08 |
| `OP-nn` | Open point, with owner and "needed before" | 08 |
| `R-nn` | Risk with mitigation | 08 |
| `ADR-nnn` | Decision record | `docs/adr/` |

- IDs are never reused or renumbered. Closed items stay, marked **Closed** with the date and the answer.
- Commit messages and pull requests reference these IDs, not issue numbers, so references survive a move to another platform.

## 3. Writing

- English; plain, short sentences; tables for structured content.
- Every Microsoft fact has a Microsoft Learn link. Community sources are named as such.
- X++ in documents is marked *draft, not compiled* until it has been built on the dev VM.
- Shell commands for the team are PowerShell.
