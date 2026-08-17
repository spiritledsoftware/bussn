# Domain Docs

This repo uses a multi-context domain-doc layout.

## Before exploring, read these

- **`CONTEXT-MAP.md`** at the repo root — follow it to each `CONTEXT.md` relevant to the work.
- **`docs/adr/`** — read system-wide ADRs relevant to the work.
- **Context-scoped `docs/adr/` directories** — read relevant ADRs beside each context.

If these files don't exist, proceed silently. The `/domain-modeling` skill creates them lazily when terms or decisions are resolved.

## File structure

```text
/
├── CONTEXT-MAP.md
├── docs/adr/
├── apps/
│   └── <context>/
│       ├── CONTEXT.md
│       └── docs/adr/
└── packages/
    └── <context>/
        ├── CONTEXT.md
        └── docs/adr/
```

`CONTEXT-MAP.md` is authoritative; contexts may live elsewhere when appropriate.

## Use the glossary's vocabulary

When output names a domain concept, use the term defined in its `CONTEXT.md`. Don't drift to synonyms the glossary explicitly avoids.

If a required concept isn't in the glossary, reconsider whether it belongs or note the gap for `/domain-modeling`.

## Flag ADR conflicts

If output contradicts an existing ADR, surface it explicitly rather than silently overriding it.
