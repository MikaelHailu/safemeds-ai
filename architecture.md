# Architecture

## System Overview

SafeMeds AI is a fully automated workflow that runs on OpenFn Lightning. It polls DHIS2 every 5 minutes for new prescription events and generates AI-powered safety assessments written back to the patient's Tracker record.

## Data Flow

```
┌─────────────────────────────────────────────────┐
│                  DHIS2 Tracker                   │
│                                                   │
│  Patient enrolled in Primary Health Record        │
│  Clinician completes Prescription stage           │
│  - Diagnosis (ICPC-2 code)                       │
│  - Drug 1, Drug 2, Drug 3 (local formulary)      │
│  - Pregnant (Yes/No)                             │
└───────────────────┬─────────────────────────────┘
                    │ Poll every 5 min
                    ▼
┌─────────────────────────────────────────────────┐
│              OpenFn Lightning                    │
│                                                   │
│  Job 01: Fetch Recent Prescriptions              │
│  ├── GET /api/events?lastUpdatedDuration=5m      │
│  ├── Deduplication: check Source Prescription    │
│  │   Event UID against existing Drug Safety      │
│  │   Alert events per TEI                        │
│  └── GET /api/trackedEntityInstances/{id}        │
│                    │                             │
│  Job 02: Resolve Drugs + RxNav                   │
│  ├── Formulary lookup (local JSON mapping)       │
│  └── GET rxnav.nlm.nih.gov/interaction/list      │
│                    │                             │
│  Job 03: Fetch OpenFDA Labels                    │
│  └── GET api.fda.gov/drug/label.json             │
│                    │                             │
│  Job 04: Claude AI Safety Assessment             │
│  ├── Build clinical prompt                       │
│  ├── prompt() → Claude Haiku                     │
│  └── Parse JSON response                         │
│                    │                             │
│  Job 05: Write Drug Safety Alert                 │
│  ├── POST /api/tracker (Drug Safety Alert event) │
│  └── Stores Source Prescription Event UID        │
│      for deduplication on next poll              │
└───────────────────┬─────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────┐
│                  DHIS2 Tracker                   │
│                                                   │
│  Drug Safety Alert stage created:                │
│  - Alert Level (Critical/Warning/Advisory/None)  │
│  - Alert Summary                                 │
│  - Recommended Action                            │
│  - Interaction Alert                             │
│  - Contraindication Alert                        │
│  - ADR Warning                                   │
│  - Treatment Appropriateness                     │
│  - Safety Check Date                             │
│  - Source Prescription Event (hidden, for dedup) │
└─────────────────────────────────────────────────┘
```

## External API Dependencies

### NIH RxNav
- **URL:** `https://rxnav.nlm.nih.gov/REST/interaction/list.json`
- **Auth:** None (public API)
- **Purpose:** Drug-drug interaction checking using RxCUI identifiers
- **Rate limit:** No hard limit — polite usage recommended

### OpenFDA
- **URL:** `https://api.fda.gov/drug/label.json`
- **Auth:** None (public API)
- **Purpose:** FDA drug label data — contraindications, warnings, pregnancy and age-specific info
- **Note:** Get a free API key at https://open.fda.gov/apis/authentication/ for higher request allowance

### Anthropic Claude (or preferred LLM)
- **Model:** `claude-haiku-4-5-20251001`
- **Auth:** API key via OpenFn credential
- **Purpose:** Clinical safety assessment and structured JSON output
- **Token usage:** ~1500 input + ~400 output tokens per assessment

## OpenFn Adaptors Used

| Job | Adaptor | Reason |
|-----|---------|--------|
| 01 | `@openfn/language-dhis2` | Native `get()` and `http.get()` for DHIS2 API |
| 02 | `@openfn/language-common` | `http.get()` for RxNav (no credential needed) |
| 03 | `@openfn/language-common` | `http.get()` for OpenFDA (no credential needed) |
| 04 | `@openfn/language-claude` | Native `prompt()` function with Claude credential |
| 05 | `@openfn/language-dhis2` | Native `http.post('tracker', ...)` for DHIS2 write |

## Key Design Decisions

### Cron polling vs webhook
DHIS2 program stage webhook notifications are not reliably supported in DHIS2 v40 — confirmed on multiple instances. The cron polling approach is more reliable and requires no DHIS2 server configuration beyond the standard programme setup.

### Separate steps for each job
Each job is a separate OpenFn step with its own adaptor and credential. This allows fine-grained credential control — jobs 02 and 03 have no credential so `http.get` calls to external public APIs are unauthenticated. Attaching a credential to these steps breaks the external API calls.

### Drug formulary as embedded JSON
The drug lookup table is embedded in job 02 rather than fetched from an external source. This avoids a network dependency and keeps resolution fast and reliable in low-connectivity environments.

### Deduplication via Source Prescription Event
Each Drug Safety Alert stores the UID of the prescription event that triggered it in a hidden data element (`Source Prescription Event`, UID: `dW6xOGoNWj7`). On each poll, job 01 checks whether a Drug Safety Alert already exists for that specific prescription event UID before processing it.

This approach correctly handles patients with multiple prescription stage events from different visits — each prescription is checked independently regardless of whether alerts exist for other prescriptions by the same patient.

### Why not update the prescription event directly?
An alternative deduplication approach would be to set a `Safety Check Completed` flag directly on the prescription event after processing. However, DHIS2 v40 does not support PATCH or POST updates to completed events via the legacy `/api/events` endpoint (returns 405 Method Not Allowed). The Source Prescription Event approach avoids this limitation entirely.
