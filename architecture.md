# Architecture

## System Overview

The DHIS2 AI Drug Safety Checker is a fully automated workflow that runs on OpenFn Lightning. It polls DHIS2 every 5 minutes for new prescription events and generates AI-powered safety assessments written back to the patient's Tracker record.

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
│  ├── Deduplication check                         │
│  └── POST /api/tracker                           │
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
- **Auth:** None (public API, 1000 req/day unauthenticated)
- **Purpose:** FDA drug label data — contraindications, warnings, pregnancy info
- **Note:** Get a free API key at https://open.fda.gov/apis/authentication/ for higher limits

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
Given the webhook for events data is not currenly supported in DHIS2, the cron polling approach has been opted for as a more reliable approach without any DHIS2 server configuration.

### Separate steps for each job
Each job is a separate OpenFn step with its own adaptor and credential. This allows fine-grained credential control — jobs 02 and 03 have no credential so `http.get` calls to external public APIs are unauthenticated.

### Drug formulary as embedded JSON
The drug lookup table is embedded in job 02 rather than fetched from an external source. This avoids a network dependency and keeps resolution fast and reliable.

### Deduplication
Job 05 checks for prescriptions that have already been checked. This prevents duplicate alerts if the workflow runs multiple times while a prescription is still within the poll window.
