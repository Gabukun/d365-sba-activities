# ADR-001: GitHub now, customer Azure DevOps at handover

**Status:** Proposed. **Date:** 2026-11-07. **Facts:** [C-03](../conventions/03-alm-and-source-control.md).

## Context

- The extension is small: one field, one copy step, one consolidation rule, one entity extension, and tests.
- AlfaPeople can create Azure DevOps only in the sandbox tenant (`alfapeopleglobalsandbox`), not in the company tenant.
- The code will end up in **Honegger's Azure DevOps**, which uses **Git** (confirmed 07.11.2026). We don't yet know how Honegger builds and deploys its models (`HGExtension`).
- Microsoft's FinOps build and deploy tasks exist only for Azure DevOps. Microsoft-hosted builds don't run SysTest anyway.
- The previous Honegger project (d365-pma-indirectcosts) is also on GitHub.

## Decision

| Concern | Choice |
|---|---|
| Source of truth until handover | Private GitHub repo `d365-sba-activities`, Git, trunk-based ([C-03 §2](../conventions/03-alm-and-source-control.md#2-branching)) |
| Build and unit tests | On the dev VM in Visual Studio (build + BP check + SysTest). No GitHub Actions X++ pipeline. |
| CI, if we need it before handover | Azure Pipelines in the sandbox Azure DevOps org, building the GitHub repo. We don't write GitHub Actions YAML that would need rewriting. |
| Docs, decisions, test cases | Markdown in the repo |
| Deployable package for Honegger environments | Built by **Honegger's** pipeline after handover (the Implementation guide advises against packages from Visual Studio). If Honegger has no pipeline, we set one up in their Azure DevOps (OP-04). |
| Handover | *Import repository* into Honegger's Azure DevOps (Git) with full history. The GitHub repo is then archived. |

## Consequences

- No feature parity gap for what this project needs: Git, PRs, Markdown docs and VM-based tests work the same in both.
- The gaps are build/deploy tasks and Test Plans. We deliberately take those from Honegger's Azure DevOps at handover instead of rebuilding them in GitHub.
- Branch protection on a private repo needs GitHub Pro/Team. On GitHub Free, review discipline is by agreement only (OP-09).
- If Honegger requires Test Plans, the Markdown test cases are imported by CSV ([C-02 §3](../conventions/02-testing.md#3-functional-test-cases)).

## Rejected alternatives

| Alternative | Why not now |
|---|---|
| Azure DevOps in the sandbox tenant as source of truth | Not the final home either, so it would mean two migrations (sandbox org → Honegger org). It is still the right place for CI if CI is needed early. |
| GitHub Actions with FSC-PS | Community-maintained with uncertain maturity, and the YAML is lost at migration. |
| Start directly in Honegger's Azure DevOps | Best end state, but needs Honegger's access, branch model and approval first (OP-04, OP-11 of the PMA project). Reconsider as soon as that is available. |
