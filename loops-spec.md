# loops -- the dev-loop engine (LOOPS.md, applied)

Canonical design for the loop harness. ASCII-only.

## Intent

Run Karpathy's LOOPS.md pattern -- a PLANNER, a GENERATOR, and an adversarial
CRITIC arguing over an on-disk CONTRACT, with the brain as shared memory -- so
multi-hour build work runs with the model driving and a human only at the
contract boundary. Project-agnostic; the critic + orchestration are the new
parts (planner/generator reuse the target project's existing skills). First
target: sw-go's next MCP features -- the PLATFORM MCP (for app-builders) and the
DOMAIN MCP (for the Ctrl-K converse-and-act menu) -- on a `platform-8.5` branch
forked off `8.4`. These are mechanical (the underlying systems/APIs already
exist; the work is ASSEMBLING them), which makes them the ideal proving ground
for the conformance critic: the #1 risk is reinventing primitives that exist,
which is exactly what the critic prevents. The `bats` backbone already exists, so
there is no new verifier to build. (The family calendar app is a later target.)

## Architecture context (5 repos)

`loops` is a SYSTEM, peer to `brain`. It USES the brain (`$BRAIN_DB` via
`brain/db/query.py`) as memory + planning rulebook, and OPERATES ON a target project
(cyb's products). It does NOT replace the project's planner/generator, hold data
(`brain-db`), or hold business code (`cyb`).

## The three roles (LOOPS.md II -- separate roles, separate contexts)

- **PLANNER** -- the target project's existing planning skills (sw-go:
  brainstorming + writing-plans + planning-work). Produces the spec AND a
  standalone contract. Never touches code.
- **GENERATOR** -- the target project's existing build skill (sw-go:
  executing-plans, TDD). Implements the next unchecked contract item. FORBIDDEN
  from grading its own work.
- **CRITIC** (loops provides -- THE MISSING ROLE) -- a subagent in its OWN
  context, told from message 1 "the work is broken, prove it." Reads the diff,
  runs the project's verification command, grades against the contract + the
  principle files at the profile's `rubric_context` path. Never the maker.

## Per-project: thin PROFILES, not bespoke trinities

Every loop run binds three roles to a project -- but you do NOT hand-build three
new skills per project. You build the harness ONCE and bind mostly-existing
parts per project via a thin **loop profile** (config, not code):

- **PLANNER + GENERATOR = the project's OWN existing skills.** Reused, never
  rebuilt by loops. (sw-go: writing-plans -> executing-plans. GTM www/ads:
  writing-initiatives -> sw-marketing-site / google-ads-build.) If a project
  lacks one, build it as a NORMAL project skill; the loop just points at it.
- **CRITIC = ONE generic skill loops provides, parameterized per project.** Its
  machinery is identical everywhere (own context, prove-it, grade vs contract,
  principles as rubric); only three INPUTS change: the verify command, the
  `rubric_context` principles path, the contract. Config, not a new skill.
- **The only genuinely new per-domain work is the VERIFIER BACKBONE the critic
  plugs into, in ~2 flavors, reused across projects:**
  - OBJECTIVE (code/functionality): a test/build command -- sw-go `bats`, iOS
    `XCTest`, www `playwright`/lighthouse. Per domain, reused.
  - SUBJECTIVE (creative): a weighted TASTE rubric in the brain -- ad copy,
    marketing copy. Per creative domain, reused.

So the cost is NOT 3 x N. It is: 1 harness (once) + a handful of verifiers (per
domain, reused) + N tiny profiles (config) + reuse of each project's existing
planner/generator. A `profile` is roughly:

    project: sw-go
    planner: writing-plans               # the project's own skill
    generator: executing-plans           # the project's own skill
    verify: ./tools/run-tests            # the objective backbone
    rubric_context: docs/principles/     # the principles dir the critic grades against
    knowledge_context: schedule-wrangler # the brain context for DISCOVER + write-back
    mode: objective                      # objective (tests) | subjective (taste rubric)

## Conformance, not just correctness (the critic's highest-value job)

the owner's top frustration with the current workflow: agents fail to read the
principles, fail to use existing skills, and reinvent primitives that already
exist. These are INVISIBLE TO TESTS -- code passes every test while violating a
principle or duplicating a helper. So the critic's rubric is NOT the test suite;
it is the project's PRINCIPLES -- the dir at the profile's `rubric_context` path,
versioned with the code it governs. Two generic critic capabilities
(parameterized by the profile):

- **Conformance grading.** The critic reads the principle files at
  `rubric_context` that govern the files the diff touches, and checks the diff
  against each principle's PROHIBITIONS (sw-go: platform-domain-split,
  typed-metadata, binding-rules, ...). Violations FAIL even when
  tests pass.
- **Adversarial reinvention search.** For every NEW symbol the diff introduces,
  the critic greps the codebase + the profile's "DO NOT REINVENT" catalog
  for an existing equivalent; finding one FAILS the item
  with the existing primitive's location. The critic does the reuse-search the
  generator skipped.

And the loop makes grounding NON-OPTIONAL (the critic is the backstop, not the
only line of defense):

- **Grounding is a CONTRACT item** (gradable): "conforms to the applicable
  binding rules (cite them)"; "introduces no duplicate primitive (list what you
  checked)." Skipping the principles = a red contract item, not a vibe.
- **PULL, not push.** The generator's DISCOVER step queries the brain
  (`knowledge_context`) for the domain knowledge and existing patterns tied to
  THESE files and seeds the work with them -- targeted retrieval beats "please
  read all five docs" (which agents skip).
- **Conformance manifest.** The generator declares principles-consulted /
  primitives-reused / new-primitives-justified; the critic VERIFIES each claim.
  Turns "did you read the docs" into an explicit, gradable artifact.

Net: today grounding is push + optional + unverified; the loop makes it pull +
required + adversarially verified. The two-field split is what makes this
possible: the principles at `rubric_context` (per-project, versioned with the
code) serve as the critic's rubric; the brain (`knowledge_context`) carries the
domain knowledge that seeds discovery and cannot fail a diff.

## The contract (LOOPS.md III/IV -- on disk, the gradable target)

Split out of the spec; three files per loop run, in the target project:

- `contract.md` -- the checklist of testable assertions (the ~27). The boundary
  the critic grades against. The spec is the boundary; the contract is what gets
  graded.
- `feature_list.json` -- machine-readable item list + status (for resume +
  orchestration).
- `progress.md` -- append-only run log (what was tried, what the critic said).
  Crash-resume from these three files.

## The loop body (LOOPS.md I/V)

generate next unchecked item -> PRE-VERIFY (cheap deterministic floor) -> critic
grades -> on FAIL: append to `progress.md`, restart generator on that item -> on
PASS: mark item done, advance -> stop when the contract is all-green OR a hard stop
trips. The harness re-invokes the model (`/loop` or `/goal` style); the model is
the subroutine, the loop is the driver. PRE-VERIFY runs the profile's `pre_verify`
checks (sw-go: `gofmt`/`go build`/`go vet`/arch guards) BEFORE the critic; any
failure is an instant fail that retries the generator with ZERO model spend --
never a critic subagent on what `gofmt`/`vet` would catch. The critic runs only on
a green floor. Generator/critic Agent calls take a `model`: `test_model` (Haiku)
to test the loop's PLUMBING, the strong `model` for real work.

## Hard stops (DEFAULTS -- tune per run; the owner sets the budget ceiling)

- max-iterations: 20 per contract item.
- no-progress: 2 consecutive iterations with no diff / no new passing assertion.
- budget ceiling: a configurable token/$ cap per session (DEFAULT ~2M tokens).
- Any hard stop -> pause and surface to the human with `progress.md`.

Cost floor: the per-iteration PRE-VERIFY stage (the profile's cheap deterministic
checks -- `gofmt`/`build`/`vet`/arch guards) runs BEFORE the critic; a failure
there is an instant fail that retries the generator at ZERO model spend, so no
critic subagent is ever spent on what those checks would catch. Model spend is
further tiered: `test_model` (Haiku) for testing the harness mechanics, the strong
`model` for real loop runs.

## Human gate (LOOPS.md V -- DEFAULT: human-gated, loosen with trust)

DEFAULT (conservative): the loop runs ONE contract item to green, then pauses for
a human "go" before the next. As trust builds, loosen toward the LOOPS.md ideal:
insert a human only when the CONTRACT itself is wrong, never when the build is.
[Loosening is the owner's call; the default is the safe start.]

## The brain in the loop (memory as the outer loop)

- DISCOVER: planner/generator query the brain (the profile's `knowledge_context`:
  `BRAIN_DB=~/Claude/brain-db python3 ~/Claude/brain/db/query.py search "..."
  --context <knowledge_context>`) for business/domain knowledge + codebase
  patterns, instead of re-deriving the project every run.
- RULEBOOK (planning): the brain informs WHAT to build and why; it is never the
  critic's conformance rubric -- that is the principles dir at the profile's
  `rubric_context` path, versioned with the code it governs. The brain cannot
  fail a diff.
- WRITE-BACK: loop outcomes -> raw -> recompiled wiki, so run N+1 knows what run
  N learned.

## Taste rubric (LOOPS.md VI -- only where work is subjective)

Where the work has objective tests (sw-go backend), the test suite IS the rubric
-- no taste rubric needed. For UI/UX (the family app's iOS surface) and for GTM
creative work later, add a weighted rubric (design / craft / ...) calibrated on
reference examples, held in the brain.

## Trace-reading (LOOPS.md VII)

The harness pipes each agent's output to a run log; on a bad verdict, the log is
the debugging surface (grep where judgment diverged). This is the one idea worth
stealing from `loopy` (its "receipt" pattern).

## Build phases

1. `contract/` -- the `contract.md` / `feature_list.json` / `progress.md` format
   + templates + a splitter (spec -> contract).
2. `skills/critic/` -- the adversarial critic skill (own context, prove-it,
   grades vs contract + principles).
3. `run/` -- the loop orchestrator (the body + hard stops + heartbeat + trace
   logging).
4. brain wiring -- the DISCOVER / RULEBOOK / WRITE-BACK glue.
5. FIRST TARGET -- fork `platform-8.5` off `platform-8.4`, point the loop at the
   two MCP features (platform MCP for app-builders; domain MCP for the Ctrl-K
   converse-and-act menu), human-gated. Verifier = the existing `bats` backbone.
   Start with the most mechanical, highest-value sub-feature. (The family
   calendar app + its iOS XCTest backbone is a later target.)

## What loops does NOT do

- Does not replace the project's planner/generator (reuse sw-go's skills).
- Does not adopt `loopy` (mine its receipt/trace idea only).
- Does not hold data (`brain-db` does) or business code (`cyb` does).

## Open items

- Per-run budget ceiling number (default ~2M tokens) -- tune on the first run.
- When to loosen the human gate to contract-failure-only (the owner's trust call).
- iOS verification backbone specifics for the family app (phase 5).
- RESOLVED -- model choice: the profile now carries two tiers -- `model` (strong)
  for real loop runs, `test_model` (`claude-haiku-4-5`) for testing the harness
  mechanics. Haiku verifies the plumbing moves; the critic's real verdicts need the
  strong tier.
