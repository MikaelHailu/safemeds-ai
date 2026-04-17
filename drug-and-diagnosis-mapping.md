# Drug Formulary and Diagnosis Mapping

## Overview

Two mapping tables are central to how the drug safety checker works:

1. **Drug Formulary Mapping** — translates local DHIS2 drug option codes into standardised generic names and NIH RxCUI identifiers
2. **ICPC-2 Diagnosis Mapping** — translates ICPC-2 diagnosis codes stored in DHIS2 into clinical terms used in the AI safety assessment prompt

Both mappings are embedded directly in Job 02 (`job-02-resolve-drugs-rxnav.js`) and also available as a standalone reference file (`drug-formulary-mapping.js`).

---

## Drug Formulary Mapping

### Why This Is Needed

DHIS2 stores drugs using local option set values like `EA104 - WARFARIN 5MG`. These codes are specific to the prototype programme and are not recognised by external APIs like RxNav or OpenFDA.

The formulary mapping converts these local codes to:
- **Generic name** (e.g. `WARFARIN`) — used in the Claude AI prompt
- **RxCUI** (e.g. `855288`) — the NIH RxNorm identifier used to query RxNav for interactions and OpenFDA for label data

### Resolution Strategy

The lookup uses a three-stage fallback:

1. **Exact name match** — looks up the full DHIS2 option string (e.g. `EA104 - WARFARIN 5MG`) in `BY_NAME`
2. **Case-insensitive match** — same lookup ignoring case
3. **Parsed generic fallback** — strips the code prefix and dose from the string, then looks up the generic name in `BY_PARSED`

### Coverage

| Category | Drugs |
|----------|-------|
| Antibiotics (ACCESS) | Amoxicillin, Metronidazole, Nitrofurantoin, Doxycycline, Co-trimoxazole |
| Antibiotics (WATCH) | Ciprofloxacin, Azithromycin, Ceftriaxone, Cefixime, Clarithromycin |
| Antibiotics (other) | Amikacin, Gentamicin, Benzathine penicillin |
| Cardiovascular | Furosemide, Hydrochlorothiazide, Digoxin, Amlodipine, Atenolol, Enalapril, Warfarin |
| Antidiabetics | Metformin |
| Gastrointestinal | Omeprazole |
| Antiepileptics | Carbamazepine |
| Antidepressants | Sertraline, Fluoxetine |

### Adding New Drugs

To add a drug to the formulary:

1. Find the exact option value as it appears in DHIS2 (copy from the option set)
2. Look up the RxCUI at [rxnav.nlm.nih.gov](https://rxnav.nlm.nih.gov/RxNormAPIs.html#uiLink:findRxcuiByString)
3. Add entries to both `BY_NAME` and `BY_PARSED` in `job-02-resolve-drugs-rxnav.js`

Example:
```javascript
// In BY_NAME:
"XX99 - METOPROLOL 50MG": { generic: "METOPROLOL", rxcui: "203834", confidence: "high", route: "ORAL" },

// In BY_PARSED:
"metoprolol": { generic: "METOPROLOL", rxcui: "203834", confidence: "high", route: "ORAL" },
```

---

## ICPC-2 Diagnosis Mapping

### Why This Is Needed

The specific DHIS2 programme setup stores diagnosis using ICPC-2 codes (e.g. `K86` for hypertension). Claude AI needs a human-readable diagnosis term to reason about drug appropriateness — for example, to identify that prescribing Warfarin for a patient with diagnosis W78 (pregnancy) is a critical safety issue.

### Relationship to ICD-10

The `term` field is also used to provide clinical context when interpreting FDA label data from OpenFDA, which references ICD-10 conditions in its contraindication and warning sections. The clinical terms chosen (e.g. "pregnancy", "hypertension", "epilepsy") are designed to match the language used in FDA labels, providing an implicit bridge between ICPC-2 and ICD-10 clinical concepts.

Implementers preferring to work with ICD-10 directly can replace the ICPC-2 codes with ICD-10 codes as keys in `ICPC_MAP`. Note however that ICPC-2 and ICD-10 have a many-to-many relationship — a single ICPC-2 code can map to multiple ICD-10 codes and vice versa — so a full crosswalk requires careful clinical review rather than simple one-to-one substitution. The WHO maintains a reference mapping at [who.int](https://www.who.int/standards/classifications).

### How It Is Used

The ICPC code is looked up in `ICPC_MAP` to extract:
- `term` — a clinical search term included in the AI prompt (e.g. `"hypertension"`)
- `label` — a short human-readable label shown in the alert summary (e.g. `"Hypertension"`)

The `term` field is designed to match the clinical language Claude AI understands from its training on medical literature.

### Coverage

| Chapter | Codes | Diagnoses |
|---------|-------|-----------|
| Cardiovascular (K) | K74, K75, K77, K86, K87 | Angina, MI, Heart failure, Hypertension |
| Respiratory (R) | R74, R75, R78, R81, R96 | URTI, Sinusitis, Bronchitis, Pneumonia, Asthma |
| Digestive (D) | D73, D84, D85 | Gastroenteritis, Peptic ulcer, Duodenal ulcer |
| Endocrine (T) | T89, T90 | Type 1 and Type 2 diabetes |
| Urinary (U) | U70, U71 | Pyelonephritis, UTI |
| Pregnancy (W) | W78, W84 | Pregnancy, High risk pregnancy |
| Female genital (X) | X74 | Vaginitis |
| Infectious (A) | A77, A78 | Malaria, Tuberculosis |
| Psychological (P) | P74, P76 | Anxiety, Depression |
| Neurological (N) | N87, N89 | Epilepsy, Migraine |
| Blood (B) | B80 | Anaemia |

### Adding New Diagnoses

To add a diagnosis:

1. Find the ICPC-2 code at [www.helsedirektoratet.no](https://www.helsedirektoratet.no/digitalisering-og-e-helse/helsefaglige-kodeverk/icpc/icpc-2e--english-version)
2. Add an entry to `ICPC_MAP` in `job-02-resolve-drugs-rxnav.js`

Example:
```javascript
"L84": { term: "back pain", label: "Low back pain" },
```

Choose a `term` that matches how the condition is described in clinical and pharmacological literature — this is what Claude AI uses to reason about drug appropriateness.

---

## Unmapped Drugs and Diagnoses

### Unmapped drugs
If a drug cannot be resolved (not in `BY_NAME` or `BY_PARSED`), it is logged as a warning and passed to Claude as `[NOT IN FORMULARY MAPPING]`. Claude will still attempt a safety assessment but will note the limitation in the confidence field.

### Unmapped diagnoses
If an ICPC code is not in `ICPC_MAP`, the raw code is passed to Claude (e.g. `ICPC-2: L84`). Claude may still recognise common codes from its training data, but the assessment quality may be lower.

Both cases are logged in the OpenFn run history for review.
