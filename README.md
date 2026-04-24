# SafeMeds AI

> AI-powered prescription safety checking for primary healthcare settings, integrated with DHIS2 Tracker via OpenFn.

---

## Overview

This project implements an automated drug safety checking workflow for primary healthcare clinics operating in resource-constrained settings. When a prescriber completes a prescription in DHIS2 Tracker, the system automatically:

1. Fetches the prescription event from DHIS2
2. Resolves drugs against a local formulary mapping
3. Checks drug-drug interactions via RxNav (NIH)
4. Fetches FDA label data including pregnancy and age warnings via OpenFDA
5. Runs a clinical safety assessment using an LLM
6. Writes a structured Drug Safety Alert back to the patient's DHIS2 Tracker record

The system is designed as a prototype for primary healthcare settings and can be adapted to any DHIS2 Tracker programme.

---

## Architecture

```
DHIS2 Tracker (Prescription Stage)
        │
        ▼ (cron poll every 5 min)
OpenFn Lightning
        │
        ├── Job 01: Fetch Recent Prescriptions (DHIS2 adaptor)
        ├── Job 02: Resolve Drugs + RxNav Interactions (language-common)
        ├── Job 03: Fetch OpenFDA Labels (language-common)
        ├── Job 04: AI Safety Assessment (language-claude)
        └── Job 05: Write Drug Safety Alert to DHIS2 (DHIS2 adaptor)
        │
        ▼
DHIS2 Tracker (Drug Safety Alert Stage)
```

---

## Features

- **Formulary-aware drug resolution** — maps local drug codes (e.g. `EA104 - WARFARIN 5MG`) to generic names and RxCUI identifiers
- **Drug-drug interaction checking** — via NIH RxNav API
- **FDA label data** — contraindications, warnings, pregnancy and age-specific warnings via OpenFDA
- **AI clinical assessment** — Claude Haiku generates structured safety alerts with alert levels (Critical / Warning / Advisory / None)
- **WHO AWaRe antibiotic stewardship** — integrated into the assessment prompt
- **ICPC-2 diagnosis mapping** — 27 common primary care diagnoses supported
- **Pregnancy and age-specific alerts** — special handling for paediatric, elderly, and pregnant patients
- **Deduplication** — prevents duplicate Drug Safety Alert events for the same prescription
- **Fully automated** — runs every 5 minutes via OpenFn cron trigger

---

## Alert Levels

| Level | Criteria |
|-------|----------|
| **Critical** | High-severity drug interaction, or contraindicated drug in pregnant/paediatric/elderly patient |
| **Warning** | Moderate interaction, contraindication, or WATCH antibiotic for viral indication |
| **Advisory** | Low-severity interaction, AWaRe stewardship concern, or non-first-line drug per WHO |
| **None** | No safety issues identified |

---

## DHIS2 Programme Structure

### Programme
- **Name:** Primary Health Record
- **UID:** `d4iemvqrYHO`

### Programme Stages

| Stage | UID | Purpose |
|-------|-----|---------|
| Prescription | `DjNzlXFgoxQ` | Clinician enters diagnosis and drugs |
| Drug Safety Alert | `Rxhkap8IQxZ` | AI-generated safety assessment |

### Key Data Elements

#### Prescription Stage (input)
| Data Element | UID |
|---|---|
| Diagnosis (ICPC-2) | `VRwZifj1OUr` |
| Pregnant | `QetEHInoxjr` |
| Drug 1 | `TUpvPA4MLld` |
| Drug 2 | `ETIWseOtnkj` |
| Drug 3 | `AUdKW6pGigu` |


#### Drug Safety Alert Stage (output)
| Data Element | UID |
|---|---|
| Alert Level | `yYC03FbRzgg` |
| Alert Summary | `KPLPSTV0XZn` |
| Recommended Action | `V59caCKDySM` |
| Safety Check Date | `QVPNMC9k5DW` |
| Treatment Appropriateness | `JmS2VgfpSlG` |
| Interaction Alert | `O3pjWHqRadJ` |
| Contraindication Alert | `AZFoP6efYEd` |
| ADR Warning | `MsJEBMLYKgO` |

#### Tracked Entity Attributes
| Attribute | UID |
|---|---|
| Age | `rTTZqMtvAy8` |
| Patient ID | `fzunsCCOgyY` |

---

## Prerequisites

- [OpenFn Lightning](https://app.openfn.org) account (free tier supports one workflow)
- DHIS2 instance (v2.38+) with the Primary Health Record programme
- Anthropic API key ([console.anthropic.com](https://console.anthropic.com))

---

## Setup

### 1. OpenFn Credentials

Create two credentials in OpenFn Lightning → Settings → Credentials:

**DHIS2 credential** (type: DHIS2)
```json
{
  "hostUrl": "https://your-dhis2-instance.org",
  "username": "your-username",
  "password": "your-password"
}
```

**Claude API credential** (type: Claude)
```json
{
  "apiKey": "sk-ant-api03-YOUR_KEY"
}
```

### 2. Import the Workflow

In OpenFn Lightning, create a new workflow and import `workflows/cron/workflow-import.yaml`.

Attach credentials to steps:
| Step | Credential |
|------|-----------|
| Fetch Recent Prescriptions | DHIS2 |
| Resolve Drugs and RxNav Check | *(none)* |
| Fetch OpenFDA Labels | *(none)* |
| Claude Safety Assessment | Claude API |
| Write to DHIS2 Tracker | DHIS2 |

### 3. Configure the Cron Schedule

The workflow is set to run every 5 minutes (`*/5 * * * *`). Enable the trigger in the workflow editor.

### 4. Test

Run the workflow manually with an empty input `{}`. If a prescription was completed in the last 5 minutes, it will be processed automatically.

---

## External APIs Used

| API | Purpose | Authentication |
|-----|---------|---------------|
| [NIH RxNav](https://rxnav.nlm.nih.gov) | Drug-drug interaction checking | None (public) |
| [OpenFDA](https://api.fda.gov) | FDA drug label data | None (public) |
| [Anthropic Claude](https://www.anthropic.com) | AI clinical safety assessment | API key |

---

## Formulary

The drug formulary mapping (`workflows/cron/drug-formulary-mapping.js`) covers ~60 drugs across common primary healthcare categories including antibiotics, cardiovascular drugs, antidiabetics, antiepileptics, and antidepressants. Each entry maps the local drug code to:

- Generic name
- RxCUI (NIH drug identifier)
- Route of administration
- Confidence level

---

## Scaling Considerations

This is a **prototype** implementation designed for clinic-level deployment. The architecture is intentionally simple to demonstrate the integration pattern. The following considerations apply when moving toward production use.

### Current Limitations

| Limitation | Impact |
|---|---|
| One patient processed per cron run | If multiple prescriptions are completed simultaneously, only the first is processed per 5-minute window |
| Sequential Claude API calls (~5s each) | OpenFn's 60s timeout limits processing to ~10 patients per run |
| Single workflow constraint (free tier) | Cannot run parallel workflows to handle concurrent prescriptions |
| No retry mechanism | If the Claude API or DHIS2 is temporarily unavailable, the prescription is skipped until the next poll window |

### External API Considerations

The three external APIs used each have characteristics worth considering for production deployment:

**NIH RxNav** — a public API maintained by the US National Library of Medicine with no authentication required. It has no hard rate limit but is intended for moderate usage. For high-volume deployments, consider caching interaction data locally using the RxNav bulk download.

**OpenFDA** — a public API with no authentication required for moderate use. For higher volume, a free API key is available at [open.fda.gov](https://open.fda.gov/apis/authentication/) and significantly increases the request allowance. For production deployments, FDA label data could be cached locally since it changes infrequently.

**Anthropic Claude** — requires a paid API key. The Claude Haiku model used in this prototype is optimised for speed and cost. For concurrent high-volume processing, consider upgrading to a higher API tier to avoid rate limiting. Each safety assessment uses approximately 1500 input tokens and 400 output tokens.

### Workarounds for Higher Volume

**Short term — process all events per run**
Change `patient: patients[0]` in job 01 to loop through all patients in the batch. Each 5-minute window will process all new prescriptions up to the 60s timeout limit.

**Medium term — parallel processing via webhook**
Use two OpenFn workflows: a lightweight cron job that fetches all new prescriptions and POSTs each one individually to a second workflow's webhook URL. The second workflow (the existing pipeline) then processes each prescription independently and in parallel. This approach largely addresses the concurrency limitation within OpenFn's platform constraints.

**Large scale — national programme deployment**
For deployment across hundreds of facilities processing thousands of prescriptions daily:
- Use OpenFn's enterprise concurrency features to run parallel workflow instances
- Move Claude API calls to a dedicated microservice with a queue (e.g. AWS SQS + Lambda)
- Use DHIS2's bulk analytics API rather than event polling for more efficient data retrieval
- Cache OpenFDA label data locally to eliminate repeated external calls

The `Safety Check Completed` flag (`SRgePliKEnM`) on each prescription event ensures no prescription is ever processed twice, regardless of the deployment scale chosen.

---

## Clinical Disclaimer

This tool is intended to **support** — not replace — clinical decision-making. All alerts should be reviewed by a qualified healthcare professional. The system uses publicly available drug safety data and AI-generated assessments which may not be complete or current.

---

## ⚠️ Prototype Disclaimer

SafeMeds AI is a prototype and has not been validated for clinical use.
This tool is intended to demonstrate the technical feasibility of AI-powered prescription safety checking integrated with DHIS2 Tracker. It should not be used in a live clinical environment without:

- Independent clinical validation of the AI assessment outputs
- Review and approval by qualified pharmacists or clinicians
- Assessment of data privacy and patient consent requirements under applicable national regulations
- Evaluation of third-party data processing implications (Anthropic Claude API, NIH RxNav, OpenFDA)
- Load and reliability testing appropriate to the deployment context

The AI-generated safety alerts are not a substitute for clinical judgement. Prescribers and supervisors must independently verify any recommendations before acting on them.
The authors accept no liability for clinical decisions made on the basis of outputs generated by this tool.

---

## License

Copyright (c) 2026 Mikael Hailu

---

## Author

Mikael Hailu
