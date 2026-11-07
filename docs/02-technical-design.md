# 02 – Technical design

> **Status:** draft. Standard names marked **VERIFY** come from community sources or inference and must be confirmed on the dev VM (discovery, section 4) before we code against them. X++ in this document is a **draft and has not been compiled**.

## 1. Standard objects – what we know

| Concept | Name | Source | Status |
|---|---|---|---|
| Subscription billing package/model | `SubscriptionBilling` (display name *Subscription Billing* 10.40.3825.34730 in *About > Installed models*, 07.11.2026) | Microsoft [AppChecker sample rules](https://github.com/microsoft/Dynamics365FO-AppChecker), screenshot | Model confirmed; VERIFY folder name D-01 |
| Billing schedule header | `SubBillScheduleTable` | Microsoft KB title in the [10.0.32 quality update list](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/get-started/quality-updates-build-details-10-0-32) | Confirmed name, VERIFY fields |
| Billing schedule line | `SubBillScheduleLine`, key `SubBillBillingScheduleNumber` + `LineNum` | Community (blog 2025-07, OData metadata) | VERIFY D-01 |
| Project fields on the line | `ProjId` (control `SubBillScheduleLine_ProjId`); ProjCategoryId, ProjLinePropertyId, ProjTaxGroup, ProjTaxItemGroup | Form information (screenshot 07.11.2026) for ProjId; community OData metadata for the rest | ProjId confirmed; rest VERIFY D-01 |
| Generate invoice working table | `SubBillInvoiceParmLines` | [What's new 10.0.47](https://learn.microsoft.com/dynamics365/finance/get-started/whats-new-changed-10-0-47) | Confirmed name |
| Sales order creation | class `SubBillCreateSalesOrder`, methods `createSalesOrder`, `initSalesTable`, `createSalesLine(SalesLine, SubBillSalesLineConsolidated, SalesId, boolean ×4)` | [Community blog with CoC example, 2025-07-11](https://1yaowei0.github.io/2025/07/11/xpp-extendsubbillingschedule/) | **VERIFY D-03** (signature, visibility, insert timing) |
| Consolidation buffer | `SubBillSalesLineConsolidated` (fields `SubBillBillingScheduleNumber`, `LineNum`) | same blog | VERIFY D-05 |
| Back-references on SalesLine | `SubBillDetailLineRecId`, `SubBillTermScheduleLineRecId`, … | Community SQL schema export | VERIFY |
| SO line activity | `SalesLine.ActivityNumber`, EDT `ProjActivityNumber` | [CDM SalesLine](https://learn.microsoft.com/common-data-model/schema/core/operationscommon/tables/supplychain/salesandmarketing/worksheetline/salesline) | Confirmed |
| Activity master | `smmActivities` (relation of `ProjActivityNumber` in AX 2012) | [AX 2012 reference](https://learn.microsoft.com/previous-versions/dynamics/ax-2012/reference/gg857354(v=ax.60)) | VERIFY D-10 for D365 |
| Billing schedule line entity | OData public name `SubBillBillingScheduleLine`; AOT name unknown | Community | VERIFY D-08 |
| Billing schedule form | `SubBillScheduleTable`, line data source `SubBillScheduleLine`, line tab *Projekt* | Form information (screenshot 07.11.2026) | **Confirmed (D-02)** |
| SO line activity control | `SalesTable` form, control `Project_ActivityNumber` | Form information (screenshot 07.11.2026) | Confirmed; starting point for D-04 |
| Honegger extension models | `HGExtension` (usr, 2025.3.6.4); ISV *Axtension®* Platform / Invoice Processing / Content Gate | *About > Installed models* | Check for overlaps: D-12 |

## 2. Objects to create

Package **`HOGsbaSubBillActivity`**, solution prefix **`HOGsba`** (proposal, DEC-02). Naming per [C-01](conventions/01-naming-and-extensions.md).

| Type | Name | Purpose | Depends on |
|---|---|---|---|
| Descriptor | `HOGsbaSubBillActivity` | Model, layer usr. References: SubscriptionBilling, ApplicationSuite, ApplicationPlatform, ApplicationFoundation (VERIFY which package holds `ProjActivityNumber`/`smmActivities`) | D-01 |
| Table ext. | `SubBillScheduleLine.HOGsba` | Field **`HOGsbaActivityNumber`** (EDT `ProjActivityNumber`, standard label). Added to the line's project field group. Optional index if D-05 needs one. | D-01 |
| Class (CoC) | `SubBillScheduleLine_HOGsba_Extension` | `validateField`, `validateWrite` (FR-03, also for imports), `modifiedField` (FR-04) | D-04 |
| Form ext. | `SubBillScheduleTable.HOGsba` | Control `SubBillScheduleLine_HOGsbaActivityNumber` on the line's Project tab, next to Project ID | – |
| Class (CoC) | `SubBillScheduleTableFormSubBillScheduleLine_HOGsba_Extension` (data source) | Lookup (FR-02), enabled only when the line has a project. Reuse the standard lookup of `SalesLine.ActivityNumber`. | D-04 |
| Class (CoC) | `SubBillCreateSalesOrder_HOGsba_Extension` | Hook into sales order line creation (FR-05) | **D-03** |
| Class | `HOGsbaSalesLineActivity` | Copy rule: billing schedule line → `SalesLine.ActivityNumber` (design choices 4, 5) | D-04 |
| Class (CoC) | consolidation hook, name TBD | Activity becomes part of the consolidation key (FR-06) | **D-05** |
| Class (CoC) | copy-schedule hook, name TBD | Only if copy doesn't use `buf2Buf`/`data()` (FR-08) | D-06 |
| Class (CoC) | renewal hooks, names TBD | Keep the activity on renewed or added lines (FR-11); take it from the SO line in support and renewal (FR-12) | D-13 |
| Class (CoC) | termination credit hook, name TBD | Only if the credit path doesn't go through the D-03 hook (FR-13) | D-09 |
| Entity ext. | `<LineEntity>.HOGsba` + staging table ext. | Field `HOGsbaActivityNumber` (FR-07) | D-08 |
| Test package | `HOGsbaSubBillActivityTest` | SysTest classes (doc 07 §3) | – |
| Labels | none planned | Standard EDT label. Create a label file `HOGsba` only if a message needs new text. | D-04 |

## 3. Design choices

1. **Field name `HOGsbaActivityNumber`.** This is the standard name `ActivityNumber` with the mandatory extension prefix ([C-01 §2](conventions/01-naming-and-extensions.md#2-prefixes)). Microsoft is actively extending Subscription billing towards projects (project fee journals, project deferrals). If it adds `ActivityNumber` to the line one day, an unprefixed field would break the build.
2. **Same EDT as the sales order line** (`ProjActivityNumber`). Length, label, help text and relation are then identical, and the copy is a plain assignment.
3. **Reuse the standard lookup and validation** that the sales order line uses for its Activity number (D-04). We don't write our own query over the WBS. This keeps behaviour identical (closed activities, project hierarchy) and survives service updates.
4. **Copy as late and as narrowly as possible.** Prefer a hook where the sales order line is initialised **before insert**. Avoid updating an already inserted line (an extra database round trip, and it re-runs `SalesLine.update` logic). Final choice after D-03.
5. **Copy only when the project matches.** Set `SalesLine.ActivityNumber` only if the sales order line has the same `ProjId` as the billing schedule line. Otherwise do nothing and log a warning. Never fail invoice generation because of the activity.
6. **Consolidation.** If *Consolidate by item* or *Consolidate all periods* can merge several billing schedule lines into one sales order line, the activity must be part of the comparison, just like Project already is ([sb-bill-sched-project](https://learn.microsoft.com/dynamics365/finance/accounts-receivable/sb-bill-sched-project): consolidation compares Project). D-05 shows where that comparison happens.
7. **Standard side effects.** If standard runs logic when `SalesLine.ActivityNumber` changes (for example defaulting dimensions), our copy must trigger it the same way standard does when it sets the field (D-04).

## 4. Discovery (on the dev VM, before coding)

Send the result as text (call stacks, short source excerpts). **Don't paste whole Microsoft source files into the repo.**

| ID | Question | How |
|---|---|---|
| **D-01** | Package, tables and fields of Subscription billing | PowerShell below. Then in Application Explorer: `SubBillScheduleLine` → fields and field groups of the Project tab. |
| **D-02** | Billing schedule form, line data source, Project tab group | **Done 07.11.2026:** form `SubBillScheduleTable`, control `SubBillScheduleLine_ProjId`. Remaining: name of the Project tab group (Visual Studio, form design). |
| **D-03** | Where is `SalesLine` created and inserted by *Generate invoice*? | Attach the debugger to the web process (`w3wp.exe`), set a breakpoint in `SalesLine.insert()`, run *Generate invoice* → *Create sales order* for one line. Send the **full call stack** and the signature and visibility of the method that fills the new `SalesLine` before insert. |
| **D-04** | Standard lookup, validation and side effects of `SalesLine.ActivityNumber` | Application Explorer: `SalesLine` → `validateField`, `modifiedField` (case `ActivityNumber`). `SalesTable` form → control `Project_ActivityNumber` → its lookup. Send the relevant lines and the names of the standard methods they call. |
| **D-05** | Consolidation: where are lines grouped into one sales order line, and which fields are compared? | Same call stack as D-03 with *Consolidate by item* = Yes and two lines of the same item. Send the method that builds `SubBillSalesLineConsolidated`. |
| **D-06** | How does *Copy schedule* copy lines? | Breakpoint in `SubBillScheduleLine.insert()`, run *Copy schedule*. Field-by-field or `buf2Buf`? |
| **D-07** | **Business value check, no code needed:** does a manually entered `SalesLine.ActivityNumber` reach the posted project transactions? | Billing schedule with project → *Generate invoice* (Create sales order) → set Activity number on the SO line by hand → project invoice proposal → post. Check the posted project transactions (and the voucher dimensions) for the activity. **Do this first.** |
| **D-08** | Billing schedule line data entity: AOT name and staging table | Data management → entity list → filter *billing schedule*. Application Explorer: entity → *Staging table* property. |
| **D-09** | Other paths that create sales order lines: customer split, termination *Issue credit* | Repeat the D-03 breakpoint for both. |
| **D-10** | `ProjActivityNumber` in D365: relation, string size, configuration key | Application Explorer: EDT `ProjActivityNumber` → properties and relations |
| **D-11** | Does the preview feature *Database Query Optimization and Reduced Chattiness* (Subscription billing) change the D-03 path? | Only if hog-uat has it on or will: repeat D-03 with the feature on. |
| **D-12** | Does Honegger's `HGExtension` (or an Axtension model) already extend `SubBillScheduleLine`, `SubBillCreateSalesOrder` or `SalesLine.ActivityNumber`? Which prefix do HGExtension objects use? | PowerShell below (second block), on a VM that has Honegger's models. |
| **D-13** | Renewal paths: *Renew automatically*, *Add renewal term*, *Lines to add per renewal*, and support and renewal from a posted SO invoice. Which methods create or copy the lines? | Breakpoint in `SubBillScheduleLine.insert()` for each path; send the call stacks. |

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
} | Sort-Object Type, Name | Export-Csv "$env:USERPROFILE\Desktop\D-01-subbill-objects.csv" -NoTypeInformation -Encoding UTF8
```

The CSV contains object names only (no source code) and can go into `docs/discovery/`.

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

### 5.1 Billing schedule line: validation and reset

```xpp
/// <summary>
/// Activity number on billing schedule lines: validation and reset.
/// DRAFT – not compiled. VERIFY D-01 (field ProjId on SubBillScheduleLine), D-04 (standard check).
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

        if (_fieldId == fieldNum(SubBillScheduleLine, ProjId))     // VERIFY D-01
        {
            this.HOGsbaActivityNumber = '';                            // FR-04
        }
    }

    /// <summary>Activity must be empty or belong to the line's project (FR-03).</summary>
    public boolean hogsbaValidateActivity()
    {
        if (!this.HOGsbaActivityNumber)
        {
            return true;
        }
        if (!this.ProjId)                                           // VERIFY D-01
        {
            return checkFailed("@SYS...");                          // VERIFY D-04: reuse the standard label
        }
        // VERIFY D-04: call the same standard check that SalesLine.validateField uses
        // for ActivityNumber with (ProjId, ActivityNumber). No own WBS query.
        return true;
    }
}
```

### 5.2 Copy to the sales order line

```xpp
/// <summary>
/// Copies the project activity from the billing schedule line to the generated sales order line.
/// DRAFT – not compiled. Hook point, signature and insert timing: VERIFY D-03.
/// If SalesLine is already inserted when createSalesLine returns, move the copy to the
/// method that initialises SalesLine before insert (design choice 4).
/// </summary>
[ExtensionOf(classStr(SubBillCreateSalesOrder))]
final class SubBillCreateSalesOrder_HOGsba_Extension
{
    // Signature copied from a community source – VERIFY D-03.
    public SalesLine createSalesLine(
        SalesLine                       _salesLine,
        SubBillSalesLineConsolidated    _salesLineConsolidated,
        SalesId                         _salesId,
        boolean                         _allocEnabled,
        boolean                         _splitByItemGroup,
        boolean                         _isInvoiceCreator,
        boolean                         _includeBillingDatesToItem)
    {
        SalesLine salesLine = next createSalesLine(
            _salesLine, _salesLineConsolidated, _salesId,
            _allocEnabled, _splitByItemGroup, _isInvoiceCreator, _includeBillingDatesToItem);

        SubBillScheduleLine scheduleLine = SubBillScheduleLine::find(
            _salesLineConsolidated.SubBillBillingScheduleNumber,
            _salesLineConsolidated.LineNum);                        // VERIFY D-05: one source line per SO line?

        HOGsbaSalesLineActivity::apply(salesLine, scheduleLine);

        return salesLine;
    }
}
```

```xpp
/// <summary>
/// Copies the activity of a billing schedule line to a sales order line.
/// Plain class (not an extension class) so it can be unit tested without the generate-invoice process.
/// DRAFT – not compiled.
/// </summary>
final class HOGsbaSalesLineActivity
{
    /// <summary>Sets the activity on the sales order line if the project matches (design choice 5).</summary>
    public static void apply(SalesLine _salesLine, SubBillScheduleLine _scheduleLine)
    {
        if (!_scheduleLine.HOGsbaActivityNumber
            || _salesLine.ActivityNumber == _scheduleLine.HOGsbaActivityNumber)
        {
            return;
        }
        if (_salesLine.ProjId != _scheduleLine.ProjId)               // VERIFY D-01
        {
            warning(strFmt("@SYS...", _scheduleLine.HOGsbaActivityNumber)); // VERIFY label
            return;
        }
        _salesLine.ActivityNumber = _scheduleLine.HOGsbaActivityNumber;
        // VERIFY D-04: does standard call modifiedField(ActivityNumber) for side effects?
        // VERIFY D-03: if _salesLine is already inserted, an update is needed here
        //              (prefer moving the copy before insert instead).
    }
}
```

`HOGsbaSalesLineActivity::apply` is the unit under test in doc 07 (TC-U03 to TC-U05). It doesn't depend on the generate-invoice process.

## 6. Performance

- `SubBillScheduleLine::find` adds one read per generated sales order line, on the primary key. That is acceptable at Honegger's volumes (OP-01). If D-03 shows that the source line buffer is already available in the hook, use it and skip the find.
- No new loops, no set-based operations, no new indexes on `SalesLine`.

## 7. Error handling

- Validation errors (FR-03) stop saving the billing schedule line, as standard field validation does.
- During invoice generation, a mismatch (design choice 5) **never** stops generation. It produces a warning in the infolog/batch log, and the sales order line is created without an activity.
