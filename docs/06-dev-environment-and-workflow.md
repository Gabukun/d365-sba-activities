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
5. Turn on best-practice checks for both models ([C-01 §7](conventions/d365-finops/01-naming-and-extensions.md#7-best-practice-checks)).

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
| Build and BP check | Gabriel in Visual Studio | Claude runs `xppc` and `xppbp` from the command line and fixes errors in a loop (commands: §5.1). Database sync stays in Visual Studio. |
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

### 4b. Starting a work session on the VM

1. Clone both repositories side by side, e.g. `C:\Repos\d365-sba-activities` and `C:\Repos\youniform`, and link the packages (§2).
2. Start Claude Code in `C:\Repos\d365-sba-activities`. It reads `CLAUDE.md`, which points to the conventions and to doc 02 §4 (discovery).
3. Work in a feature branch per topic (e.g. `feature/D-12-hgextension-overlap`). Results go into doc 02 / doc 08 and `docs/discovery/`. Push and open a PR, so the cloud session can review.

## 5. Build, check, test (each change)

1. Build `HOGsbaSubBillActivity` and `HOGsbaSubBillActivityTest` (*Dynamics 365 > Build models*, with *Synchronize database* when tables changed).
2. Zero BP deviations, or a justified suppression.
3. Run the SysTest classes (Test Explorer, category `HOGsba`).
4. Run the affected functional TCs (doc 07) and log the results.
5. Commit on a feature branch and open a PR. The PR text lists the doc IDs (FR, D, TC, DEC).

### 5.1 Command-line build and best-practice check (VM)

Run on 09.11.2026 on `dvhd10049vm`, tools version 7.0.8199.32 in `C:\AosService\PackagesLocalDirectory\Bin`. Switches are taken from `xppc.exe -?` and `xppbp.exe -?`. The `xppbp` switches also match the [CAR command on Microsoft Learn](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/lcs-solutions/code-migration-lcs-solutions). Microsoft Learn has no reference page for the `xppc` command line.

```powershell
# Run from any folder. Logs go to <repo>\logs (gitignored). $tag names the work package, e.g. WP-02.
$pld   = 'C:\AosService\PackagesLocalDirectory'
$model = 'HOGsbaSubBillActivity'
$logs  = 'C:\Repos\d365-sba-activities\logs'
$tag   = 'WP-02'
New-Item -ItemType Directory -Force $logs | Out-Null
Push-Location "$pld\Bin"

# 1. Compile the model (writes $pld\$model\bin, same output as Visual Studio)
& .\xppc.exe "-metadata=$pld" "-compilermetadata=$pld" "-modelmodule=$model" `
    "-output=$pld\$model\bin" "-referenceFolder=$pld" "-refPath=$pld\$model\bin" `
    "-log=$logs\$tag-build.log" "-xmlLog=$logs\$tag-build.xml"
"xppc exit code: $LASTEXITCODE"

# 2. Best-practice check, all rules, our model only
& .\xppbp.exe "-metadata=$pld" "-compilermetadata=$pld" "-packagesroot=$pld" `
    "-module=$model" "-model=$model" -all `
    "-log=$logs\$tag-bp.log" "-xmlLog=$logs\$tag-bp.xml"
"xppbp exit code: $LASTEXITCODE"

Pop-Location
```

Result on 09.11.2026 (WP-01 content, 1 element):

| Step | Duration | Errors | Warnings |
|---|---|---|---|
| `xppc` | 6 s | 0 | 1: `ExternalReference` / `AssemblyFileNotFound` for `dynamics://Reference/Microsoft.Dynamics___vices.PricingEngine` (missing assembly `Microsoft.Dynamics.Commerce.Runtime.Entities.AttributeBasedPricing`) |
| `xppbp -all` | 2 s | 0 | 0 |

The compile warning is **not from our model**. The reference element is `ApplicationSuite\Foundation\AxReference\Microsoft.Dynamics___vices.PricingEngine.xml` (Microsoft package). The Visual Studio build writes the same warning to `Metadata\HOGsbaSubBillActivity\BuildModelResult.log`. Treat it as known VM noise; any other warning counts.

What the command line does **not** cover:
- **Database sync.** After table or table extension changes, sync from Visual Studio (*Build models* with *Synchronize database*). `SyncEngine.exe` is in `Bin` but has not been tried.
- **Labels.** The model has no label file yet. Once it has one, `LabelC.exe` (in `Bin`) has to run before `xppc`. Command: VERIFY when the label file is added.
- **Cross-references.** `xppc` updates `DYNAMICSXREFDB` only with `-xref`; the command above leaves it out. Visual Studio updates it (project property *GenerateCrossReferences* = True).

Don't run `xppc` while Visual Studio builds the same model: both write to `$pld\$model\bin`.

### 5.2 Visual Studio project file (`.rnrproj`)

Solution `Projects\HOGsbaSubBillActivity\HOGsbaSubBillActivity.sln`, project `Projects\HOGsbaSubBillActivity\HOGsbaSubBillActivity\HOGsbaSubBillActivity.rnrproj`. Checked 09.11.2026:

- `<Model>HOGsbaSubBillActivity</Model>`. The solution shows the project as `HOGsbaSubBillActivity (USR) [HOGsbaSubBillActivity]`. A project created with the wrong model (it happened once with *FleetManagement*) cannot take our elements.
- No absolute or user-profile paths. Only `$(MSBuildProgramFiles32)`, `$(MSBuildBinPath)` and `$(BuildTasksDirectory)`.
- The project only **references** elements. The element XML stays in `Metadata\` (CLAUDE.md).

One `Content` item per element. Confirmed format (written by Visual Studio for WP-01):

```xml
<Content Include="AxTableExtension\SubBillScheduleLine.HOGsba">
  <SubType>Content</SubType>
  <Name>SubBillScheduleLine.HOGsba</Name>
  <Link>SubBillScheduleLine.HOGsba</Link>
</Content>
```

- `Include` = `<metadata type folder>\<element name>`, **without** `.xml`. The folder is the one the element has under `Metadata\HOGsbaSubBillActivity\HOGsbaSubBillActivity\` (`AxTableExtension`, `AxFormExtension`, `AxClass`, …).
- `Name` and `Link` = element name.
- Only `AxTableExtension` is confirmed so far. For other types, check the first entry of each type against what Visual Studio writes (VERIFY).
- Editing the `.rnrproj` by hand while the solution is open makes Visual Studio ask to reload the project. Tell the person who has Visual Studio open before changing it (one writer per file).

## 6. Packaging and deployment

Not on the VM for customer environments. The deployable package comes from a pipeline after handover (ADR-001, OP-04). For a quick technical check, Visual Studio *Create deployment package* can be used **locally only**.
