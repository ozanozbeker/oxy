# Domain docs

This repo is single-context: one `CONTEXT.md` and one `docs/adr/`, both at the root.

## Before exploring, read these

- **`CONTEXT.md`** at the repo root.
- **`docs/adr/`**: read the ADRs about the code you are about to change.

If a file is missing, **proceed silently**. Do not flag its absence or suggest creating it. `/domain-modeling` creates each file when a term or decision is first resolved. `/grill-with-docs` and `/improve-codebase-architecture` call it.

## Use the glossary's vocabulary

Name each domain concept with the term `CONTEXT.md` defines. This applies to issue titles, refactor proposals, hypotheses and test names. Do not switch to a synonym the glossary lists as avoided.

If the glossary lacks the concept, you are either inventing a term or finding a gap. Reconsider an invented term. Note a gap for `/domain-modeling`.

## Flag ADR conflicts

If your output contradicts an existing ADR, say so explicitly instead of overriding it silently:

> _Contradicts ADR-0007 (title), but worth reopening because…_
