# Setup Guide

## Prerequisites

- OpenFn Lightning account — [app.openfn.org](https://app.openfn.org)
- DHIS2 instance (v2.38+) with the Primary Health Record programme metadata
- Anthropic/Other LLM API key — [console.anthropic.com](https://console.anthropic.com)

---

## Step 1: Import DHIS2 Metadata

Import the programme metadata into your DHIS2 instance. The programme must include:

### Prescription Stage (input) — data elements required:

| Data Element | UID | Value Type |
|---|---|---|
| Diagnosis (ICPC-2) | `VRwZifj1OUr` | Text |
| Drug 1 | `TUpvPA4MLld` | Text |
| Drug 2 | `ETIWseOtnkj` | Text |
| Drug 3 | `AUdKW6pGigu` | Text |
| Pregnant | `QetEHInoxjr` | Text |

### Drug Safety Alert Stage (output) — data elements required:

| Data Element | UID | Value Type |
|---|---|---|
| Alert Level | `yYC03FbRzgg` | Text (option set) |
| Alert Summary | `KPLPSTV0XZn` | Long text |
| Recommended Action | `V59caCKDySM` | Long text |
| Safety Check Date | `QVPNMC9k5DW` | Date |
| Treatment Appropriateness | `JmS2VgfpSlG` | Long text |
| Interaction Alert | `O3pjWHqRadJ` | Long text |
| Contraindication Alert | `AZFoP6efYEd` | Long text |
| ADR Warning | `MsJEBMLYKgO` | Long text |
| Source Prescription Event | `dW6xOGoNWj7` | Text (hidden — for deduplication) |

> ⚠️ The `Source Prescription Event` data element must be added to the Drug Safety Alert stage but should be hidden from the data entry form. It is used internally for deduplication and should not be visible to clinicians.

### Tracked Entity Attributes required:

| Attribute | UID |
|---|---|
| Age | `rTTZqMtvAy8` |
| Patient ID | `fzunsCCOgyY` |

---

## Step 2: Create OpenFn Credentials

In OpenFn Lightning → Settings → Credentials, create two credentials:

### DHIS2 Credential
- **Name:** `DHIS2` (or any name)
- **Type:** DHIS2
- **Fields:**
  - Host URL: `https://your-dhis2-instance.org`
  - Username: `your-admin-username`
  - Password: `your-password`

> ⚠️ Use plain password authentication. Do NOT use a Personal Access Token (PAT) — the DHIS2 adaptor's `http.post('tracker', ...)` operation does not support PAT auth correctly.

### Claude API Credential
- **Name:** `Claude-API`
- **Type:** Claude
- **API Key:** `sk-ant-api03-YOUR_KEY`

> The Claude adaptor credential type handles authentication automatically. No Raw JSON credential is needed.

---

## Step 3: Create the Workflow

1. In OpenFn Lightning, go to **Workflows → New Workflow**
2. Set the trigger to **Cron** with expression `*/5 * * * *`
3. Add 5 jobs in sequence
4. Paste the code from each job file in `workflows/cron/`
5. Attach credentials to the correct steps:

| Step | Adaptor | Credential |
|------|---------|-----------|
| Fetch Recent Prescriptions | `@openfn/language-dhis2@latest` | DHIS2 |
| Resolve Drugs and RxNav Check | `@openfn/language-common@latest` | *(none)* |
| Fetch OpenFDA Labels | `@openfn/language-common@latest` | *(none)* |
| Claude Safety Assessment | `@openfn/language-claude@latest` | Claude-API |
| Write to DHIS2 Tracker | `@openfn/language-dhis2@latest` | DHIS2 |

> ⚠️ Jobs 02 and 03 must have **no credential** attached. The `http.get` calls to RxNav and OpenFDA are unauthenticated public API calls — attaching a credential to these steps breaks them.

---

## Step 4: Test the Workflow

1. Complete a prescription event in DHIS2 Tracker with a known high-risk combination:
   - Diagnosis: `W78` (Pregnancy)
   - Drug 1: `EA104 - WARFARIN 5MG`
   - Drug 2: `AA64 - DOXYCYCLINE 100MG`
   - Pregnant: Yes

2. Immediately run the workflow manually in OpenFn with input `{}`

3. Check the patient's Tracker record in DHIS2 — a Drug Safety Alert event should appear with **Critical** alert level within 30 seconds

4. Run the workflow again immediately — it should fetch 0 events and stop cleanly, confirming deduplication is working correctly

---

## Step 5: Enable Automatic Running

Enable the cron trigger in the OpenFn workflow editor. The workflow will now run automatically every 5 minutes.

---

## Adapting to Your DHIS2 Instance

To use this workflow with a different DHIS2 programme:

1. Update all UIDs in `job-01-fetch-prescriptions.js` to match your programme and data elements
2. Update the Drug Safety Alert data element UIDs in `job-05-write-to-dhis2.js`
3. Update the drug formulary in `job-02-resolve-drugs-rxnav.js` to match your local drug list
4. Create the Drug Safety Alert stage in your DHIS2 programme with the required data elements including `Source Prescription Event`
5. Update the ICPC-2 diagnosis mapping if your programme uses different diagnosis codes

---

## Troubleshooting

| Issue | Solution |
|-------|---------|
| 401 Unauthorized on DHIS2 | Check credential — use plain password, not PAT |
| `require is not defined` | Remove any `require('axios')` calls — use `http.get` with a plain URL string instead |
| Claude returning fallback values | Check Claude-API credential is attached to the Claude Safety Assessment step |
| No credential on jobs 02/03 | Remove any credential from Resolve Drugs and Fetch OpenFDA steps |
| Duplicate alerts | Deduplication uses Source Prescription Event UID — check `dW6xOGoNWj7` is in the Drug Safety Alert stage |
| 0 events fetched | Prescription may be older than 5 minutes — temporarily change `POLL_DURATION` to `'1h'` for testing |
| 405 Method Not Allowed on events update | Expected — DHIS2 v40 does not support PATCH on completed events. Deduplication is handled via Source Prescription Event UID instead |
| `prompt()` error: messages.0.content required | `state.userPrompt` is undefined — check job 04 step 1 returns `userPrompt: 'skip'` when no patient |
