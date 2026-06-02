# skill-load-bearing

An Agent Skill for surfacing and verifying **load-bearing assumptions** — the falsifiable,
currently-unproven claims a code plan depends on. Run it just before planning (to inform the
plan) or just after (to harden it before execution).

The skill discovers assumptions with a max-capability subagent, assigns each the cheapest
reliable validation method (run code > inspect code > official docs > broader internet),
dispatches parallel subagents to validate them, and loops until the plan rests on verified facts.

## Layout

```
load-bearing-assumptions/
├── SKILL.md                         # workflow + when to use
└── references/
    ├── validation-methods.md        # the preference tree, safety, worked examples
    └── subagent-prompts.md          # copy-paste prompts for finder/strategist/validators
```

## Install

Copy the skill folder into your skills directory (a copy, not a symlink, so edits in this
repo don't go live until you deliberately re-copy):

```bash
cp -r load-bearing-assumptions ~/.codex/skills/load-bearing-assumptions
```

Then invoke it by name (e.g. "use load-bearing-assumptions on this plan").
