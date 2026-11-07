# C-01 – Naming and extension conventions (X++, FinOps)

Reusable across Honegger (HOG) FinOps extension projects. Researched 2026-11-07.

**Tags:** **[MS]** = rule from Microsoft Learn or an official Microsoft repository (link given). **[Team]** = our decision on top of the Microsoft rules. **[Community]** = observed practice in well-known public repositories.

## 1. Packages and models

| Rule | Tag | Source |
|---|---|---|
| One **new package** per solution (Visual Studio > Model management > Create model > *Create new package*). Never a new model inside a Microsoft package; overlayering is not supported. | MS | [Models and packages](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/dev-tools/models) |
| Layer `usr`. | MS | same |
| Put components with a different release cadence into separate packages. | MS | [Continuous delivery FAQ](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/dev-tools/continuous-delivery-faq) |
| Test code in its own package: the package name + `Test` (for example `HOGsbaSubBillActivityTest`). The word `Test` is required for test discovery. The package is excluded from the deployable package. | MS | [Testing and validations](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/perf-test/testing-validation), [Exclude test packages](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/dev-tools/exclude-test-packages) |
| Package name: see §2 (solution prefix + standard module prefix + topic, e.g. `HOGsbaSubBillActivity`). | Team | |
| Check the customer's **existing** extension packages (Honegger: `HGExtension`, usr layer) before choosing prefixes, and before wrapping a method they may already wrap. | Team | Discovery D-12 |
| Publisher: *AlfaPeople Switzerland* until code ownership with Honegger is agreed. | Team | |

## 2. Prefixes

| Scope | Pattern | Example | Tag |
|---|---|---|---|
| Customer prefix | `HOG` | | Team |
| Solution prefix: customer prefix + **lower-case** module acronym | `HOG<fff>` | `HOGsba` (SBA = Subscription billing, d365-sba-activities). d365-pma-indirectcosts still uses `HOGSoc`; align it when the conventions move to *youniform*. | Team (decided 07.11.2026) |
| Package / model | Solution prefix + standard module prefix + topic | `HOGsbaSubBillActivity`, test package `HOGsbaSubBillActivityTest` | Team (decided 07.11.2026) |
| **New elements** (tables, EDTs, enums, classes, forms, menu items, privileges, label file) | `HOG<fff><Name>` | `HOGsbaSalesLineActivity` | MS: "include a prefix that's specific to your solution in every element name" ([Naming guidelines for extensions](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/extensibility/naming-guidelines-extensions)) |
| **Fields, field groups, indexes, relations** added to a standard element | Solution prefix + the **standard field name** of the same concept | `HOGsbaActivityNumber` | MS: "Include a prefix, term, or abbreviation at the beginning of the name of the metadata node", because the name must be unique across the base element and all its extensions. Team: we reuse the standard name after the prefix ("as close to standard as possible"). |
| **Form controls** added in a form extension | Visual Studio default `<DataSource>_<Field>` | `SubBillScheduleLine_HOGsbaActivityNumber` | Team. Follows the standard control pattern (e.g. `SubBillScheduleLine_ProjId`), and it is unique because the field carries the prefix. If a BP check asks for the prefix at the beginning, rename to `HOGsba<DataSource>_<Field>`. |
| New methods and variables in extension classes | lower-case prefix + name | `hogsbaCopyActivity()` | MS: prefix when uniqueness is not certain. Avoid generic names such as *Approver, Delay, Group, Lookup, Process*, which Microsoft says are likely to collide. |

> **Why not just `ActivityNumber`?** If Microsoft later adds `ActivityNumber` to the same standard table, our extension no longer compiles. The prefix costs nothing and removes that risk. Microsoft has been extending Subscription billing towards projects (see doc 02), so the risk is real here.

> **Field name vs control name.** *Form information* in the client (right-click a field) shows the **form name** and the **control name**, not the table field.
> - `Project_ActivityNumber` on `SalesTable` means: group *Project*, field `ActivityNumber`.
> - `SubBillScheduleLine_ProjId` on `SubBillScheduleTable` means: data source `SubBillScheduleLine`, field `ProjId`.
>
> Table fields never carry these data-source or group parts. To see the real field name, use the control's *Data field* property in Visual Studio, or *Show details* in the client.

## 3. Extension elements and extension classes

| Item | Pattern | Example | Tag / source |
|---|---|---|---|
| Table, form, EDT, enum, data entity and menu **extension element** | `<StandardElement>.<Prefix or model>` | `SalesLine.HOGsba` | MS: the prefix goes in as an *infix*; never `SalesLine.Extension`. Visual Studio defaults to the model name. |
| **Extension class** (CoC) | `<StandardType>_<Prefix>_Extension` | `SalesLine_HOGsba_Extension` | MS: start with the augmented type, end with `_Extension`, include the prefix ([Naming guidelines](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/extensibility/naming-guidelines-extensions), [Class extensions](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/extensibility/class-extensions)). Team: the underscore form, as in d365-pma-indirectcosts. Microsoft's own FastTrack code uses it too (`SysFindTrace_SABatchTracing_Extension`). |
| Form, form data source, form control | `<Form>Form_<Prefix>_Extension`, `<Form>Form<DataSource>_<Prefix>_Extension` | `SalesTableForm_HOGsba_Extension` | MS: add the element type for forms and nested elements; one class per form, data source, data field or control. |
| Label file of our own | `HOG<Fff>` | `HOGsba` | Community (FastTrack samples) |
| Changing someone else's labels | `<LabelFileId>_Extension` | | MS ([Customization: overlayering and extensions](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/extensibility/customization-overlayering-extensions)) |

Microsoft's [archive-custom-fields article](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/sysadmin/archive-custom-add-fields) uses `SalesTable_Contoso_Extension` for a *table extension element*. That contradicts the naming guidelines; follow the naming guidelines.

## 4. Data types and labels

| Rule | Tag |
|---|---|
| A new field that represents a standard concept uses the **same EDT** as the standard field (for example the EDT of `SalesLine.ActivityNumber`). This keeps lookups, relations, labels, help text and string length identical. | Team |
| Reuse the standard label of that EDT. Create our own labels only for new concepts. | Team (Microsoft has no rule either way) |
| An EDT extension can change only form help, label, string size and help text. | MS ([Customization: overlayering and extensions](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/extensibility/customization-overlayering-extensions)) |
| New EDTs extend a standard EDT where one exists. | Community |
| Label languages: en-US and de (de-CH only if the tenant uses it). | Team (precedent) |

## 5. Chain of Command (CoC)

Source: [Class extension – Method wrapping and Chain of Command](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/extensibility/method-wrapping-coc), [Extensibility attributes](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/extensibility/extensibility-attributes).

1. Extension classes are `final` and carry `[ExtensionOf(classStr(...))]` (or `tableStr`, `formStr`, `formDataSourceStr`, `formDataFieldStr`, `formControlStr`, `dataEntityViewStr`).
2. You can wrap only **public or protected** methods. You can't wrap private, internal, `final` (unless `[Wrappable(true)]`), `[Wrappable(false)]` or `[Hookable(false)]` methods, or constructors.
3. **Always call `next`.** It must not sit inside an `if`, a loop or a logical expression, and must not come after a `return`. It may sit inside try/catch (Platform update 21 and later).
4. Use the same signature as the wrapped method, without default values. Static methods also need the `static` keyword.
5. `[Replaceable]` methods may skip `next`, but call `next` whenever the replacement logic doesn't apply.
6. Tables and data entities: system methods that aren't implemented (`modifiedField`, `insert` and so on) can be wrapped from Platform update 22.
7. Forms: wrap only root-level form methods. CoC on X++ methods of nested form types compiles but **doesn't run**.
8. Prefer CoC to pre/post event handlers. Microsoft keeps those for backward compatibility; they break when parameters change ([X++ events](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/dev-ref/xpp-events)).
9. If no wrappable extension point exists, don't use reflection or copy standard code. Log an **extensibility request** ([Implementation guide – extend your solution](https://learn.microsoft.com/dynamics365/guidance/implementation-guide/extend-your-solution-guidance-product-fo)).
10. **[Team]** Wrap as **late** and as **narrowly** as possible. Prefer the `initFrom…`/`init…` method that copies source values onto the target record over the big `run()` or `create()` method of a process class. This survives refactoring by Microsoft better.

## 6. Data entities

Source: [Extend a data entity (training)](https://learn.microsoft.com/training/modules/integration-concepts-finance-operations/extend-data-entity), [Data entities and data packages](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/data-entities/data-entities-data-packages).

1. Adding a field to a standard entity takes two parts: an **entity extension** (`<Entity>.HOG<Fff>`) and a **staging table extension**. You create the staging table extension by hand, because "Regenerate staging table" isn't available for entity extensions.
2. Use the same field name and EDT on the entity and the staging table. A type mismatch is a best-practice error.
3. After deployment, run **Refresh entity list** in Data management, or *Generate source mapping* on existing projects.

## 7. Best-practice checks

1. Turn on BP checks for our models (Visual Studio > Dynamics 365 > Options > Best practices). From the command line: `xppbp.exe -module:<Package> -model:<Model> -all -xmllog=<file>` ([Best practice checks](https://learn.microsoft.com/training/modules/explore-test-framework-tools-finance-operations/05-best-practice)).
2. Aim for **zero deviations**. Every suppression goes into the model's BP suppression file with a written justification, as in the FastTrack samples (`AxIgnoreDiagnosticList/<Model>_BPSuppressions.xml`).
3. Before go-live, a **clean Customization Analysis Report (CAR)** is required for the go-live readiness review ([Implementation guide](https://learn.microsoft.com/dynamics365/guidance/implementation-guide/extend-your-solution-guidance-product-fo)). The CAR warns when more than 10 new fields are added to one table ([CAR](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/dev-tools/customization-analysis-report)).

## 8. Repository layout (Metadata)

Same layout as Microsoft's [continuous build and test automation guidance](https://learn.microsoft.com/dynamics365/fin-ops-core/dev-itpro/dev-tools/continuous-build-test-automation) and the FastTrack samples:

```
Metadata/
└── <Package>/
    ├── Descriptor/<Model>.xml
    └── <Model>/
        ├── AxClass/
        ├── AxTableExtension/
        └── ...
Projects/
└── <Package>/<Package>.rnrproj      (Visual Studio project, kept outside Metadata)
```

Only our own packages go into Git. Never commit Microsoft standard source code. If we need to quote standard code for discovery, quote the relevant lines in a doc, not whole files.

## 9. Markers in documents and code

| Marker | Meaning |
|---|---|
| `VERIFY` | A standard object, field or method name that **must be confirmed on the dev VM** before coding against it |
| `D-nn` | Discovery item: a step to run on the VM, with the expected evidence (call stack, source excerpt, screenshot) |
| `DEC-nn` / `OP-nn` / `R-nn` | Decision, open point, risk (doc 08) |
| `TC-nn` | Test case (doc 07) |
| `ADR-nnn` | Architecture/ALM decision record in `docs/adr/` |
