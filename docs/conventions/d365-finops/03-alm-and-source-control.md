# C-03 – ALM and source control (FinOps extensions)

> **Snapshot.** Master copy: [youniform `conventions/d365-finops/03-alm-and-source-control.md`](https://github.com/youserpath/youniform/blob/main/conventions/d365-finops/03-alm-and-source-control.md), snapshot of 07.11.2026. Don't edit here; change youniform and refresh this copy (DEC-11).

Reusable across all D365 Finance & Operations extension projects. Customer-specific values (prefix, existing models, publisher, environments) are in `customers/<customer>.md`. Examples use Honegger (`HOG`). Researched 2026-11-07. Tags as in [C-01](01-naming-and-extensions.md). Each project records its own ALM decision as an ADR (template: `templates/d365-finops-extension/docs/adr/ADR-000-template.md` in youniform; example: d365-sba-activities ADR-001).

## 1. Version control: Git

| Fact | Source |
|---|---|
| Git is Microsoft's default and recommended version control. TFVC is feature complete and gets no new investment. | [Git vs TFVC](https://learn.microsoft.com/azure/devops/repos/tfvc/comparison-git-tfvc) |
| New Azure DevOps projects have TFVC creation disabled by default. | [No TFVC in new projects](https://learn.microsoft.com/azure/devops/release-notes/roadmap/2024/no-tfvc-in-new-projects) |
| X++ works in Git. The *Dynamics 365 Finance and Operations Tools* (build tasks) are "offered exclusively as an Azure DevOps extension", so Microsoft suggests Azure DevOps for full CI/CD. | [X++ in Git](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/dev-tools/git-intro) |
| On a VHD or cloud-hosted VM, map Git to `PackagesLocalDirectory` with symbolic links: each own package folder lives in the repo and is linked into `PackagesLocalDirectory`. Stop the web server (IIS) and batch service first. | [Microsoft ISE blog: X++ and Git](https://devblogs.microsoft.com/ise/xpp-and-git/) (2022) |
| In a UDE, Visual Studio has a separate *folder for your own custom metadata*. The repo can be that folder; no links needed (inference). | [UDE tools setup](https://learn.microsoft.com/power-platform/developer/unified-experience/finance-operations-install-config-tools) |

**"Azure DevOps or plain Git?"** Azure Repos *is* Git. Both GitHub and Azure DevOps host the same Git repository. Only Azure DevOps has Microsoft's FinOps build/deploy tasks and Test Plans. Source control, pull requests, docs and code review work the same in both.

**Repository layout:** see [C-01 §8](01-naming-and-extensions.md#8-repository-layout-metadata). `.gitignore` entries for X++ build output come from community practice ([atomicax, 2019](http://www.atomicax.com/article/giting-started-git-d365fo-and-other-puns)); Microsoft publishes no template.

**Pitfalls (community):**
- A file that is already tracked stays tracked even after it is added to `.gitignore`.
- Visual Studio creates new models inside `PackagesLocalDirectory`. Move them into the repo and re-link them.
- Keep Visual Studio project files inside the repo (`Projects/`).

## 2. Branching

- Microsoft's criteria: isolate in-development, in-test and production code; lock the production branch; require PR reviewers. Git works with trunk-based development or GitFlow. Keep branches short-lived and squash-merge ([Branching for FinOps](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/dev-tools/branching)). **[MS]**
- The Implementation guide uses **Dev / Main / Release** branches, says not to create deployable packages from Visual Studio, and to "keep service updates on the build machine less than or equal to production" ([ALM guidance](https://learn.microsoft.com/dynamics365/guidance/implementation-guide/application-lifecycle-management-product)). **[MS]**
- **[Team]** Our repos (small extensions, few developers) use **trunk-based** branching:
  - `main` is always buildable.
  - Short-lived `feature/<id>-<topic>` branches are merged by pull request with squash.
  - A tag `vX.Y.Z` marks each version that is delivered to the customer.
  - When a repo moves into the customer's Azure DevOps, it follows the customer's branch model.

## 3. Build and package

| Fact | Source |
|---|---|
| Microsoft-hosted builds on Azure Pipelines use NuGet packages: `Microsoft.Dynamics.AX.Platform.CompilerPackage`, `…Platform.DevALM.BuildXpp`, `…Application1.DevALM.BuildXpp`, `…Application2.DevALM.BuildXpp`, `…ApplicationSuite.DevALM.BuildXpp`. | [Hosted build automation](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/dev-tools/hosted-build-automation), [PPAC build pipeline tutorial](https://learn.microsoft.com/power-platform/admin/unified-experience/tutorial-build-pipeline-azure-devops) |
| You get the packages from the LCS Shared Asset Library, or from the Power Platform Visual Studio extension (*Download Dynamics365 FnO NuGets for CI/CD*). | same, and [UDE pipelines](https://learn.microsoft.com/power-platform/developer/unified-experience/finance-operations-pipelines) |
| Hosted builds do compilation and packaging only. They do **not** run SysTest and do not synchronize the database. | [Hosted build automation](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/dev-tools/hosted-build-automation) |
| The Azure DevOps extension *Dynamics 365 Finance and Operations Tools* provides these tasks: Create Deployable Package (v3 also creates the Power Platform unified package), Add Licenses, Update Model Version, and LCS upload/deploy. | [Marketplace](https://marketplace.visualstudio.com/items?itemName=Dyn365FinOps.dynamics365-finops-tools) |
| Microsoft's sample pipeline (`xpp-ci.yml`) is a starting point. It is outdated (`windows-2019`, VS 2019); update it to a current image and VS 2022. | [Dynamics365-Xpp-Samples-Tools](https://github.com/microsoft/Dynamics365-Xpp-Samples-Tools/tree/master/CI-CD/Pipeline-Samples) |
| PPAC-managed environments are deployed with `PowerPlatformDeployPackage@2`. It needs a Windows agent; deployments often run for more than 60 minutes. | [Release pipeline tutorial](https://learn.microsoft.com/power-platform/admin/unified-experience/tutorial-release-pipeline-azure-devops) |
| **GitHub Actions:** Microsoft publishes no X++ build action. Microsoft components that could be combined: `pac package compile --package-type erp` (preview) and the Power Platform `deploy-package` action (not documented for FinOps). | [pac package](https://learn.microsoft.com/power-platform/developer/cli/reference/package), [Power Platform GitHub Actions](https://learn.microsoft.com/power-platform/alm/devops-github-available-actions) |
| Community options for GitHub include **FSC-PS** (GitHub templates and actions; maturity uncertain) and **d365fo.tools** (PowerShell, version 0.7.36 published 2026-10-20). | [FSC-PS](https://github.com/fscpscollaborative/fscps), [d365fo.tools](https://www.powershellgallery.com/packages/d365fo.tools) |

**Version rule [Team, from the Implementation guide]:** build with the **same or a lower** version than the target environment. Microsoft's pages conflict on the NuGet version (the hosted-build page says "later than or equal to", the newer PPAC tutorial says "equal to or lower than"). We follow the Implementation guide and the newer tutorial.

## 4. Platform status (Nov 2026)

| Fact | Source |
|---|---|
| No new LCS projects for new customers since 2026-02-16. Existing projects continue. | [LCS project creation freeze](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/lifecycle-services/lcs-project-creation-freeze) |
| LCS → PPAC environment migration is in public preview (Sept 2026) and one-way. Cloud-hosted dev environments can't be migrated; use a UDE or a downloadable VHD instead. | [Migrate LCS environments](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/lifecycle-services/migrate-lifecycle-services-environments-power-platform-admin-center) |
| A UDE is a Power Platform sandbox with one AOS. It needs a qualifying license and at least 1 GB of free Operations **and** Dataverse capacity, and can't be provisioned at a chosen version. | [UDE types](https://learn.microsoft.com/power-platform/admin/unified-experience/unified-environment-types-and-templates), [UDE FAQ](https://learn.microsoft.com/power-platform/developer/unified-experience/finance-operations-faq) |
| VHDs are published twice a year, after the April and October releases. | [Version and update policy](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/migration-upgrade/versions-update-policy) |
| 10.0.49: GA 2026-09-11, end of service 2027-05-21. 10.0.50: GA planned for 2026-12-22. | [Service update availability](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/get-started/public-preview-releases) |

## 5. GitHub vs Azure DevOps: feature parity and migration

| Capability | GitHub | Azure DevOps | Effort to move into the customer's Azure DevOps |
|---|---|---|---|
| Git repository | Yes, full history | Yes (Git; TFVC off by default) | **Low.** *Import repository* keeps the full history. Authenticate with a GitHub PAT; LFS objects are not imported ([Import Git repo](https://learn.microsoft.com/azure/devops/repos/git/import-git-repository)). **High if the target is TFVC:** no supported Git→TFVC history migration, so the result is a snapshot check-in and the history stays archived in GitHub. |
| Pull requests and branch protection | Protected branches on private repos need GitHub Pro/Team ([plans](https://docs.github.com/en/get-started/learning-about-github/githubs-plans)) | Branch policies | Policies are recreated by hand. PR discussions are not Git objects and stay behind (inference). |
| X++ CI build | No Microsoft action; community (FSC-PS) or custom scripts. Alternatively Azure Pipelines can build a GitHub repo ([Build GitHub repos](https://learn.microsoft.com/azure/devops/pipelines/repos/github)). | Microsoft-documented | Rewrite the YAML; no converter in this direction (GitHub Actions Importer only converts *to* GitHub). |
| Deployable / unified package | Scripted | *Create Deployable Package* task | Use the customer's pipeline. |
| Deploy to LCS / PPAC | Community (d365fo.tools) or the Power Platform action (FinOps not documented) | LCS tasks; `PowerPlatformDeployPackage@2` | The customer's service connections. |
| Issues / boards | Issues, Projects | Boards | **Medium.** CSV import, at most 1,000 items per file; history and comments are lost ([CSV import](https://learn.microsoft.com/azure/devops/boards/queries/import-work-items-from-csv)). |
| Docs / wiki | Markdown in the repo | Markdown in the repo; *Publish code as wiki* | **Low.** Keep docs in the repo, not in a separate wiki. |
| Test plans / RSAT | None found | Test Plans (authoring needs Basic + Test Plans); RSAT unsupported after 2027-05-15 | Markdown test cases → CSV import ([C-02 §3](02-testing.md)). |
| X++ unit tests | Run on the dev VM | Run on the dev VM, or UDE pipeline task | Same in both. |

**Azure DevOps licensing:**
- Basic: the first 5 users are free. Stakeholder: free.
- Basic + Test Plans: $52 per user per month. The pricing page shows contradictory figures, so confirm before quoting.
- Free tier also includes 1 Microsoft-hosted job (1,800 min/month) and 2 GiB of Artifacts storage. The FinOps NuGet packages may exceed that (inference).
- Sources: [access levels](https://learn.microsoft.com/azure/devops/organizations/security/access-levels), [pricing](https://azure.microsoft.com/pricing/details/devops/azure-devops-services/).

## 6. Rules that keep the later migration cheap [Team]

1. **Everything in the repo:** docs, decisions, test cases and evidence. No GitHub-only wiki, no information that lives only in issues or Projects.
2. **Microsoft's layout:** `Metadata/<Package>/…` and `Projects/…`, as Azure Pipelines expects.
3. **No hard-wired CI** until the customer's pipeline is known. If CI is needed earlier, use Azure Pipelines (it can build a GitHub repo), not GitHub Actions. That avoids rewriting YAML later.
4. Commit messages and PR titles reference doc IDs (`TC-05`, `D-03`, `DEC-04`), not GitHub issue numbers, so the references survive the move.
5. Never commit secrets, customer production data or Microsoft source code.
