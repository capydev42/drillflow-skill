# DrillFlow — format reference & agent skill

> **Generated. Do not edit here.** Every file in this repository is copied from
> the DrillFlow source repository on each release; changes made here are
> overwritten by the next sync. Please open issues rather than pull requests.

[DrillFlow](https://drillflow.de) is a local-first, web-based diagram editor
built around **drill-down**: a node can own a whole sub-diagram, and you
navigate *into* it as its own canvas, through arbitrarily many levels. This
repository holds everything needed to generate DrillFlow diagrams
programmatically — the format specification, a JSON Schema, a Claude Skill and
worked examples.

The editor itself is closed-source; this repository is the public, documented
part of it.

## Contents

| Path | What it is |
|---|---|
| `SPEC.md` | The canonical document-format reference. |
| `schema/drillflow-document.schema.json` | JSON Schema (draft 2020-12) for validating a generated document. |
| `SKILL.md` | A Claude Skill that teaches an agent to generate DrillFlow diagrams. |
| `examples/` | A minimal document, a Mermaid flowchart with nested subflows, and its JSON equivalent. |

Canonical, always-current copies live on the site — prefer these URLs when
something needs to fetch at runtime:

- <https://drillflow.de/en/format/> — the spec, rendered
- <https://drillflow.de/schema/drillflow-document.schema.json> — the schema
  (this URL is also the schema's `$id`)
- <https://drillflow.de/skills/drillflow-diagrams/SKILL.md> — the skill
- <https://drillflow.de/llms.txt> — index for LLMs

## Quick start

The short version: write a Mermaid flowchart, put each level in a `subgraph`,
and import it at <https://app.drillflow.de> via **☰ menu → Import → Mermaid…**.
Every subgraph becomes a real drill-down subflow.

```
flowchart TD
  A[Take order] --> B{In stock?}
  B -->|yes| C[Ship]

  subgraph A[Take order]
    A1[Check customer] --> A2[Validate items]
  end
```

For colours, layers, images or a shareable link, generate native JSON instead —
see `SPEC.md`.

## Using the skill

Copy `SKILL.md` (and `examples/`) into your agent's skills directory — for
Claude Code that is `.claude/skills/drillflow-diagrams/`.

## Licence

MIT — see `LICENSE`. This covers the documentation, schema, skill and examples
in this repository, not the DrillFlow application.

## Issues

Format questions and bug reports are welcome in this repository's issue
tracker, or by mail to <support@drillflow.de>.
