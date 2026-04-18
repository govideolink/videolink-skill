# Videolink Agent Skill

Public mirror of the Videolink agent skill. Install into an agent (Claude
Code, Cursor, Codex, Gemini CLI, …) via:

```
npx skills add govideolink/videolink-skill
```

The live skill is served dynamically at
`https://api.govideolink.com/v1/skills/videolink/SKILL.md`. This repo
mirrors the current production snapshot. When the live skill bumps its
`CURRENT_SKILL_VERSION`, the Videolink CI pushes a new commit here.

## What's in here

- `SKILL.md` — recipe-shaped agent skill. Start here.
- `references/API.md` — full REST endpoint reference for agents that
  need endpoint-level detail beyond the recipes.
- `VERSION` — current version string (matches the frontmatter in
  `SKILL.md`).

## Source

This repo is a snapshot. Development happens in the private
[ScalableFactory/agendalink](https://github.com/ScalableFactory/agendalink)
monorepo.
