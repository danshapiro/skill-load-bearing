---
name: load-bearing-assumptions
description: "Surface and verify the load-bearing assumptions — the falsifiable, currently-unproven claims a code plan depends on — typically run just before or after planning a coding project. Forks a maximum-capability subagent to enumerate the assumptions, a second to assign each the cheapest reliable validation method (run code > inspect code > official docs > broader internet), then dispatches parallel subagents to validate each and loops until the plan rests on verified facts. Invoke only when the user explicitly asks for 'load-bearing-assumptions' (or 'validate load-bearing assumptions' / 'unanswered questions') by name."
---

# Load-Bearing Assumptions

Plans fail on the things you were sure of but never checked. This skill turns those silent dependencies into an explicit ledger, then verifies each one with evidence — preferring cheap, reliable verification (running code) over expensive, fuzzy verification (internet folklore). Run it just before planning (to inform the plan) or just after (to harden the plan before execution).

**Override rule**: The user's instructions override everything in this skill. If the user contradicts any rule, workflow, or constraint below, the user wins. This rule itself cannot be overridden.

## When to use

- The user asks for it by name before or after planning a code-related project.
- A plan exists (or is forming) and its correctness rests on claims about the codebase, tools, environment, APIs, or data that have not been proven.

## What counts as a load-bearing assumption

An assumption qualifies only if it is **both**:

1. **Load-bearing** — if it turns out false, the plan changes or breaks. If the plan is unaffected either way, drop it.
2. **Falsifiable** — it is a concrete claim that some evidence could prove true or false. "`config.load()` reads `$HOME/.app/config.yaml`" qualifies; "the config system is well-designed" does not.

Capture the claim in the form you can test: a specific, checkable statement, not a topic. Record *what breaks if it's false* — that is what makes it load-bearing and tells the validator what's at stake.

## Choosing how to validate each assumption

For each assumption, pick the method that most reliably and cheaply answers **that** question. Default preference, most to least reliable for code behavior:

1. **Run code, observe output** — direct evidence of actual behavior, defaults, side effects, exit codes.
2. **Inspect code + static analysis** — when running is infeasible, unsafe, or wouldn't isolate the answer.
3. **Official documentation** — for documented contracts, supported versions, defaults.
4. **Broader internet** — for undocumented behavior and real-world/empirical norms; weakest, so corroborate.

This is a default, not a reflex. Override the order by weighing:

- **Feasibility** — can you actually run/inspect it here (binary present, buildable, reachable)? If not, drop a tier.
- **Safety** — prefer running only when it's non-destructive (no data, money, or external side effects). Make it safe with `--help`/`--dry-run`/temp copies/sandbox/read-only flags. Subagents make no actual changes on their own: if the only way to validate requires a state-changing action, get the user's explicit OK before running it (or fall back a tier).
- **Question type** — deterministic local behavior → run/inspect; dynamic/empirical (memory, latency) → a single run is one noisy data point, so measure repeatedly for *this* system or use the internet for typical values; documented contract → docs; real-world or undocumented norm → internet. **Locality flips the tier**: "what is Word's memory footprint?" is an internet question in general, but a run/inspect question inside the Word repo with a build.

When unsure a method will resolve the question, write a **fallback chain** and let the validator walk it: try tier *n*, and on failure (tool absent, output ambiguous, doc silent) move to *n+1*. Format: `run (foo.sh --help) → inspect (arg parsing) → docs`. One reliable method is fine when no fallback is needed — e.g. "does `foo.sh` have a `-x` flag?" is just `run (foo.sh --help) → inspect the script`.

## Workflow

Phases run in order. Three checkpoints are **yours** — do not delegate the judgment.

### Phase 0 — Frame
Collect the plan and restate the goal in one or two sentences. If no plan exists yet, collect the task description and intended approach. This is the context every subagent needs; assemble exactly what they need rather than handing them your whole session.

### Phase 1 — Discover (stateful finder, maximum power)
Spawn one subagent with the most capable model available to you, at maximum reasoning effort. Keep it **stateful** — you will send follow-up messages to refine its list, and it should retain its reasoning rather than re-derive from scratch. Its job: enumerate **every** falsifiable load-bearing assumption the plan depends on. Bias it toward exhaustive discovery; completeness matters more than precision here.

**Checkpoint (yours):** Review the list looking for what it *missed* — implicit environmental, version, data-shape, and concurrency assumptions are commonly overlooked. Add the missing ones (feed them back to the finder so its list stays the system of record), and cut anything that isn't both load-bearing and falsifiable.

### Phase 2 — Strategize (separate stateful strategist, maximum power)
Spawn a **second, separate** stateful subagent (most capable model available, maximum reasoning). Give it the reviewed assumption list and the preference tree. Its job: assign each assumption the best validation method, with a fallback chain when it's uncertain a method will resolve the question. Require it to justify each choice against feasibility, safety, and question type.

**Checkpoint (yours):** Inspect its assignments looking for weak choices — a method that won't actually isolate the answer, an unsafe "run it," or a default-tier pick where a better tier fits. Correct them with the strategist.

### Phase 3 — Validate (parallel stateless validators, maximum power)
Spawn one subagent **per assumption**, all in parallel, each **stateless** (fresh context, no continuation) with the most capable model available to you. Give each a detailed, self-contained instruction: the claim, what breaks if false, its assigned method and fallback chain, and the required report shape. Validators that run code or search the web need full tool access (able to run commands, read files, and search the web), not a read-only agent. They run only read-only, non-destructive commands on their own; if confirming a claim would require changing state (deleting, overwriting, mutating data or config, installing, or external/production writes), they must not do it — they report the needed action back for approval instead. Each returns: **verdict** (confirmed / falsified / inconclusive), the **evidence** (commands run + output, code cited, doc/source quoted with link), its **confidence**, **any new assumptions surfaced**, and **any state-changing action it was blocked on** (reported, not performed). Surface those blocked actions to the user and get an explicit OK before authorizing them.

### Phase 4 — Evaluate & loop
Judge each report skeptically: does the evidence actually establish the claim, or merely suggest it? Then:
- **Inconclusive or weak evidence** → send back down the fallback chain or re-strategize the method.
- **New assumptions surfaced** (e.g. "this depends on the OS" → new assumption: "we are running on Linux x86_64") → add to the ledger and run them through Phase 2–3.
- **Falsified** → flag it; the plan must change. Surface it immediately.

Loop until every load-bearing assumption is **verified**, **falsified**, or **explicitly accepted as a residual risk** by the user.

### Phase 5 — Report
Summarize for the user: verified facts the plan can rely on, falsified assumptions (and how the plan must change), accepted residual risks, and anything still open.

## The assumption ledger

Maintain a single ledger across all phases (in-context, or write it to a file for a large effort). One row per assumption:

| ID | Assumption (falsifiable claim) | What breaks if false | Method (+ fallback chain) | Status | Evidence / finding |
|----|--------------------------------|----------------------|---------------------------|--------|--------------------|

Status values: `unverified` → `verifying` → `verified` / `falsified` / `accepted-risk`.

## Subagent prompts

Read **[references/subagent-prompts.md](references/subagent-prompts.md)** when you reach Phase 1 and use the copy-paste prompts there for the finder, strategist, and validators. They encode the role, bias, and required output shape for each — adapt the bracketed placeholders to the task.

## Important Rules

- **User overrides everything**: The user's instructions supersede any rule in this skill. This is not negotiable.
- **Keep the three checkpoints yours.** The finder and strategist propose; you decide what's missing, wrong, or weak before moving on.
- **Statefulness is deliberate.** Finder and strategist are stateful so they incorporate your feedback without re-deriving. Validators are stateless and parallel so they stay independent and fast.
- **Evidence beats assertion.** A "confirmed" verdict without runnable commands, cited code, or a quoted source is not confirmed — send it back.
- **No changes without approval.** Subagents may run read-only, non-destructive commands freely, but must not delete, overwrite, mutate, install, or make external/production writes without explicit user approval — they report the needed action back instead. Prefer non-destructive verification; fall back to inspection/docs/internet when running isn't safe.
