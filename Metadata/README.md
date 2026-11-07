# Metadata

X++ packages of this solution. Each folder here is linked into `PackagesLocalDirectory` on the dev VM (doc 06 §2).

Planned, created after discovery (doc 02 §4):

```
Metadata/
├── HOGsbaSubBillActivity/
│   ├── Descriptor/HOGsbaSubBillActivity.xml
│   └── HOGsbaSubBillActivity/
│       ├── AxClass/                 SubBillScheduleLine_HOGsba_Extension, SubBillCreateSalesOrder_HOGsba_Extension, HOGsbaSalesLineActivity, form CoC classes
│       ├── AxTableExtension/        SubBillScheduleLine.HOGsba, <staging table>.HOGsba
│       ├── AxFormExtension/         <billing schedule form>.HOGsba
│       └── AxDataEntityViewExtension/ <line entity>.HOGsba
└── HOGsbaSubBillActivityTest/
    ├── Descriptor/HOGsbaSubBillActivityTest.xml
    └── HOGsbaSubBillActivityTest/AxClass/   HOGsbaScheduleLineActivityTest, HOGsbaSalesLineActivityTest
```
