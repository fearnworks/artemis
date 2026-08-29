# Persona skills

## Status

Implemented.

## Problem

Personas need curated reference playbooks (skills) beyond their fixed
instructions, but the bot must not gain arbitrary runtime skill discovery.
A deployment should decide exactly which approved skills each persona may
load, and the model should reach that content through the harness's standard
skill machinery rather than a bespoke loader, parser, or extra file-reading
tool.

## Scope

This protocol owns:

- an optional curated `skills` list on a persona profile, containing exact
  bundled skill directory names approved in source review
- resolution of those names to bundled skill directories and handoff to PI's
  stock skill loading (`DefaultResourceLoader` `additionalSkillPaths`)
- the built-in `read` tool, enabled only for personas that declare skills,
  because PI's skill system loads full skill content on demand through `read`
- a loud post-reload assertion that PI loaded every declared skill
- one bundled skill, `asd-ste100`, declared by the `wartermis` profile

It does not change Discord authorization, memory tools, GitHub tools, HSVAI
tools, provider selection, persistence, or response delivery. It adds no
custom skill parsing, no custom prompt section, and no skill-scoped custom
tool.

## Observable behavior

A persona with no `skills` list (default `generic`, and `artemis`) behaves
exactly as before: no skill paths, no `read` tool, no skills section in the
system prompt. A persona that declares skills (`wartermis` with
`asd-ste100`) gets the skill's name and description injected into the system
prompt by PI in the Agent Skills XML format, and the built-in `read` tool is
enabled so the model can load the full `SKILL.md` when a task matches
(PI's progressive-disclosure design). The `/skill:asd-ste100` command
expansion also becomes available through the PI session. Skill content ships
in the image at `/app/skills`; skill text is reviewed application content,
never user-provided.

## Contracts and data flow

```text
PersonaProfile.skills (curated names)
  -> personaSkillPaths: <cwd>/skills/<name> (throws at initialize if missing)
  -> DefaultResourceLoader additionalSkillPaths (noSkills stays true)
  -> loader.reload() -> assertPersonaSkillsLoaded (declared set == loaded set)
  -> PI system prompt skills section (requires builtin read tool)
  -> model reads SKILL.md on demand via read
```

Ordering and coupling invariants:

- Skill paths are resolved once in `initialize()`, so `checkHealth` fails at
  boot for a broken deployment instead of on the first Discord message.
- The `read` tool entry is derived from the resolved paths list (enabled when
  the list is non-empty), not re-derived from the profile.
- `getResourceLoader` asserts after `reload()` that every declared name
  appears in `loader.getSkills().skills`; PI's own Agent Skills validation
  (frontmatter name/description rules) is therefore the single authority for
  "loadable". A directory without a valid `SKILL.md` passes the existence
  check but fails this assertion.
- PI only appends the skills section when the session's active tools include
  `read` (observed in `pi-coding-agent` `dist/core/system-prompt.js`); that
  internal condition could change between PI versions, in which case the
  failure mode is a visible missing skills listing with `read` still enabled.
- `noSkills: true` remains set, so no global, project, package, or settings
  discovery location is active; the curated paths are the only source.

## Configuration

No environment variables. The curated list lives in each persona profile
under `src/personas/` (`wartermis` declares `skills: ["asd-ste100"]`).
Skill bundles live under `skills/<name>/` in the repository and are copied to
`/app/skills` in the image by the Dockerfile. The runtime working directory
must remain the application root (`/app`), which is where the skills root and
PI's loader cwd are resolved from.

## Persistence

Skills are not persisted. The system prompt is rebuilt per conversation kind
and HSVAI corpus revision by the cached resource loaders; the curated list is
static per process. No SQLite schema change.

## Security and privacy

Skill files are reviewed application source, vendored with provenance notes
(`skills/asd-ste100/UPSTREAM.md`), and never user-provided. Enabling skills
adds PI's built-in `read` tool to the session, which can read files inside
the container filesystem; the container holds no secret files (credentials
are environment-provided), but operators should treat model-readable
container files as prompt-injection reachable. No skill content is sent
anywhere except the configured model provider as part of the prompt.

## Failure handling

- A declared skill directory that does not exist fails `initialize()` (and
  therefore `checkHealth`) at boot with the persona and skill named.
- A declared skill directory without a loadable `SKILL.md` fails the
  post-reload assertion with the persona and skill named.
- Personas without declared skills are unaffected; their sessions gain no
  `read` tool and their prompts gain no skills section.

## Verification

- `test/pi-gateway.test.ts` covers: the wartermis path list and `read` tool
  enablement, the no-skills persona staying unchanged, the loud failure for a
  missing skill directory, and the post-reload loaded-set assertion
  (`assertPersonaSkillsLoaded` through `piInternals`).
- `npm run check:design` and `npm run guardrail` remain the completion gates.

## References

- [Persona profiles](persona-profile.md)
- [Configurable model provider](model-provider.md)
- [Baseline design](baseline.md)
- [Clean-room rebuild guide](rebuild-guide.md)
- [Design document index](README.md)
- PI skill documentation: `node_modules/@earendil-works/pi-coding-agent/docs/skills.md`
  (Agent Skills standard, progressive disclosure, `read`-based on-demand loading)
