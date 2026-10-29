# Wokwi agent skills

Skills that teach AI coding agents (Claude Code, Cursor, Codex, Copilot and others)
how to run, test, debug and profile embedded firmware on the [Wokwi](https://wokwi.com) simulator:
set up CI, run ESP-IDF projects from `idf.py`, script buttons and sensors, soak-test
for memory leaks. One folder per skill, each a `SKILL.md` in the open
[Agent Skills](https://agentskills.io/specification) format, so the same files work in
every agent that reads skills.

Agents can already write firmware. What they cannot do is find out whether it works
without a board on the desk. These skills close that loop.

## Skills

TODO

## Install

**Claude Code, as a plugin** (all skills, namespaced `/wokwi:<skill>`):

```text
/plugin marketplace add wokwi/skills
/plugin install wokwi@wokwi-skills
```

**Claude Code, one project, no plugin:** copy `skills/<name>/` into the repo's
`.claude/skills/` (or `~/.claude/skills/` for every project).

**Any other agent:** copy the folder to wherever that agent loads skills from, or paste
the `SKILL.md` body into the instructions file it reads (`AGENTS.md`, `.cursorrules`,
`copilot-instructions.md`). The files are plain Markdown with a small YAML header.

## License

MIT, see [LICENSE](LICENSE).
