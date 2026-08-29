# Vendored skill: asd-ste100

The `asd-ste100` skill applies ASD-STE100 controlled-language rules to text an agent or a
downstream system must parse without a human to resolve ambiguity. It is the first bundled
persona skill: the `wartermis` persona declares it, and the persona-skills contract in
`design/persona-skills.md` loads and exposes it.

| Field | Value |
|---|---|
| Source repository | `cfai/cfai-system` |
| Source path | `.agents/skills/asd-ste100/` |
| Source commit | `fdcf5cb682b8aa8d4ed486b9995d26e431df5a30` |
| Source commit date | 2026-08-26 |
| Skill version | 0.4.0 |
| Vendored | 2026-08-29 |

## Vendored paths

| Path | Source path |
|---|---|
| `SKILL.md` | `.agents/skills/asd-ste100/SKILL.md` |
| `references/writing-rules.md` | `.agents/skills/asd-ste100/references/writing-rules.md` |
| `examples/before-after.md` | `.agents/skills/asd-ste100/examples/before-after.md` |
| `UPSTREAM.md` | rewritten for this repository (provenance record) |

## Local patches

`SKILL.md`, `references/`, and `examples/` are byte-identical to the source. Only this
provenance file is rewritten, because the source repository's own `UPSTREAM.md` records a
different vendoring chain.

## Why this repository runs it

Persona skills are reviewed application content, not runtime or user-provided material.
The skill's scope — machine-parsed English such as tool descriptions, error strings, and
status text — matches the bot's technical replies, while its own rules exclude the
persona's creative banter.
