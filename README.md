# loops -- a dev-loop engine for Claude Code

Multi-hour autonomous build runs on your codebase, with the model driving and
you sitting at the contract boundary instead of inside the loop.

Three roles, three contexts, one on-disk contract:

- **PLANNER** -- writes the spec AND a standalone contract: `contract.md`
  (testable assertions), `feature_list.json` (machine-readable status),
  `progress.md` (append-only log, crash-resume). Never touches code.
- **GENERATOR** -- implements the next unchecked contract item. Forbidden from
  grading its own work.
- **CRITIC** -- an adversarial evaluator in its OWN context, told from message
  one: "the code is broken, prove it." It reads the diff, runs your real test
  suite, and grades every assertion. Defaults to FAIL on uncertainty, because a
  false PASS is the expensive error.

The loop body: generate -> pre-verify (cheap deterministic checks, zero model
spend) -> critic grades -> retry on FAIL, advance on PASS -> stop when the
contract is all-green or a hard stop trips (max iterations, no-progress, or a
token/$ ceiling). You approve the contract up front and get paged when
something trips. That is the whole human job.

## Quick start (let Claude do it)

The fastest setup is no setup:

    git clone https://github.com/calibrate-your-business/loops
    cd loops
    claude

Then prompt:

    Read the README and set up my project.

Claude will follow the section below, interview you about your codebase, and
walk you through a gated first run.

## Setup (for Claude)

You are setting up the loop engine for the user's project. Do these steps in
order; ask before assuming.

1. **Interview the user.** You need:
   - the target repo path, and the branch loops should work on
   - the FULL test-suite command (one command, non-interactive, exits nonzero
     on failure -- e.g. `npm test`, `pytest`, `./tools/run-tests`)
   - cheap pre-verify checks (formatter, compiler, linter -- whatever catches
     failures without a model in the loop)
   - which of their existing skills/conventions do planning and implementation
     (if they have none, plain Claude Code with a written spec works; note that
     in the profile)
   - objective or hybrid mode: objective when tests are the whole rubric;
     hybrid when part of the work is subjective (copy, UI) and needs a
     weighted taste rubric graded alongside the tests
   - optional memory: a brain instance, a markdown directory of principles, or
     nothing (see "Composable" below)

2. **Write `profiles/<their-project>.md`.** Profiles are gitignored on purpose
   -- they are private, per-project bindings; every fork writes its own. Do
   not un-ignore the directory. The shape:

       # profile: <project>

       planner:        <their planning skill, or "spec-first with plain Claude">
       generator:      <their build skill, or "plain Claude Code, TDD">
       verify:         <the FULL suite command -- never a subset>
       rubric_context: <brain context / principles dir, or "none">
       mode:           objective | hybrid
       repo:           <absolute path to the target repo>
       model:          <strong tier, for real runs>
       test_model:     <cheap tier, ONLY for testing the harness mechanics>

       pre_verify:
         - <formatter check>
         - <build/compile check>
         - <lint / static checks>

   Add two more sections when the project has them: the principles the critic
   grades against, and a reinvention catalog (existing primitives the critic
   should FAIL new duplicates of). See `contract/CONTRACT-FORMAT.md` for how
   those become gradable assertions.

3. **Generate the first contract.** Copy `contract/templates/` into the target
   repo (a `docs/loops/` or similar directory on the working branch) and fill
   them in from the user's spec: `contract.md`, `feature_list.json`,
   `progress.md`. Every assertion's `check:` must be falsifiable -- a named
   test, a command, or a greppable fact that could actually go red. Flag any
   assertion that just restates its claim; a contract that cannot fail the
   generator has not constrained anything.

4. **Dry-run one iteration, human-gated.** With the gate ON (the default): run
   `skills/run-loop` on the smallest contract item, using `test_model` if the
   user wants to verify the plumbing cheaply first. Confirm the generator
   produces a diff, pre-verify fires, the critic returns a structured verdict,
   and `progress.md` gets its append-only block. Then pause for the user's
   "go" -- that pause is the gate working.

5. **Explain the hard stops and how to tune them.** Defaults, all set per run:
   - `max_iters` per contract item: 20
   - no-progress: 2 consecutive iterations with no diff or no newly-passing
     assertion
   - budget ceiling: a token/$ cap per session (default ~2M tokens)

   Any stop pauses the loop and surfaces `progress.md` plus the blocking
   assertion. Advise starting tight (low max_iters, low ceiling, gate on every
   item) and loosening only as trust builds -- the end state is a human who
   intervenes only when the CONTRACT is wrong, never when the build is.

## What you actually need (honesty section)

- **Claude Code CLI.** The roles are Claude Code skills; the critic runs as a
  subagent in its own context. No CLI, no loop.
- **A real, runnable test suite.** The critic is exactly as good as the verify
  command you give it. One command, full suite, deterministic, nonzero on
  failure. If your tests are flaky or thin, the loop will confidently ship
  whatever slips through them -- fix the suite before trusting long runs.
- **Discipline about contract quality.** The contract is the interface between
  you and the machine. Vague assertions ("works correctly") produce vague
  passes. The run-loop skill flags non-falsifiable checks before building, but
  it cannot invent rigor you did not put in.

## What is in the box

    loops-spec.md               the full design (read this next)
    contract/CONTRACT-FORMAT.md the contract file formats + conformance manifest
    contract/templates/         contract.md / feature_list.json / progress.md
    skills/run-loop/            the orchestrator: loop body, hard stops, gate
    skills/critic/              the adversarial evaluator: four gates, structured verdict
    profiles/                   gitignored -- your private per-project bindings

## Composable, not coupled

The memory/rulebook layer is OPTIONAL. The loop runs fine with nothing behind
`rubric_context` -- the critic then grades correctness (your tests) and the
contract's own assertions. Add memory when you want more:

- **A brain instance** ([calibrate-your-business/brain](https://github.com/calibrate-your-business/brain))
  -- the critic grades diffs against your principle pages and a "do not
  reinvent" catalog, and run outcomes write back, so run N+1 knows what run N
  learned.
- **Any markdown directory** -- point `rubric_context` at a folder of
  principles; the critic reads what governs the touched files.
- **Nothing** -- objective mode, tests are the rubric.

Works well with:

- [brain](https://github.com/calibrate-your-business/brain) -- memory +
  rulebook engine; makes conformance gradable and runs cumulative.
- [automations](https://github.com/calibrate-your-business/automations) --
  schedule recurring loop runs instead of launching them by hand.
- [x-bookmarks](https://github.com/calibrate-your-business/x-bookmarks) -- an
  example repo instrumented for loop runs.

## Why an adversarial critic

Because the failure mode of long autonomous runs is not broken code -- tests
catch that. It is code that passes every test while duplicating a helper that
already existed, violating a written principle, or "verifying" a slice of the
suite instead of the whole thing. The critic exists to be the reader the
generator skipped: it re-runs the full suite itself, re-derives every claim in
the generator's manifest from the files on disk, and greps for reinvented
primitives. It grades the files, never the narration.

## License

MIT -- Copyright (c) 2026 Calibrate Your Business. See [LICENSE](LICENSE).
