# 06 – Dev environment and working model

## 1. Who does what, where

| Place | What happens there | Who |
|---|---|---|
| **GitHub** `d365-sba-activities` (private) | Source of truth until handover: metadata, docs, decisions, test cases ([ADR-001](adr/ADR-001-github-now-azure-devops-at-handover.md)) | Both |
| **Claude cloud session** | Research, designs, X++ drafts, reviews, comparing exports, writing docs | Claude |
| **Dev VM** `dvhd10049vm`, F&O environment `dvhdm100049dm` (Ycompany Entra tenant, reachable over Tailscale only) | Discovery (D-xx), build, BP checks, SysTest, functional tests | Gabriel, or Claude when it runs on the VM (§4) |
| **hog-uat** (Honegger sandbox) | Feature and setup export (DMF), acceptance tests, Task recordings | Gabriel, Honegger |
| **Honegger Azure DevOps** | Build, deployable package, deployment (after handover) | Per OP-04 |

The cloud session **can't reach the VM**: the VM is only on the tailnet, and a connection attempt from the session was refused. Results from the VM come back as text or files in the repo, or from Claude running on the VM itself.

## 2. Git on the VM (VHD with PackagesLocalDirectory)

Method from the [Microsoft ISE blog](https://devblogs.microsoft.com/ise/xpp-and-git/): our package folders live in the repo and are linked into `PackagesLocalDirectory`.

```powershell
# Run in an elevated PowerShell on the VM. Close Visual Studio first.
$repo = 'C:\Repos\d365-sba-activities'          # adjust to your clone path
git clone https://github.com/Gabukun/d365-sba-activities.git $repo
```

```powershell
# Link every package folder under <repo>\Metadata into PackagesLocalDirectory
$repo = 'C:\Repos\d365-sba-activities'
$pld  = @('C:\AosService\PackagesLocalDirectory', 'K:\AosService\PackagesLocalDirectory') |
    Where-Object { Test-Path $_ } | Select-Object -First 1

Get-Service -Name W3SVC, DynamicsAxBatch -ErrorAction SilentlyContinue | Stop-Service -Force

Get-ChildItem (Join-Path $repo 'Metadata') -Directory | ForEach-Object {
    $link = Join-Path $pld $_.Name
    if (Test-Path $link) { Write-Warning "$link already exists - skipped (move it into the repo first)"; return }
    New-Item -ItemType SymbolicLink -Path $link -Target $_.FullName | Out-Null
    "Linked $link -> $($_.FullName)"
}

Get-Service -Name W3SVC, DynamicsAxBatch -ErrorAction SilentlyContinue | Start-Service
```

## 3. Create the model (once)

1. Visual Studio > Dynamics 365 > Model management > **Create model**: name `HOGsbaSubBillActivity`, publisher *AlfaPeople Switzerland*, layer `usr`, **Create new package**. References: per doc 02 §2 (after D-01).
2. Visual Studio creates it inside `PackagesLocalDirectory`. Close Visual Studio, stop the services, **move** the folder to `<repo>\Metadata\HOGsbaSubBillActivity`, run the link script, then start the services again.
3. Repeat for the test package `HOGsbaSubBillActivityTest` (references: `HOGsbaSubBillActivity`, `TestEssentials`, ATL packages).
4. Create the Visual Studio projects under `<repo>\Projects\` (not inside `Metadata`).
5. Turn on best-practice checks for both models ([C-01 §7](conventions/01-naming-and-extensions.md#7-best-practice-checks)).

## 4. Giving Claude access to F&O

| Need | Route | Status |
|---|---|---|
| **Code and metadata on the VM**: standard object names, CoC signatures, build, SysTest | Run Claude **on the VM**: Claude Code in the repo folder, or `claude remote-control` there so the session shows up in the Claude app. Reads `PackagesLocalDirectory` directly and runs PowerShell, builds and debugger-free discovery. | **Recommended.** No Entra app needed. |
| **Honegger data and configuration** (hog-uat) | **Dynamics 365 ERP MCP server**. Requires version ≥ 10.0.47 (or 10.0.46 PQU-2 / 10.0.45 PQU-7) and Tier 2 or UDE ([prerequisites](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/copilot/copilot-mcp#prerequisites)). Claude is not a default client: Honegger registers an Entra app and adds its client ID under *Allowed MCP clients* ([security](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/copilot/mcp/mcp-security)). The server acts **as the signed-in user** with that user's roles, so use a dedicated user with a read-only role. Then add it at claude.ai → Connectors and start a new session. | Possible. Claude's connector sign-in with this server is **not documented by Microsoft**, so test it first (OP-10). |
| Data on **our VM** through MCP | Not supported: the MCP server needs Tier 2 or UDE, not a VHD/cloud-hosted dev box | – |
| Reusing the VHD's sign-in app (`D365FO-VHD-FINOPS-DEV01`) as the MCP client | That app registration is the VM's own sign-in, used by the *Generate Self-Signed Certificates* script, with redirect URIs on `usnconeboxax1aos.cloud.onebox.dynamics.com` ([VHD setup](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/dev-tools/vhd-setup)). | **No.** Changing it risks breaking VM sign-in. One identity for two purposes can't be revoked or audited separately. An MCP client always gets its own app registration. |
| Data on our VM through OData + Entra app (Ycompany) | Technically possible, but the session can't reach the VM. It would need a public tunnel to a dev AOS plus an allowed host in the session's network policy. | **Not recommended**: security exposure for little gain over §4 row 1 |

What MCP does **not** give us: AOT metadata and source (it exposes data, forms and actions), and Feature management (excluded form).

### 4a. What changes when Claude runs on the VM

| Step | Today (cloud session) | With Claude Code on the VM |
|---|---|---|
| Discovery D-01 to D-13 | Gabriel runs each step and pastes the result | Claude reads the standard metadata in `PackagesLocalDirectory` itself (tables, classes, forms, entities) and follows call chains in the code. Runtime proof (which path really runs) still needs one debugger session or a temporary trace; Claude doesn't drive the Visual Studio debugger. |
| `VERIFY` markers | Stay until you confirm | Mostly resolved by Claude directly in the source |
| Writing objects | X++ drafts in Markdown | Claude writes the metadata files in `Metadata/` (table, form and entity extensions, classes) |
| Build and BP check | Gabriel in Visual Studio | Claude runs the build and `xppbp` from the command line and fixes errors in a loop (exact build command: VERIFY on the VM) |
| SysTest | Gabriel in Test Explorer | Same, unless a command-line test runner works on the VM (VERIFY) |
| Functional tests in the browser | Gabriel | Gabriel. Claude doesn't click through the F&O client. |
| Commit, push, PR | Claude in the cloud session | Claude on the VM (needs Git credentials for GitHub on the VM) |
| Research (Microsoft Learn) | Claude, via the claude.ai connector | Also on the VM: claude.ai connectors are available when you sign in with the claude.ai account ([MCP docs](https://code.claude.com/docs/en/mcp)) |

**Requirements and setup (from the [Claude Code docs](https://code.claude.com/docs/en/setup), checked 07.11.2026):**
- **Supported OS:** Windows Server 2019 and later, natively, without WSL.
- **Install** from PowerShell, no administrator rights needed:
  ```powershell
  irm https://claude.ai/install.ps1 | iex
  ```
  Then, in a new terminal, check with `claude --version`.
- **Git for Windows** is optional (recommended, for the Bash tool). Without it, commands run in PowerShell.
- **Outbound HTTPS** to `api.anthropic.com`, `claude.ai`, `claude.com`, `platform.claude.com`, `downloads.claude.ai` (installer and updates) and `mcp-proxy.anthropic.com` (claude.ai connectors) ([network](https://code.claude.com/docs/en/network-config)).
- **`claude remote-control`** makes the VM session usable from claude.ai/code and the mobile app. It uses outbound connections only, with no inbound ports ([remote control](https://code.claude.com/docs/en/remote-control)).
  - It needs a Pro, Max, Team or Enterprise plan. On Team/Enterprise, an owner must enable it.
  - The docs don't state Windows support for this command, so test it.

**Guardrails:**
- Start in the default (ask) permission mode.
- Commit shared deny rules to `.claude/settings.json`, for example `PowerShell(Remove-Item *)` ([permissions](https://code.claude.com/docs/en/permissions)).
- Native Windows has **no sandbox**, so permission rules are not a security boundary. Run Claude as a normal user, not a domain admin, and keep secrets off the VM.
- If the VM database holds a copy of Honegger's data, Claude can read it through SQL. Honegger must agree, or use a test company without customer data (OP-16).
- Microsoft standard source code is read on the VM as context and is never committed (CLAUDE.md).

## 5. Build, check, test (each change)

1. Build `HOGsbaSubBillActivity` and `HOGsbaSubBillActivityTest` (*Dynamics 365 > Build models*, with *Synchronize database* when tables changed).
2. Zero BP deviations, or a justified suppression.
3. Run the SysTest classes (Test Explorer, category `HOGsba`).
4. Run the affected functional TCs (doc 07) and log the results.
5. Commit on a feature branch and open a PR. The PR text lists the doc IDs (FR, D, TC, DEC).

## 6. Packaging and deployment

Not on the VM for customer environments. The deployable package comes from a pipeline after handover (ADR-001, OP-04). For a quick technical check, Visual Studio *Create deployment package* can be used **locally only**.
