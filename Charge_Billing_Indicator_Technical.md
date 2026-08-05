# Billing Indicators — Technical Developer Reference

## Architecture Overview

```mermaid
flowchart LR
    subgraph DB["Database Layer"]
        SP1["GetV4ChrgWorksEncounter<br/>(stored procedure)"]
        SP2["GetEncounterDiagnosis<br/>(stored procedure)"]
        SP3["GetPreferencesByUserIDWithEnterpriseValue<br/>(stored procedure)"]
        T1["Insurance_Class_DE"]
        T2["ATP_IMOProbIT"]
        T3["Problem_DE_ICDCode"]
        T4["ICD_Reference"]
        T5["AHSPreference"]
        T6["ICD10_Diagnosis_DE"]
    end

    subgraph JS["Client JS Layer"]
        CM["ChargeModule.js<br/>(AngularJS module bootstrap)"]
        CS["Charge.js → ChargeSrv<br/>(IIFE service, onChargeLoad)"]
        CU["ChargeWorksUtils4.js<br/>(utility functions)"]
        MA["ModalAddDxCharge.js<br/>(add dialog controller)"]
        AN["AddNewCharge.js<br/>(search grid controller)"]
        CC["CHContext.js<br/>(encounter context)"]
    end

    subgraph HTML["Templates"]
        H1["Charge.html<br/>(#diagnosisGrid)"]
        H2["AddNewDxChrgItem.html<br/>(modal wrapper)"]
        H3["AddNewCharge.html<br/>(search grid)"]
    end

    CM -->|"$timeout → ChgWrksInitializePromise()"| CS
    CS -->|"CHW.oGetChargeEncounter()"| CC
    CC -->|"RU.GetSingle('ChargeEncounter', id)"| SP1
    SP1 -->|"reads Insurance_Class_DE"| T1
    CS -->|"onChargeLoad() → initDiagnosisGrid()"| H1
    H1 -->|"loadDiagnosisGrid() → ChgWrksGetEncounterDiagnosisList()"| SP2
    SP2 -->|"returns billable flags from"| T2
    SP2 -->|"returns billable flags from"| T3
    CS -->|"GetPreferencesByUserIDWithEnterpriseValue()"| SP3
    SP3 -->|"reads"| T5
    H1 -->|"column template calls"| CU
    CS -->|"showAddDxCharge() → ShellAPI.showModalDialog()"| MA
    MA -->|"initContent()"| AN
    AN -->|"dxColumns template calls"| CU
```

---

## Initialization Call Chain

```mermaid
sequenceDiagram
    participant CM as ChargeModule.js
    participant CS as Charge.js (ChargeSrv)
    participant CC as CHContext.js
    participant CU as ChargeWorksUtils4.js
    participant DB as Database

    CM->>CU: ChgWrksInitializePromise("WORKS CHWORKS OneCharge", 0)
    CU->>CC: CHW.oGetChargeEncounter(false, true)
    CC->>DB: RU.GetSingle("ChargeEncounter", id)<br/>→ SP: GetV4ChrgWorksEncounter
    DB-->>CC: moEnc object with ISMEDICAREMANAGEDCAREFLAG,<br/>ISCOMMERCIALHCCFLAG, insurance data
    CC-->>CM: encounter ready
    CM->>CS: ChargeSrv.onChargeLoad(false)

    Note over CS: Loads preferences:
    CS->>DB: ShellAPI.GetPreference("ShowBillingInformationfor")
    CS->>DB: ShellAPI.GetPreference("Derive Billing Indicators From")
    CS->>CU: getChgPrefValue(msWorksPrefs, "ChgWorksEnableNonBillDX")
    CS->>CU: getChgPrefValue(msWorksPrefs, "MedicareHCCCheck")

    Note over CS: Initializes grids:
    CS->>CS: initDiagnosisGrid()
    CS->>CU: ChgWrksGetEncounterDiagnosisList()
    CU->>DB: ResourceQuery("LoadPlural", "EncounterDiagnosis")<br/>→ SP: GetEncounterDiagnosis
    DB-->>CU: diagnosis rows with ICD10PMSBILLABLEFLAG,<br/>ICD10IMOBILLABLEFLAG, ICD10HCCFLAG, etc.
    CU->>CU: ParseJSONPropertiesToBool(oData, keyAttributes)
    CU->>CU: forEach → getBillableStatus(itm)
    CU->>CU: forEach → IsUnspecified(itm)
    CU->>CU: forEach → getHCCStatus(enc, itm, billableFlag)
    CU-->>CS: processed diagnosis data
    CS->>CS: Grid renders → column template calls getBillingImgUrl()
```

---

## Preference Loading Details

| Preference Key                   | Loaded In                                        | API Call                             | Stored In Variable              | Possible Values                                                                                                              |
| -------------------------------- | ------------------------------------------------ | ------------------------------------ | ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| `ShowBillingInformationfor`      | `Charge.js` `onChargeLoad()` line 467            | `ShellAPI.GetPreference(...)`        | `msShowBillingInformationfor`   | `"ICD-10"`, `"ICD-9"`                                                                                                        |
| `Derive Billing Indicators From` | `Charge.js` `onChargeLoad()` line 468            | `ShellAPI.GetPreference(...)`        | `msDeriveBillingIndicatorsFrom` | `"Practice Management Only"`, `"Enterprise Only"`, `"Practice Management and Enterprise"`                                    |
| `ChgWorksEnableNonBillDX`        | `Charge.js` `onChargeLoad()` line 469            | `getChgPrefValue(msWorksPrefs, ...)` | `msPrefIsBillable`              | `"N"` (off), `"Y"`, `"Non-billable codes not selectable"`                                                                    |
| `MedicareHCCCheck`               | `Charge.js` `onChargeLoad()` line 470            | `getChgPrefValue(msWorksPrefs, ...)` | `msPrefIsHCC`                   | `"Y"`, `"N"`                                                                                                                 |
| `Medicare Managed HCC Icons`     | `ChargeWorksUtils4.js` `getHCCStatus()` line 903 | `ShellAPI.GetPreference(...)`        | local in function               | `"Show only v24 HCC Icons "` (trailing space), `"Show only v28 HCC Icons"`, `"Show v24 and v28 HCC Icons "` (trailing space) |

Preferences are accessed at runtime via `ChargeSrv.getPreference(key)` which reads from the module-scoped variables in `Charge.js`.

In the Add Dx dialog, preferences are re-loaded independently in `ModalAddDxCharge.js` `vm.initContent()` (lines 104-113).

The DB source for preferences is the `AHSPreference` table, loaded via stored procedure `GetPreferencesByUserIDWithEnterpriseValue` (resource action ID 170, resource ID 48 `CMSPreference`).

---

## Core Function: `getBillableStatus(item)`

**File:** `ChargeWorksUtils4.js` lines 813-870

**Input:** A diagnosis data item with these properties (boolean after `ParseJSONPropertiesToBool`):

| Property               | Type    | Source                                    |
| ---------------------- | ------- | ----------------------------------------- |
| `DIAGNOSISDE`          | number  | Diagnosis dictionary entry ID             |
| `ICD10PMSBILLABLEFLAG` | boolean | PMS billable for ICD-10                   |
| `ICD9PMSBILLABLEFLAG`  | boolean | PMS billable for ICD-9                    |
| `ICD10IMOBILLABLEFLAG` | boolean | Enterprise/IMO billable for ICD-10        |
| `ICD9IMOBILLABLEFLAG`  | boolean | Enterprise/IMO billable for ICD-9         |
| `POSTCOORDLEXFLAG`     | boolean | Post-coordinated lexicon flag             |
| `ISSPECIFIED`          | string  | Pre-existing specificity (may be mutated) |

**Returns:** `"true"` | `"false"` | `"NotReady"` | `""` (empty = indicators disabled)

```mermaid
flowchart TD
    START["getBillableStatus(item)"] --> Q0{"item.DIAGNOSISDE == 0?"}
    Q0 -- "Yes" --> RET_TRUE["return 'true'"]
    Q0 -- "No" --> Q1{"msPrefIsBillable != 'N'?"}
    Q1 -- "No (disabled)" --> RET_EMPTY["return '' (no indicators)"]
    Q1 -- "Yes" --> Q2{"msDeriveBillingIndicatorsFrom?"}

    Q2 -- "'Practice Management Only'" --> PM{"msShowBillingInformationfor<br/>== 'ICD-10'?"}
    PM -- "Yes" --> PM_10["bIsBillable = item.ICD10PMSBILLABLEFLAG"]
    PM -- "No" --> PM_9["bIsBillable = item.ICD9PMSBILLABLEFLAG"]

    Q2 -- "'Enterprise Only'" --> ENT{"msShowBillingInformationfor<br/>== 'ICD-10'?"}
    ENT -- "Yes" --> ENT_10["bIsBillable = item.ICD10IMOBILLABLEFLAG"]
    ENT -- "No" --> ENT_9["bIsBillable = item.ICD9IMOBILLABLEFLAG"]

    Q2 -- "else (default =<br/>'Practice Management<br/>and Enterprise')" --> BOTH{"msShowBillingInformationfor<br/>== 'ICD-10'?"}
    BOTH -- "Yes" --> BOTH_10["bIsBillable = item.ICD10PMSBILLABLEFLAG<br/>&& item.ICD10IMOBILLABLEFLAG"]
    BOTH -- "No" --> BOTH_9["bIsBillable = item.ICD9PMSBILLABLEFLAG<br/>&& item.ICD9IMOBILLABLEFLAG"]

    PM_10 --> POST
    PM_9 --> POST
    ENT_10 --> POST
    ENT_9 --> POST
    BOTH_10 --> POST
    BOTH_9 --> POST

    POST{"bIsBillable != true<br/>AND item.POSTCOORDLEXFLAG?"}
    POST -- "Yes" --> NR["bIsBillable = 'NotReady'"]
    POST -- "No" --> FINAL
    NR --> RET_NR["return 'NotReady'"]

    FINAL{"!bIsBillable (falsy)?"}
    FINAL -- "Yes" --> RET_FALSE["return 'false'"]
    FINAL -- "No (truthy)" --> SPEC{"item.ISSPECIFIED truthy?"}
    SPEC -- "Yes" --> MUTATE["item.ISSPECIFIED = IsUnspecified(item)<br/>(side-effect: mutates item)"]
    SPEC -- "No" --> RET_TRUE2["return 'true'"]
    MUTATE --> RET_TRUE2
```

**Note:** When `bIsBillable` is the result of `&&` on two booleans, JavaScript returns `true`/`false` (boolean), not a string. The subsequent `bIsBillable != true` check works because `false != true` is `true`. If only one flag is used (PM Only or Enterprise Only), `bIsBillable` is directly a boolean from `ParseJSONPropertiesToBool`.

---

## Core Function: `IsUnspecified(item)`

**File:** `ChargeWorksUtils4.js` lines 871-897

**Returns:** `"true"` | `"false"`

**Key behavior:** Only evaluates when `msShowBillingInformationfor == "ICD-10"`. For ICD-9, always returns `"false"`.

| Derive From              | Property Checked           | Unspecified When              |
| ------------------------ | -------------------------- | ----------------------------- |
| Enterprise Only          | `item.ICD10IMOSPECIFICITY` | `== "1"`                      |
| Practice Management Only | `item.ICD10PMSSPECIFICITY` | `== "1"`                      |
| Both (else)              | Either property            | Either `== "1"` (uses `\|\|`) |

**Note on "Both" logic:** The `else` branch uses JavaScript `||` with the ternary expressions. If `ICD10IMOSPECIFICITY` is truthy and equals `"1"`, it returns `"true"`. Otherwise it falls through to check `ICD10PMSSPECIFICITY`.

---

## Core Function: `getHCCStatus(moEnc, item, bIsBillable)`

**File:** `ChargeWorksUtils4.js` lines 899-925

**Returns:** `"true"` | `"false"`

**Preconditions:** Only evaluates if ALL of:

- `moEnc` is truthy
- `bIsBillable == "true"` (string comparison)
- `ChargeSrv.getPreference("msPrefIsHCC") == "Y"`

```mermaid
flowchart TD
    START["getHCCStatus(moEnc, item, bIsBillable)"] --> PRE{"moEnc AND<br/>bIsBillable == 'true' AND<br/>msPrefIsHCC == 'Y'?"}
    PRE -- "No" --> RET_F["return 'false'"]
    PRE -- "Yes" --> MCARE{"moEnc.ISMEDICAREMANAGEDCAREFLAG == 'Y'?"}

    MCARE -- "Yes" --> VER{"ShellAPI.GetPreference(<br/>'Medicare Managed HCC Icons')?"}
    VER -- "'Show only v24 HCC Icons '" --> V24["item.ICD10HCCFLAG === true?"]
    VER -- "'Show only v28 HCC Icons'" --> V28["item.ICD10HCCFLAGNEXT === true?"]
    VER -- "'Show v24 and v28 HCC Icons '<br/>or default" --> V24V28["item.ICD10HCCFLAG === true<br/>OR item.ICD10HCCFLAGNEXT === true?"]

    MCARE -- "No" --> COMM{"moEnc.ISCOMMERCIALHCCFLAG == 'Y'?"}
    COMM -- "Yes" --> COMM_CHK["item.PATIENTICD10COMMERCIALHCCFLAG === true?"]
    COMM -- "No" --> RET_F

    V24 -- "true" --> RET_T["return 'true'"]
    V24 -- "false" --> RET_F
    V28 -- "true" --> RET_T
    V28 -- "false" --> RET_F
    V24V28 -- "true" --> RET_T
    V24V28 -- "false" --> RET_F
    COMM_CHK -- "true" --> RET_T
    COMM_CHK -- "false" --> RET_F
```

**Note:** `ISMEDICAREMANAGEDCAREFLAG` takes priority. If both Medicare and Commercial flags are `"Y"`, only Medicare HCC logic runs (the Commercial branch is in `else if`).

---

## Core Function: `getBillingImgUrl(dataItem)`

**File:** `ChargeWorksUtils4.js` lines 1200-1230

Orchestrates the three core functions and maps results to CSS icon class names.

```mermaid
flowchart TD
    START["getBillingImgUrl(dataItem)"] --> GUARD{"dataItem has ICD10DIAGNOSIS<br/>OR ICD10DIAGNOSISCODE<br/>OR ICD10CODE<br/>OR POSTCOORDLEXFLAG<br/>OR ENTRYCODE?"}
    GUARD -- "No" --> RET_EMPTY["return '' (no icon)"]
    GUARD -- "Yes" --> CALC["bIsBillable = getBillableStatus(dataItem)<br/>bIsUnspecified = IsUnspecified(dataItem)<br/>hccImageClass = bIsUnspecified == 'true'<br/>  ? 'Icon_HCC_blue_unspecified'<br/>  : 'Icon_HCC_blue'"]

    CALC --> Q1{"bIsBillable == 'false'?"}
    Q1 -- "Yes" --> IC1["return 'Icon_NBVisitCharges'<br/>tooltip: 'Problem Not Billable'"]
    Q1 -- "No" --> Q2{"bIsBillable == 'NotReady'?"}
    Q2 -- "Yes" --> IC2["return 'Icon_NotreadyBillable'<br/>tooltip: 'More specific problem<br/>needed for billing'"]
    Q2 -- "No" --> Q3{"bIsBillable == 'true'?"}
    Q3 -- "No (empty string)" --> RET_EMPTY
    Q3 -- "Yes" --> Q4{"getHCCStatus() == 'true'?"}
    Q4 -- "Yes" --> IC3["return hccImageClass<br/>(Icon_HCC_blue or<br/>Icon_HCC_blue_unspecified)"]
    Q4 -- "No" --> Q5{"bIsUnspecified == 'true'?"}
    Q5 -- "Yes" --> IC4["return 'Icon_Unspecified'<br/>tooltip: 'Problem billing<br/>status unknown'"]
    Q5 -- "No" --> IC5["return 'Icon_Blank'<br/>(no tooltip)"]
```

---

## Database: Encounter Insurance Flag Derivation

**Stored Procedure:** `dbo.GetV4ChrgWorksEncounter`

**File:** `Database/Objects/Clinical/Stored Procedures/dbo.GetV4ChrgWorksEncounter.prc`

```mermaid
flowchart TD
    START["GetV4ChrgWorksEncounter(@ID)"] --> LOAD["Load Encounter, Visit, Appointment<br/>from dbo.Encounter e<br/>INNER JOIN dbo.Visit v"]

    LOAD --> INS_SRC{"Visit has PrimaryInsuranceDE?<br/>(v.PrimaryInsuranceDE != 0)"}

    INS_SRC -- "No (= 0)" --> DEFAULT["Use e.InsuranceClassDE<br/>(encounter default)"]
    DEFAULT --> SINGLE["SELECT from Insurance_Class_DE<br/>WHERE ID = @DefaultInsuranceDE"]
    SINGLE --> SET_FLAGS["@IsMedicareFLAG = MedicareFLAG<br/>@IsMedicareManagedCareFLAG = MedicareManagedCareFLAG<br/>@IsCommercialHCCFlag = CommercialManagedCareFLAG"]

    INS_SRC -- "Yes (> 0)" --> VISIT_INS["Use visit insurance:<br/>@PrimaryInsuranceDE<br/>@SecondaryInsuranceDE<br/>@TerciaryInsuranceDE"]
    VISIT_INS --> AGG["SELECT with SUM/CASE<br/>FROM Insurance_Class_DE<br/>WHERE ID IN (Pri, Sec, Ter)"]
    AGG --> AGG_FLAGS["If ANY insurance has flag = 'Y'<br/>→ encounter flag = 'Y'<br/>else → 'N'"]

    AGG_FLAGS --> OUT["Output: moEnc.ISMEDICAREMANAGEDCAREFLAG<br/>moEnc.ISCOMMERCIALHCCFLAG"]
    SET_FLAGS --> OUT
```

**Table:** `dbo.Insurance_Class_DE` — columns: `MedicareFLAG`, `MedicareManagedCareFLAG`, `CommercialManagedCareFLAG`

---

## Database: Diagnosis Billable Flags

**Stored Procedure:** `GetEncounterDiagnosis` (resource action ID 269, resource `EncounterDiagnosis`)

Returns per-diagnosis properties used by client-side logic:

| Output Property                 | DB Source                                      | Description                                       |
| ------------------------------- | ---------------------------------------------- | ------------------------------------------------- |
| `ICD10PMSBILLABLEFLAG`          | `Problem_DE_ICDCode` JOIN `ICD10_Diagnosis_DE` | `'Y'` if active ICD-10 code mapping exists in PMS |
| `ICD10IMOBILLABLEFLAG`          | `ICD_Reference` WHERE `IsBillableFLAG = 'Y'`   | `'Y'` if Enterprise/IMO marks as billable         |
| `ICD9PMSBILLABLEFLAG`           | `Problem_DE_ICDCode` JOIN `ICD9_Diagnosis_DE`  | Same logic for ICD-9                              |
| `ICD9IMOBILLABLEFLAG`           | `ICD_Reference` WHERE `ICDType = 'ICD9CM'`     | Same logic for ICD-9                              |
| `POSTCOORDLEXFLAG`              | `ATP_IMOProbIT.Post_Coord_Lex_FLAG`            | Indicates code needs post-coordination            |
| `ICD10IMOSPECIFICITY`           | `ATP_IMOProbIT`                                | `1` = unspecified (Enterprise)                    |
| `ICD10PMSSPECIFICITY`           | `ATP_IMOProbIT`                                | `1` = unspecified (PMS)                           |
| `ICD10HCCFLAG`                  | Encounter Diagnosis query                      | V24 HCC category flag                             |
| `ICD10HCCFLAGNEXT`              | Encounter Diagnosis query                      | V28 HCC category flag                             |
| `PATIENTICD10COMMERCIALHCCFLAG` | Encounter Diagnosis query                      | Commercial HCC flag for patient                   |

Client-side parsing in `ChargeWorksUtils4.js` `ChgWrksGetEncounterDiagnosisList()` (lines 278-280):

```javascript
ParseJSONPropertiesToBool(oData, ['ICD10PMSBILLABLEFLAG', 'ICD10IMOBILLABLEFLAG',
  'POSTCOORDLEXFLAG', 'ICD10HCCFLAG', 'ICD10HCCFLAGNEXT', 'PATIENTICD10COMMERCIALHCCFLAG', ...])
```

Converts `"Y"`/`"N"` strings to `true`/`false` booleans before billing logic runs.

---

## Database: Server-Side Billable Derivation

**Stored Procedure:** `dbo.DeriveBillableIndicators`

**File:** `Database/Objects/Clinical/Stored Procedures/dbo.DeriveBillableIndicators.prc`

This SP mirrors the client-side `getBillableStatus()` logic and is used by orders/problems workflows. It reads the `Derive Billing Indicators From` and `ShowBillingInformationfor` preferences from the `AHSPreference` table and checks:

- **PMS Billable:** `Problem_DE_ICDCode` joined with `ICD10_Diagnosis_DE` — code must have an active mapping
- **Enterprise/IMO Billable:** `ICD_Reference` table — `IsBillableFLAG = 'Y'` for the code
- **"Practice Management and Enterprise":** Both checks must pass (AND logic)

---

## Display Integration Points

**1. Main Diagnosis Grid** — `Charge.js` `initDiagnosisGrid()` (lines 2265-2280):

- Column with `class: "imgDxGridColumn"` renders the billing icon
- Template calls `getBillingImgUrl(dataItem)` and `getBillingImgToolTip(imgClass, dataItem)`
- Data loaded via `loadDiagnosisGrid()` → `ChgWrksGetEncounterDiagnosisList()`

**2. Add Dx Search Grid** — `AddNewCharge.js` `$scope.dxColumns`:

- First column (`field: "ICD10DIAGNOSISCODE"`) renders `getBillableImageClass(dataItem)` via `ChargeWorksUtils4.js` line 1177
- Last checkmark column also calls `checkBillableDx(dataItem)` to apply `dxRowIconDisabledState` CSS

**3. Badge Tooltip** — `ModalAddDxCharge.js` `$scope.buildAddedItemsToolTip()` (lines 361-381):

- Calls `getBillableImageClass(excludeRemovedDxList[i])` per diagnosis
- Renders into `#dxToolTipContent` div

**4. Selectability Gate** — `ChargeWorksUtils4.js` `checkBillableDx(dxItm)` (lines 1721-1726):

- Returns `false` when `msPrefIsBillable == "Non-billable codes not selectable"` AND `getBillableStatus()` returns `"false"` or `"NotReady"`
- Used by `addOrRemoveItems()` in `AddNewCharge.js` to block selection

---

## Icon Constants Reference

| Constant                    | CSS Class                   | Tooltip Constant        | Tooltip Text                                                                                          |
| --------------------------- | --------------------------- | ----------------------- | ----------------------------------------------------------------------------------------------------- |
| `ICON_NOT_BILLABLE`         | `Icon_NBVisitCharges`       | `PROBLEMNOTBILLABLE`    | "Problem Not Billable"                                                                                |
| `ICON_NOT_READY`            | `Icon_NotreadyBillable`     | `PROBLEMNOTREADY`       | "More specific problem needed for billing"                                                            |
| `ICON_UNSPECIFIED`          | `Icon_Unspecified`          | `PROBLEMBILLINGUNKNOWN` | "Problem billing status unknown"                                                                      |
| `ICON_HCC_BLUE`             | `Icon_HCC_blue`             | via `getTooltip()`      | "V24 HCC Category" / "V28 HCC Category" / "Both V24 and V28 HCC Category" / "Commercial HCC Category" |
| `ICON_HCC_BLUE_UNSPECIFIED` | `Icon_HCC_blue_unspecified` | via `getTooltip(,true)` | HCC tooltip + `<br>` + "Problem billing status unknown"                                               |
| `ICON_BLANK`                | `Icon_Blank`                | (none)                  | No tooltip                                                                                            |

---

## Tooltip Logic: `getTooltip(dataItem, isunspecified)`

**File:** `ChargeWorksUtils4.js` lines 1232-1275

For **Medicare Managed Care** encounters, tooltip varies by HCC Icons preference:

| HCC Icons Preference | V24 Flag | V28 Flag | Tooltip                         |
| -------------------- | -------- | -------- | ------------------------------- |
| Show only v24        | true     | —        | "V24 HCC Category"              |
| Show only v28        | —        | true     | "V28 HCC Category"              |
| Show v24 and v28     | true     | true     | "Both V24 and V28 HCC Category" |
| Show v24 and v28     | true     | false    | "V24 HCC Category"              |
| Show v24 and v28     | false    | true     | "V28 HCC Category"              |

For **Commercial HCC** encounters: tooltip is "Commercial HCC Category" when `PATIENTICD10COMMERCIALHCCFLAG` is true.

If `isunspecified` is true, appends `<br>Problem billing status unknown` to the tooltip.

`getBillingImgToolTip()` dispatches to `getTooltip()` for HCC icons, and returns static strings for other icon types.

---

## File Reference

| File                                             | Path                                                                                 | Role                                                                                                                                                                                             |
| ------------------------------------------------ | ------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| ChargeModule.js                                  | `TW/UI/Web/Shell/scripts/ChargeModule.js`                                            | AngularJS module + `chargeController` — bootstraps workspace, calls `ChgWrksInitializePromise()`                                                                                                 |
| Charge.js                                        | `TW/UI/Web/Shell/scripts/frontpage/Charge.js`                                        | `ChargeSrv` IIFE — `onChargeLoad()` loads prefs, `initDiagnosisGrid()` renders billing icons                                                                                                     |
| ChargeWorksUtils4.js                             | `TW/UI/Web/Shell/scripts/frontpage/ChargeWorksUtils4.js`                             | `getBillableStatus()`, `IsUnspecified()`, `getHCCStatus()`, `getBillingImgUrl()`, `getBillingImgToolTip()`, `checkBillableDx()`, `getBillableImageClass()`, `ChgWrksGetEncounterDiagnosisList()` |
| ModalAddDxCharge.js                              | `TW/UI/Web/Shell/app/Modules/Dialogs/Charge/ModalAddDxCharge.js`                     | Add Dx/Charge modal controller — re-loads prefs in `vm.initContent()`, builds tooltip                                                                                                            |
| AddNewCharge.js                                  | `TW/UI/Web/Shell/scripts/frontpage/AddNewCharge.js`                                  | Search grid controller — `dxColumns` templates call billing icon functions                                                                                                                       |
| CHContext.js                                     | `TW/UI/Web/Shell/scripts/frontpage/CHContext.js`                                     | `oGetChargeEncounter()` → `oLoadEncounter()` → `RU.GetSingle("ChargeEncounter")`                                                                                                                 |
| ChgEnterprisePreferences.js                      | `TW/UI/Web/Shell/scripts/frontpage/ChgEnterprisePreferences.js`                      | Admin UI for enterprise billing prefs                                                                                                                                                            |
| ChargeDetail.js                                  | `TW/UI/Web/Shell/scripts/frontpage/ChargeDetail.js`                                  | Separate XML-based `getBillableStatus(ItemXML)` used in charge detail dialog                                                                                                                     |
| GetV4ChrgWorksEncounter.prc                      | `Database/Objects/Clinical/Stored Procedures/dbo.GetV4ChrgWorksEncounter.prc`        | Sets `ISMEDICAREMANAGEDCAREFLAG`, `ISCOMMERCIALHCCFLAG` from `Insurance_Class_DE`                                                                                                                |
| DeriveBillableIndicators.prc                     | `Database/Objects/Clinical/Stored Procedures/dbo.DeriveBillableIndicators.prc`       | Server-side billable derivation (used by orders/problems, mirrors client logic)                                                                                                                  |
| 808 Create TWResources And TWResourceActions.sql | `Database/Objects/TWCommon/Scripts/808 Create TWResources And TWResourceActions.sql` | Resource action mappings: ChargeEncounter (ID 39), EncounterDiagnosis, CMSPreference                                                                                                             |
