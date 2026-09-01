# Domain docs

## Before exploring

Read:

- `CONTEXT.md` at the repository root.
- `docs/adr/` entries relevant to the work.

If these paths do not exist, proceed silently. Domain-modeling skills create them when terminology or architectural decisions need to be recorded.

## Layout

This repository uses a single-context layout:

```
/
├── CONTEXT.md
├── docs/
│   └── adr/
└── src/
```

## Vocabulary

Use domain terms defined in `CONTEXT.md`. If a required concept is absent, reconsider whether the term belongs to the project or note the gap for domain modeling.

## ADR conflicts

Surface any conflict with an existing ADR instead of silently overriding it.
