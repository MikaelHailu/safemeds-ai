# Setup Guide

## Prerequisites

- OpenFn Lightning account — [app.openfn.org](https://app.openfn.org)
- DHIS2 instance (v2.38+) with the Primary Health Record programme metadata
- Anthropic API key — [console.anthropic.com](https://console.anthropic.com)

---

## Step 1: Import DHIS2 Metadata

Import the programme metadata into your DHIS2 instance. The programme must include:

- Prescription stage with all required data elements
- Drug Safety Alert stage with all output data elements


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

### Claude API Credential or LLM of choice
- **Name:** `Claude-API`
- **Type:** Claude
- **API Key:** `sk-ant-api03-YOUR_KEY`

---

## Step 3: Create the Workflow

1. In OpenFn Lightning, go to **Workflows → New Workflow**
2. Set the trigger to **Cron** with expression `*/5 * * * *`
3. Add 5 jobs in sequence
4. Paste the code from each job file in `workflows/cron/`
5. Attach credentials to the correct steps:

| Step | Credential |
|------|-----------|
| Fetch Recent Prescriptions | DHIS2 |
| Resolve Drugs and RxNav Check | *(none)* |
| Fetch OpenFDA Labels | *(none)* |
| Claude Safety Assessment | Claude-API |
| Write to DHIS2 Tracker | DHIS2 |

> ⚠️ Jobs 02 and 03 must have **no credential** attached. The `http.get` calls to RxNav and OpenFDA are unauthenticated public API calls — attaching a credential breaks them.

---

## Step 4: Test the Workflow

1. Complete a prescription event in DHIS2 Tracker with:
   - Diagnosis: W78 (Pregnancy)
   - Drug 1: `EA104 - WARFARIN 5MG`
   - Drug 2: `AA64 - DOXYCYCLINE 100MG`
   - Pregnant: Yes

2. Immediately run the workflow manually in OpenFn with input `{}`

3. Check the patient's Tracker record in DHIS2 — a Drug Safety Alert event should appear with **Critical** alert level within 30 seconds

---

## Step 5: Enable Automatic Running

Enable the cron trigger in the OpenFn workflow editor. The workflow will now run automatically every 5 minutes.

---

## Adapting to Your DHIS2 Instance

To use this workflow with a different DHIS2 programme:

1. Update the UIDs in `job-01-fetch-prescriptions.js` to match your programme
2. Update the data element UIDs in `job-05-write-to-dhis2.js`
3. Update the drug formulary in `job-02-resolve-drugs-rxnav.js` to match your local drug list
4. Create the Drug Safety Alert stage in your DHIS2 programme with the required data elements

---

