# Domain docs

How the engineering skills should consume this repository's domain documentation when exploring the codebase.

## Before exploring, read these

- `CONTEXT.md` at the repository root.
- `CONTEXT-MAP.md` at the repository root if it exists. It points to one `CONTEXT.md` per context. Read each one relevant to the topic.
- Relevant ADRs under `docs/adr/`. In multi-context repositories, also check `src/<context>/docs/adr/`.

If any of these files don't exist, proceed silently. Don't flag their absence or suggest creating them upfront. The `/domain-modeling` skill creates them when terms or decisions get resolved.

## File structure

This repository uses the single-context layout:

```text
/
├── CONTEXT.md
├── docs/adr/
│   ├── 0001-example-decision.md
│   └── 0002-another-decision.md
└── src/
```

## Use the glossary's vocabulary

When output names a domain concept in an issue title, proposal, hypothesis, or test name, use the term defined in `CONTEXT.md`. Don't drift to synonyms that the glossary explicitly avoids.

If the needed concept isn't in the glossary, reconsider whether the project uses that language. If it represents a real gap, note it for `/domain-modeling`.

## Flag ADR conflicts

If output contradicts an existing ADR, state the conflict rather than silently overriding it:

> _Contradicts ADR-0007 (event-sourced orders), but worth reopening because..._
