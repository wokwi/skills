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

| Skill                                                          | Use it to                                                                                         |
| -------------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| [`find-memory-leaks`](skills/find-memory-leaks/SKILL.md)       | Catch slow leaks in ESP-IDF firmware: the simulator watches the heap while a scenario drives hours of use in seconds |
| [`test-lvgl-ui`](skills/test-lvgl-ui/SKILL.md)                 | Regression-test an LVGL (or any) display UI in CI with taps, swipes and pixel-exact screenshot compare |
| [`debug-esp32-firmware`](skills/debug-esp32-firmware/SKILL.md) | Decode a panic backtrace, reproduce the bug with scripted inputs, trace a bus to VCD, attach GDB without a probe |
| [`verify-esp-idf-build`](skills/verify-esp-idf-build/SKILL.md) | Run and check an ESP-IDF build with `idf.py wokwi` right after building, no devkit attached           |

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
