# Domain docs

Use this repo's single-context domain documentation when exploring or changing its domain language and decisions.

## Before exploring

- Read `GLOSSARY.md` at the repo root if it exists.
- Read relevant decisions in `docs/adr/` if that directory exists.
- If these files do not exist, proceed without flagging their absence or suggesting that they be created upfront. Create them when terms or decisions are resolved.

## Use the glossary's vocabulary

When naming a domain concept in an issue, proposal, hypothesis, or test, use the term defined in `GLOSSARY.md`. If a needed concept is not covered, reconsider whether the project uses that language or note the gap for domain modeling.

## Flag ADR conflicts

If a proposed change contradicts an existing ADR, surface the conflict explicitly rather than silently overriding the decision.
