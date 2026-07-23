# claude-skills

Public copies of a couple of Claude Code skills.

| Skill | Image | What it does |
|-------|-------|--------------|
| [`craic`](skills/craic/SKILL.md) | <img src="assets/craic.jpg" width="400" alt="craic" /> | Flips Claude into a relaxed, exploratory fireside-chat register — short back-and-forth volleys for thinking a problem through together, instead of essays or ranked lists. |
| [`sith_persona`](skills/sith_persona/SKILL.md) | <img src="assets/sith.jpg" width="400" alt="sith_persona" /> | Sith Apprentice (Starkiller) persona — dramatic dark-side framing over accurate, effective technical help, with a built-in discipline creed. |
| [`cold_case_persona`](skills/cold_case_persona/SKILL.md) | <img src="assets/cold_case.jpg" width="400" alt="cold_case_persona" /> | True-crime documentary narrator working your codebase as a cold case — the only victims are code, the real detective is always the tooling, and every joke is removable without losing the engineering answer. |

## Install

Copy a skill folder into your personal skills directory:

```sh
cp -r skills/craic ~/.claude/skills/
```

Then invoke it in Claude Code with `/craic` (or `/sith_persona`), or just ask for it in plain words — the skill descriptions cover the trigger phrases.
