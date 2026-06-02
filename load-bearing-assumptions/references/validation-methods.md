# Choosing a Validation Method

How to pick the cheapest method that *reliably* answers a given assumption, and how to build a fallback chain when you're not sure one will resolve it.

## Contents
- [The four tiers](#the-four-tiers)
- [Override factors](#override-factors)
- [Safety guardrails for running code](#safety-guardrails-for-running-code)
- [Building a fallback chain](#building-a-fallback-chain)
- [Worked examples](#worked-examples)

## The four tiers

Default preference, most to least reliable for code behavior:

1. **Run code, observe output.** Direct evidence of what actually happens here, now. Strongest for behavior, defaults, side effects, exit codes, output shapes.
2. **Inspect code + static analysis.** Read the source, types, signatures, configs, schemas. Use when running is infeasible, unsafe, or wouldn't isolate the answer.
3. **Official documentation.** Authoritative for documented contracts, supported versions, defaults, deprecation. Beware docs that lag the installed version.
4. **Broader internet** (issues, forums, blogs, Q&A). For undocumented behavior and real-world/empirical norms. Treat as weakest: corroborate, prefer recent and authoritative, watch for version drift.

## Override factors

The tier order is a default. Override it by weighing:

- **Feasibility.** Can you actually run it? Is the binary/service/dependency present, buildable, and reachable in this environment? If not, drop to inspection or docs.
- **Safety.** Running must be non-destructive (see guardrails below). If the only way to "run" it risks data, money, or external side effects, prefer inspection/docs.
- **Question type — match the method to what kind of fact this is:**
  - *Deterministic behavior of the code/tool here* (does this flag exist, what does this function return, what's the default) → **run**, else **inspect**.
  - *Dynamic / empirical / environment-dependent* (memory footprint, latency, throughput, "how big does this usually get") → a single run is one noisy data point, not the answer. Measure repeatedly if the question is about *this* system; otherwise **internet** for typical real-world values.
  - *Documented contract or guarantee* (API stability, version support, thread-safety promise, rate-limit policy) → **docs** first; confirm with a run if the contract is locally testable.
  - *Real-world norm or undocumented gotcha* → **internet**, corroborated.
- **Locality flips the choice.** The same question changes tier with context: "Word's memory footprint" is an **internet** question in general, but a **run/inspect** question if you're inside the Word repo with a build.

## Safety guardrails for running code

Prefer running only when it is **feasible and unlikely to be destructive**. Before running, confirm it does not:
- delete, overwrite, or mutate files, databases, or state you care about;
- send irreversible external requests (payments, emails, production writes, destructive API calls);
- consume meaningful cost or rate budget;
- require credentials or network you shouldn't exercise.

Make it safe when you can: use `--help`/`--dry-run`/`--version`, a throwaway temp dir, a copy of the data, a sandbox, or a read-only flag. If you can't make it safe, fall back a tier and say why.

## Building a fallback chain

When you're not certain a method will resolve the question, write the assignment as an ordered chain and let the validator walk it: try tier *n*, and on failure (tool absent, build broken, output ambiguous, doc silent) proceed to *n+1*.

Format: `run (foo.sh --help) → inspect (parse arg handling) → docs → internet`.

A chain is not mandatory — a question with one obviously reliable method gets one method. Add fallbacks precisely where availability or conclusiveness is in doubt.

## Worked examples

**"Does `foo.sh` accept a `-x` flag?"** Deterministic, local, cheap, safe to probe.
Chain: `run (foo.sh --help, or foo.sh -x on a no-op/temp input) → inspect (read the script's arg parsing) → docs`. Running wins; the script is the ground truth for its own flags.

**"What is Microsoft Word's memory footprint?"** Dynamic and environment-dependent. Running gives one machine's number, not the general answer; official docs won't quantify it.
Chain (general context): `internet (real-world measurements, corroborated) → docs (system requirements as a floor)`.
Chain (inside the Word repo with a build): `run (launch and measure RSS, repeated) → inspect`. Locality flips the tier.

**"Does our API return HTTP 429 on rate-limit?"** A contract that's locally testable.
Chain: `docs (our API spec) → run (hit the endpoint past the limit in a safe/staging context) → inspect (the rate-limit middleware)`. Avoid hammering production — that's the safety guardrail.

**"Is library `X` thread-safe for concurrent writes?"** A documented guarantee; empirically confirming absence-of-bug is hard.
Chain: `docs (the library's concurrency contract) → inspect (its locking in source) → internet (known issues)`. Don't trust a single passing run to prove thread-safety.
