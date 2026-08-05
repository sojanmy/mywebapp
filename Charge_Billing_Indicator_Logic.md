# Charge Workspace — Billing Indicator Derivation Logic (Code-Verified)

## Overview

When a clinician opens the Charge workspace or the Add Diagnosis/Charge dialog, the system displays visual indicators (icons) next to each diagnosis code. These icons communicate whether the diagnosis is billable, requires more specificity, qualifies for a Hierarchical Condition Category (HCC), or cannot be billed.

This document explains the complete logic behind those indicators, verified directly from the **26.1 source code**.

**Source files verified:**
- ChargeModule.js — module/controller bootstrap
- Charge.js — preference loading, grid initialization
- ChargeWorksUtils4.js — billable status, unspecified check, HCC check, icon resolution
- ModalAddDxCharge.js — Add Dx/Charge dialog controller
- dbo.GetV4ChrgWorksEncounter.prc — encounter insurance flag logic
- dbo.DeriveBillableIndicators.prc — server-side billable derivation

---

## Indicator Summary

| Icon | Status | Meaning | When It Appears |
|------|--------|---------|-----------------|
| Red | **Not Billable** | This diagnosis code cannot be billed | Code failed the billable check in the configured system(s) and is not a post-coordinated code |
| Yellow | **Not Ready** | A more specific code is needed before this can be billed | Code failed billable check BUT is flagged as needing post-coordination (a more specific code can be derived) |
| Gray | **Unspecified** | The code is billable but its specificity is unknown | Code is billable, but specificity data indicates it is unspecified (ICD-10 only) |
| Blue | **HCC Category** | This billable code qualifies for a Hierarchical Condition Category | Code is billable + encounter has Medicare or Commercial managed care insurance + code has an HCC category flag |
| Blue+Gray | **HCC + Unspecified** | HCC category applies but code specificity is unknown | Same as HCC but the code is also unspecified |
| None | **Billable** (no icon) | Code is billable, specified, and has no HCC relevance | Code passed all checks cleanly — no icon is needed |

---

## Billing Indicator Derivation — Main Flow

```mermaid
flowchart TD
    START([Clinician opens Charge workspace or Add Diagnosis/Charge dialog]) --> LOAD_PREFS[System loads billing preferences from user and organization settings]

    LOAD_PREFS --> LOAD_ENC[System loads the current encounter including insurance information]

    LOAD_ENC --> LOAD_DX[System retrieves diagnosis data from the database including billable flags and HCC flags]

    LOAD_DX --> GATE{Is the Enable Non-Billable Diagnosis Indicators setting turned on?}

    GATE -- Turned off --> NO_ICONS[No billing indicators are displayed at all]

    GATE -- Turned on --> HAS_CODE{Does the diagnosis have an ICD code?}

    HAS_CODE -- No --> NO_ICON_ITEM[No indicator for this diagnosis]

    HAS_CODE -- Yes --> DERIVE{Which system should billable status be derived from?}

    DERIVE -- Practice Management Only --> PMS[Check if the diagnosis code has an active mapping in the Practice Management system]

    DERIVE -- Enterprise Only --> ENT[Check if the diagnosis code is marked as billable in the Enterprise reference data]

    DERIVE -- Practice Management and Enterprise --> BOTH_CHECK[Check BOTH systems - Code must be billable in both to pass]

    PMS --> BILLABLE_Q{Is the diagnosis billable?}
    ENT --> BILLABLE_Q
    BOTH_CHECK --> BILLABLE_Q

    BILLABLE_Q -- Not billable --> POST_COORD{Does the diagnosis require a more specific code? i.e. Post-coordinated}

    POST_COORD -- Yes --> ICON_NOTREADY[Not Ready - More specific problem needed for billing]
    POST_COORD -- No --> ICON_NOTBILLABLE[Not Billable - Problem Not Billable]

    BILLABLE_Q -- Billable --> CODING_SYS{Is ICD-10 the selected coding system for billing display?}

    CODING_SYS -- ICD-9 selected --> HCC_CHECK
    CODING_SYS -- ICD-10 selected --> UNSPEC_CHECK{Is this diagnosis code considered unspecified in the selected system?}

    UNSPEC_CHECK --> HCC_CHECK{Is the Medicare HCC Check setting enabled?}

    HCC_CHECK -- Disabled --> FINAL_NO_HCC{Was the code unspecified?}

    FINAL_NO_HCC -- Yes --> ICON_UNSPEC[Unspecified - Problem billing status unknown]
    FINAL_NO_HCC -- No --> ICON_OK[Billable - No icon displayed]

    HCC_CHECK -- Enabled --> INS_CHECK{What type of insurance does this encounter have?}

    INS_CHECK -- Medicare Managed Care --> HCC_VERSION{Which HCC version is configured to show?}
    INS_CHECK -- Commercial Managed Care --> COMM_HCC{Does this code have a Commercial HCC category?}
    INS_CHECK -- Neither --> FINAL_NO_HCC

    HCC_VERSION -- V24 only --> V24{Does this code have a V24 HCC category?}
    HCC_VERSION -- V28 only --> V28{Does this code have a V28 HCC category?}
    HCC_VERSION -- Both V24 and V28 --> V24V28{Does this code have a V24 or V28 HCC category?}

    V24 -- Yes --> HCC_UNSPEC
    V24 -- No --> FINAL_NO_HCC
    V28 -- Yes --> HCC_UNSPEC
    V28 -- No --> FINAL_NO_HCC
    V24V28 -- Yes --> HCC_UNSPEC
    V24V28 -- No --> FINAL_NO_HCC
    COMM_HCC -- Yes --> HCC_UNSPEC
    COMM_HCC -- No --> FINAL_NO_HCC

    HCC_UNSPEC{Was the code also unspecified?}
    HCC_UNSPEC -- Yes --> ICON_HCC_UNSPEC[HCC + Unspecified]
    HCC_UNSPEC -- No --> ICON_HCC[HCC Category]
```

---

## How the System Determines Insurance Type for HCC

The encounter's insurance type determines whether Medicare or Commercial HCC icons can appear. This is resolved when the encounter loads, based on the patient's insurance classes.

```mermaid
flowchart TD
    ENC([Encounter is loaded]) --> VISIT_INS{Does the visit have insurance assigned?}

    VISIT_INS -- Yes --> USE_VISIT[Use the visit's Primary, Secondary, and Tertiary insurance classes]

    VISIT_INS -- No --> USE_DEFAULT[Fall back to the encounter's default insurance class]

    USE_VISIT --> LOOKUP[Look up each insurance class in the Insurance Class dictionary]
    USE_DEFAULT --> LOOKUP

    LOOKUP --> FLAGS[Each insurance class has administrator-configured flags: Medicare Flag, Medicare Managed Care Flag, Commercial Managed Care Flag]

    FLAGS --> ANY_CHECK{Does ANY of the patient's insurance classes have the flag set?}

    ANY_CHECK -- Any has Medicare Managed Care = Yes --> MEDICARE_ENC[Encounter is treated as Medicare Managed Care - Medicare HCC icons apply]

    ANY_CHECK -- Any has Commercial Managed Care = Yes --> COMMERCIAL_ENC[Encounter is treated as Commercial Managed Care - Commercial HCC icons apply]

    ANY_CHECK -- None flagged --> NEITHER[No HCC icons apply regardless of diagnosis codes]
```

---

## How Billable Status Is Determined at the Data Level

```mermaid
flowchart TD
    DX([Diagnosis code on encounter]) --> SRC{Which source is configured?}

    SRC -- Practice Management --> PMS_CHECK[Check if the code has an active mapping in the Practice Management ICD code tables]

    SRC -- Enterprise --> ENT_CHECK[Check if the code exists in the Enterprise Reference data as billable]

    SRC -- Both --> BOTH_PASS[Code must pass BOTH checks to be billable]

    PMS_CHECK --> PMS_DETAIL[PMS Billable means: The diagnosis has an ICD-10 code mapping, that code has an active entry in the PMS diagnosis code table. If no mapping exists or code is inactive then Not PMS Billable]

    ENT_CHECK --> ENT_DETAIL[Enterprise Billable means: The ICD-10 code exists in the Enterprise ICD Reference data with Is Billable = Yes. If not found or not billable then Not Enterprise Billable]

    BOTH_PASS --> PMS_DETAIL
    BOTH_PASS --> ENT_DETAIL
```

---

## Selectability Rule

When the enterprise preference is set to **"Non-billable codes not selectable"**:

- Diagnoses that are **Not Billable** or **Not Ready** are grayed out in the search results
- The clinician **cannot add** these diagnoses to the encounter
- Billable diagnoses remain fully selectable regardless of unspecified or HCC status

---

## Preferences That Control This Logic

| Setting | Who Sets It | Options | Effect |
|---------|------------|---------|--------|
| **Enable Non-Billable Diagnosis Indicators** | Organization administrator (Enterprise level) | Off / On / Non-billable codes not selectable | Controls whether icons appear, and whether non-billable items are blocked from selection |
| **Show Billing Information For** | Individual user | ICD-10 / ICD-9 | Determines which coding system's billable flags are checked. Also controls whether the Unspecified check runs (only runs for ICD-10) |
| **Derive Billing Indicators From** | Individual user | Practice Management Only / Enterprise Only / Practice Management and Enterprise | Determines which data source(s) the billable check uses |
| **Medicare HCC Check** | Organization administrator (Enterprise level) | Yes / No | Enables or disables all HCC icon logic |
| **Medicare Managed HCC Icons** | Individual user | Show only V24 / Show only V28 / Show V24 and V28 | For Medicare encounters, controls which HCC model version icons are displayed |

---

## Where These Indicators Appear

1. **Main Charge workspace** — in the Diagnosis grid (the icon column next to each diagnosis)
2. **Add Diagnosis/Charge dialog** — in the search results grid when searching for diagnoses to add
3. **Summary badge tooltip** — when hovering over the Dx count badge in the Add dialog, the tooltip lists all added diagnoses with their billing icons

---

## HCC Tooltip Details

When an HCC icon is shown, the hover tooltip displays specific information about which HCC model applies:

| Encounter Insurance Type | HCC Preference | Condition | Tooltip Text |
|--------------------------|---------------|-----------|-------------|
| Medicare Managed Care | Show only V24 | Code has V24 HCC flag | V24 HCC Category |
| Medicare Managed Care | Show only V28 | Code has V28 HCC flag | V28 HCC Category |
| Medicare Managed Care | Show V24 and V28 | Code has both V24 and V28 flags | Both V24 and V28 HCC Category |
| Medicare Managed Care | Show V24 and V28 | Code has only V24 flag | V24 HCC Category |
| Medicare Managed Care | Show V24 and V28 | Code has only V28 flag | V28 HCC Category |
| Commercial Managed Care | (any) | Code has Commercial HCC flag | Commercial HCC Category |
| (any with HCC) | (any) | Code is also unspecified | [HCC tooltip] + Problem billing status unknown |

---

*Document generated: August 2026 — Verified from TWEHR 26.1 source code*
