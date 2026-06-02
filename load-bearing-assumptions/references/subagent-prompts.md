# Subagent Prompts

Copy-paste prompts for the three subagent roles. Replace `[bracketed]` placeholders. Spawn all three with the most capable available model (currently Opus) and maximum reasoning effort.

## Contents
- [1. Finder (Phase 1 — stateful)](#1-finder-phase-1--stateful)
- [2. Strategist (Phase 2 — stateful)](#2-strategist-phase-2--stateful)
- [3. Validator (Phase 3 — stateless, one per assumption, in parallel)](#3-validator-phase-3--stateless-one-per-assumption-in-parallel)

Statefulness: spawn the **finder** and **strategist** once and continue the same conversation when you send review feedback, so they refine their own work instead of re-deriving it. Spawn each **validator** fresh — never continue one — and dispatch them in parallel.

---

## 1. Finder (Phase 1 — stateful)

```
You are finding the load-bearing assumptions in a plan. Think as hard as you can.

GOAL: [restate the goal in 1–2 sentences]

PLAN / INTENDED APPROACH:
[paste the plan or task + approach]

RELEVANT CONTEXT:
[codebase facts, environment, tools, APIs, data the plan touches — only what's needed]

A load-bearing assumption is a claim that is BOTH:
- load-bearing: if it's false, the plan changes or breaks, AND
- falsifiable: a concrete claim some evidence could prove true or false.

Find EVERY such assumption the plan silently depends on. Be exhaustive — completeness
matters more than precision; I will prune. Probe the easily-missed categories:
environment/OS/arch, tool availability and versions, API/library contracts and defaults,
data shape and volume, concurrency and ordering, permissions and auth, error/edge behavior,
and "this works the way I remember" beliefs.

For each, output a row:
- claim (the specific, testable statement — not a topic)
- what breaks if it's false
- why you think it's currently unproven

Exclude anything that isn't both load-bearing and falsifiable (opinions, aesthetics,
things that don't change the plan either way). Return the list only.
```

When you send additions/cuts back, keep it terse: "Add these: …. Remove these (not load-bearing/falsifiable): …. Return the updated list."

---

## 2. Strategist (Phase 2 — stateful)

```
You are assigning each assumption the best way to VALIDATE it. Think as hard as you can.

ASSUMPTIONS (reviewed list):
[paste the ledger claims + "what breaks if false"]

ENVIRONMENT: [OS/arch, what's installed/buildable/reachable, repo on hand, network/creds available]

Choose, per assumption, the method that most reliably and cheaply answers THAT question.
Default preference order:
  1. run code, observe output   2. inspect code + static analysis
  3. official documentation     4. broader internet

But this is a default, not a reflex. Override it by weighing:
- FEASIBILITY: can it actually be run/inspected here?
- SAFETY: prefer running only when feasible AND non-destructive (no data/money/external
  side effects; use --help/--dry-run/temp copies/sandbox to make it safe).
- QUESTION TYPE: deterministic local behavior → run/inspect; dynamic/empirical (footprint,
  latency) → measure repeatedly or use internet for typical values; documented contract → docs;
  real-world/undocumented norm → internet. Locality can flip the tier.

When you're not certain a method will resolve it, give an ordered FALLBACK CHAIN
(e.g. "run (foo.sh --help) → inspect (arg parsing) → docs"). One reliable method is fine
when no fallback is needed.

For each assumption output:
- chosen method (+ fallback chain)
- one-line justification citing feasibility / safety / question type
- the exact first action (command to run, file to inspect, doc/page to read, or query)
```

When you send corrections back: name the assumption, what's wrong with the choice (won't isolate the answer / unsafe to run / wrong tier), and ask for a revised assignment.

---

## 3. Validator (Phase 3 — stateless, one per assumption, in parallel)

Spawn one per assumption with full tools (general-purpose agent type — it may need to run commands, read files, and search the web). Fill in every placeholder; the validator has no other context.

```
Validate one assumption and report back. Think rigorously; do not rubber-stamp it.

ASSUMPTION (the claim to test): [exact falsifiable claim]
WHAT BREAKS IF FALSE: [stakes]
ASSIGNED METHOD + FALLBACK CHAIN: [e.g. run (foo.sh --help) → inspect (arg parsing) → docs]
CONTEXT NEEDED: [repo path, command, env, doc URL, anything required to act]
SAFETY: prefer running only if non-destructive; if the method is unsafe or infeasible,
move to the next link in the chain and say why.

Execute the chain: try the first method; on failure, ambiguity, or inconclusive output,
proceed to the next link. Reach a verdict only on evidence you actually gathered.

Return exactly:
- VERDICT: confirmed | falsified | inconclusive
- EVIDENCE: the proof — commands run + their output, code quoted with file:line, or
  doc/source quoted with a link. No evidence = not confirmed.
- CONFIDENCE: high | medium | low, with the reason.
- NEW ASSUMPTIONS SURFACED: any new falsifiable dependency you discovered (e.g. "the answer
  depends on the OS" → "we are running on [X]"), or "none".
```
