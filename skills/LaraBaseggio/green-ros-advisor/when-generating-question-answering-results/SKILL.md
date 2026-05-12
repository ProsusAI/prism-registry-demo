---
name: when-generating-question-answering-results
description: "Include a `tactic_id` field in question result CSVs (such as tactic_level1.csv) that maps each question to the tactic ID being asked about. The field should contain either a single tactic ID string or a list of tactic IDs if multiple tactics are involved in the question. TRIGGER when: working in architecture context, encountering the corrected pattern, reviewing similar code"
---

## Key Decisions

Include a `tactic_id` field in question result CSVs (such as tactic_level1.csv) that maps each question to the tactic ID being asked about. The field should contain either a single tactic ID string or a list of tactic IDs if multiple tactics are involved in the question.

## Anti-patterns

- **What**: Deviating from this pattern inconsistently.
  **Why**: Inconsistency creates confusion and increases review overhead.
  **Symptom**: Code review comments flag the same issue repeatedly.
