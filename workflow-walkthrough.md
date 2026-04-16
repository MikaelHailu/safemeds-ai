# SafeMeds AI — Workflow Walkthrough

![SafeMeds AI Workflow](https://github.com/MikaelHailu/safemeds-ai/blob/main/SafeMeds_Workflow.png)

## Overview

SafeMeds AI operates as a fully automated pipeline triggered every 5 minutes. When a clinician completes a prescription in DHIS2 Tracker, the workflow detects it, enriches it with drug safety data from three external sources, runs an AI clinical assessment, and writes a structured Drug Safety Alert back to the patient's record — all without any manual intervention.

---

## Step by Step

### DHIS2: Patient Enrollment and Clinical Visit

The process begins in DHIS2 Tracker. A patient is enrolled in the **Primary Health Record** programme and attends a clinical visit. The clinician opens the **Prescription stage** and records:

- **Diagnosis** — using an ICPC-2 code (e.g. `W78` for pregnancy, `K86` for hypertension)
- **Drug 1, Drug 2, Drug 3** — selected from the local formulary option set (e.g. `EA104 - WARFARIN 5MG`)
- **Pregnancy status** — Yes / No
- **Age** — captured as a tracked entity attribute at enrolment

When the clinician marks the stage as **Complete**, it becomes eligible for safety checking.

---

### Automated Polling (every 5 minutes)

OpenFn Lightning polls DHIS2 every 5 minutes for prescription events that are:
- Status: `COMPLETED`
- Updated in the last 5 minutes
- `Safety Check Completed` flag (`SRgePliKEnM`) not yet set to `true`

The `Safety Check Completed` flag ensures each prescription is processed exactly once, even if the same patient has multiple prescriptions at different visits.

---

### Job 1: Fetch Prescription

OpenFn retrieves the full prescription event from DHIS2, including all data element values. It then makes a second call to fetch the patient's **tracked entity attributes** (age, patient ID) which are stored at the enrollment level rather than the event level.

The patient context assembled at this stage:
- Tracker identifiers (event UID, TEI UID, enrollment UID, org unit)
- Diagnosis ICPC-2 code
- Raw drug codes from the local formulary
- Pregnancy status
- Age

---

### Central Mapping Tables (Job 2)

Before any external API calls, two mapping tables translate the raw DHIS2 data into standardised clinical identifiers:

**Drug Formulary Mapping**
Converts local drug codes like `EA104 - WARFARIN 5MG` into:
- Generic name: `WARFARIN`
- RxCUI: `855288` (NIH RxNorm identifier)

The RxCUI is the key that unlocks both the RxNav interaction API and the OpenFDA label API. Without it, neither external service can identify the drug.

**ICPC-2 Diagnosis Mapping**
Converts ICPC-2 codes like `W78` into:
- Clinical term: `pregnancy` — used in the AI prompt and to interpret FDA label warnings
- Human-readable label: `Pregnancy` — used in the alert display

The clinical term is also used to contextualise FDA label data, which references conditions using ICD-10 language. The terms are chosen to bridge ICPC-2 and ICD-10 clinical concepts implicitly. Implementers preferring ICD-10 can substitute ICD-10 codes as keys, noting that ICPC-2 and ICD-10 have a many-to-many relationship requiring careful clinical review.

---

### Job 2: Resolve Drugs + RxNav Interaction Check

Using the RxCUI identifiers from the formulary mapping, the workflow calls the **NIH RxNav API** (`rxnav.nlm.nih.gov`) with all drug RxCUIs for the prescription.

RxNav returns known drug-drug interaction pairs with clinical descriptions. For example, for Warfarin + Doxycycline it returns the interaction description noting that Doxycycline can increase Warfarin's anticoagulant effect.

If fewer than two drugs are resolved (e.g. single-drug prescription), the interaction check is skipped.

---

### Job 3: Fetch OpenFDA Labels

For each resolved drug, the workflow calls the **OpenFDA API** (`api.fda.gov`) using the RxCUI to retrieve the FDA-approved drug label. The following sections are extracted:

- **Warnings** — general clinical warnings
- **Contraindications** — absolute contraindications
- **Drug interactions** — known interactions from the label
- **Adverse reactions** — common and serious ADRs
- **Pregnancy warning** — extracted from the `use_in_specific_populations` section
- **Paediatric warning** — extracted if patient is under 12
- **Geriatric warning** — extracted if patient is 65 or older

Population-specific warnings are only passed to the AI if relevant to the patient's context — a pregnancy warning is only included if the patient is pregnant, for example.

---

### Job 4: AI Clinical Assessment (Claude)

All collected data is assembled into a structured clinical prompt and sent to **Claude Haiku** (Anthropic). The prompt includes:

- Full patient context (age group, pregnancy status, diagnosis)
- Drug information with FDA label excerpts
- RxNav interaction findings
- WHO AWaRe antibiotic classification for any antibiotics prescribed
- Explicit escalation criteria for the AI to apply

Claude is instructed to respond with a single valid JSON object — no narrative text — containing:

| Field | Description |
|---|---|
| `alert_level` | Critical / Warning / Advisory / None |
| `alert_summary` | One sentence — most important finding |
| `treatment_appropriateness` | Yes / Caution / No with explanation |
| `interaction_alert` | Drug-drug interaction findings |
| `contraindication_alert` | Absolute contraindications identified |
| `adr_warning` | Key adverse drug reactions, prefixed with PREGNANCY WARNING or AGE WARNING if applicable |
| `recommended_action` | Specific actionable instruction for the prescriber |
| `alternatives` | Alternative drugs from the formulary if needed |
| `confidence` | High / Medium / Low |
| `confidence_reason` | Explanation if confidence is not High |

**Escalation logic applied by Claude:**
- **Critical** — high-severity drug interaction, OR contraindicated drug in a pregnant/paediatric/elderly patient
- **Warning** — moderate interaction, contraindication, or WATCH-tier antibiotic for a viral indication
- **Advisory** — low-severity interaction, AWaRe stewardship concern, or non-first-line drug per WHO
- **None** — no safety issues identified

---

### Job 5: Write Drug Safety Alert to DHIS2

The structured assessment is written back to DHIS2 as a new **Drug Safety Alert** stage event linked to the same patient enrollment. All nine output fields are populated as data element values.

After a successful write, the workflow updates the original **Prescription event** by setting `Safety Check Completed = true`. This prevents the same prescription from being picked up in future polling cycles.

---

## End Result in DHIS2

The clinician or supervisor can open the patient's enrollment dashboard in DHIS2 and see a new **Drug Safety Alert** event alongside the prescription, containing:

- **Alert Level** — colour-coded severity (Critical shown prominently)
- **Alert Summary** — one-sentence clinical finding
- **Interaction Alert** — specific drug pairs and interaction descriptions
- **Contraindication Alert** — absolute contraindications for this patient
- **ADR Warning** — adverse drug reactions with pregnancy/age flags
- **Treatment Appropriateness** — overall prescription assessment
- **Recommended Action** — what the prescriber should do
- **Safety Check Date** — when the check was performed

The entire process from prescription completion to alert creation takes under 30 seconds.
