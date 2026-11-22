# Labeling Guide — Patient Notes AI

Version 2 · 2026-11-22

This guide defines how every condition in a clinical note is labeled. The note generator, the answer key (`ground_truth/`), the LLM prompt and the evals all follow it. If a definition changes, update this file first, then the prompt and the generator, then reprocess.

## 1. What we label

For each note, every **condition mentioned** gets one label:

```json
{"condition": "<standard name>", "status": "<one of 5 statuses>"}
```

- One label per condition per note.
- `condition` is always the **standard name** (section 3), never the abbreviation written in the note.
- Everything else in the note (medications, labs, vitals, allergies) is out of scope for now.

## 2. Statuses

| Status | Meaning | Typical phrases |
| --- | --- | --- |
| `current` | Active now; found or treated at this visit | presents with, diagnosed with, currently being treated for |
| `pre_existing` | Diagnosed before **and** still present | known case of |
| `history` | In the past; may be resolved | history of, past medical history significant for |
| `operated` | Treated surgically in the past | operated for, s/p surgery for, status post resection of |
| `ruled_out` | The patient does not have it | negative for, no evidence of, ruled out |

### Rules

1. **Surgical-only rule:** `operated` is allowed only for surgical conditions: `lung tumor`, `appendicitis`, `gallstones`.
2. **"Known case of" = `pre_existing`.** It is not `history` (the condition is still present) and not `current` (it was not newly found at this visit).
3. **"s/p" means "status post"** (after a procedure) and maps to `operated`.
4. **Negation wins.** "Ruled out", "negative for" and "no evidence of" are always `ruled_out`, whatever else the sentence says.
5. **Abbreviations and casing do not matter.** "HTN", "htn" and "high blood pressure" are all `hypertension`.

## 3. Standard condition names and accepted aliases

| Standard name | Aliases seen in notes | Surgical |
| --- | --- | --- |
| pneumonia | pneumonia, PNA | no |
| type 2 diabetes | type 2 diabetes, T2DM, DM2, diabetes mellitus type 2 | no |
| hypertension | hypertension, HTN, high blood pressure | no |
| asthma | asthma | no |
| lung tumor | lung tumor, lung mass, pulmonary neoplasm | yes |
| chronic kidney disease | chronic kidney disease, CKD | no |
| malaria | malaria | no |
| urinary tract infection | urinary tract infection, UTI | no |
| migraine | migraine | no |
| copd | COPD, chronic obstructive pulmonary disease | no |
| fatty liver | fatty liver, hepatic steatosis, NAFLD | no |
| appendicitis | appendicitis, acute appendicitis | yes |
| gallstones | gallstones, cholelithiasis | yes |

The same mapping lives in the Delta table `workspace.patient_notes.ref_condition_aliases`. To add an alias, add a row there and here.

## 4. Examples

| Note text | Labels |
| --- | --- |
| Known case of HTN. Presents with PNA. | hypertension: pre_existing; pneumonia: current |
| History of malaria. Negative for asthma. | malaria: history; asthma: ruled_out |
| Status post resection of lung mass. | lung tumor: operated |
| S/p surgery for cholelithiasis. No evidence of UTI. | gallstones: operated; urinary tract infection: ruled_out |

## 5. Change log

| Version | Date | Change | Why |
| --- | --- | --- | --- |
| 1 | 2026-11-21 | 3 statuses: current, history, ruled_out | Initial design |
| 2 | 2026-11-22 | Added `pre_existing` and `operated`; surgical-only rule; added appendicitis and gallstones | Evals showed the LLM read "known case of" as current while the key said history. Clinical review: it means a pre-existing, still-present disease. Surgical history needed its own status. |