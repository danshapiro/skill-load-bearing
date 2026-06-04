# Load Bearing

You know how your LLM likes to talk about 'load bearing' assumptions? Well it's time they bear some loads. This surfaces and verifies the claims a code plan depends on that haven't been tested yet. Run it just before planning (to inform the plan) or just after (to harden it before execution).

This takes about 30 min for a meaty spec (can be much longer or shorter), plus however many tokens parallel subagents can digest in said 30 minutes.

## Install

This is a simple skill — no code, just a `SKILL.md` and a reference doc — so install it the
normal way for your agent: drop the `load-bearing-assumptions/` folder into your agent's
skills directory (e.g. `~/.claude/skills/` for Claude Code, `~/.agents/skills/` for Codex).
Then invoke it by name.

## In practice

Here's a few ways I've used it in the last 24 hours.

### [freshell](https://github.com/danshapiro/freshell) — improving resume when switching tabs

First I used Superpowers to write a plan, then:

> "Use the load bearing skill to test this plan."

Verdict: **not safe to implement as written.**

- **Falsified** — a progress cursor the plan trusted as "rendered through here" actually
  advanced when output was merely *enqueued* → a warm resume would silently skip unseen
  content; fixed by tracking a true rendered high-water mark.
- **Falsified** — replay data staged as a per-client array copy would multiply memory under
  realistic defaults → switched to a cursor into a bounded shared queue.

### [msgvault](https://github.com/wesm/msgvault) — a scheduled sync job sharing an embedded database

This time I just talked about what the problem and solution were going to be, and said:

> "Run the load bearing skill on that plan."

Nothing falsified — the plan was sound — but verification hardened it:

- **Confirmed the root cause** with live read-only queries plus source inspection: the job
  opened its own writable DB handle, and closing it triggered a destructive checkpoint.
- **Confirmed necessity** — a planned new helper was genuinely required, since no existing
  call did the job.
- **Surfaced a concurrency trap** — marking a run complete didn't check it was still running,
  so a nested run could clobber another's status → motivated a defensive guard and test.
- **Flagged deployment drift** on the live host as a preflight check, not a plan blocker.

### OAuth scope design for Nanoclaw

Another one that started with Superpowers:

> "Use the load bearing skill on the spec."

- **Falsified** — a scope assumed "draft-only, can't send" could in fact send → the
  draft/send boundary had to move into the broker layer instead of relying on the scope.
- **Falsified** — a narrow "app-created-files-only" storage scope couldn't write into a
  pre-existing user folder → redesigned so the agent creates its own folder, avoiding a
  high-privilege scope at consent time.
- **Caught** — a managed service the plan named had been discontinued and replaced → added
  the new provisioning step and credential before committing.
- **Separated out** — org-level facts only the user's admin console could confirm, flagged
  as residual risks rather than reported as verified.

### A [nanoclaw](https://github.com/danshapiro/nanoclaw-glowforge-slack) deployment system

> "Use the load bearing skill on this spec."

- **Verified by running** the plan's own embedded tests against fake binaries and temp
  fixtures, rather than trusting the prose.
- **Found a silent under-scope failure** — when the deployed commit isn't in the local
  checkout (diverged, force-pushed, or shallow clone), the diff silently falls back to the
  tip commit → a partial rollout that still reports success, exactly the failure the tool
  exists to prevent. Fix: fail loudly instead.

The recurring win across runs is the "**X was assumed equivalent to Y**" class, where
inspection proves they diverge ("accepted" ≠ "rendered"; "creates drafts" ≠ "can't send";
"narrow scope" ≠ "reaches existing data"). It consistently preferred cheap, reliable checks —
running the plan's own tests or live read-only queries — over documentation or folklore, and
kept externally-verifiable facts, user-only residual risks, and deployment drift in separate
buckets.


## Layout

```
load-bearing-assumptions/
├── SKILL.md                         # workflow + when to use
└── references/
    └── subagent-prompts.md          # copy-paste prompts for finder/strategist/validators
```
