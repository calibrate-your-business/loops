# loop profile: plan-docs (deliverable = a PLAN DOCUMENT, not software)

The thin binding for a loop whose output is a PLAN DOCUMENT. Unlike the
machine-local per-project profiles (host paths, corporate specifics), this one is
COMMITTED and shared: a plan run's rubric is the same everywhere, so the profile
is a house standard, not a private binding.

    deliverable:       plan-doc                 # the output is a plan, not code
    generator:         writing-plans-on-disk    # the target repo's plan-writing skill
                                                #   cyb monorepo: writing-plans-on-disk
                                                #   sw-go:        writing-plans
    verify:            plan-document-reviewer    # subjective mode: the reviewer rubric IS the gate (no test command)
    mode:              subjective                # a plan has no test suite; the rubric is taste + conformance
    rubric_context:    sw-go/.claude/skills/writing-plans/plan-document-reviewer-prompt.md
                                                # the FIXED house plan rubric that GRADES -- same for every plan run,
                                                # not a docs/principles dir (a plan is graded by the reviewer, not code principles)
    knowledge_context: <target project's brain context>
                                                # bound per run; INFORMS the plan-writing generator's DISCOVER; cannot fail a diff
    model:             opus                      # strong tier for real runs
    test_model:        haiku                      # harness-plumbing checks only

## KEY DISTINCTION -- a plan run is ONE item; the contract is the house standard

When the deliverable is a PLAN, the "contract" is NOT a bespoke per-plan
checklist -- it is the FIXED HOUSE PLAN STANDARD (the `pre_verify` lint plus the
`plan-document-reviewer` rubric below), the same rubric for every plan run. The
BUILD-run contract trio (`contract.md` / `feature_list.json` / `progress.md`) is
for multi-item build runs only. A plan run is ONE item -- the plan -- so there is
NO `feature_list.json`. Keep `progress.md` solely as the iteration trace (the
verdict each pass); nothing else.

## pre_verify (deterministic, zero model spend -- INSTANT FAIL, no critic)

The target repo's plan lint. Never spend a critic subagent on what these catch:

- NARRATION-MARKER scan -- a plan states the work, never the story of the document
  (rulings, reviews, amendments, status). FAIL on any marker (mirrors cyb
  `.claude/hooks/enforce-no-narration-plans.sh`): `per owner`, `owner ruling`,
  `ruled 20`, `amended 20`, `critic PASS`, `critic FAIL`, `critic-verified`,
  `WITHDRAWN`, `supersession blockquote`, `absorbed the critic`, `review pass`,
  `pending re-approval`. `README.md` and `OPEN-WORK.md` are exempt -- status rows
  live there, spec bodies are the target.
- ASCII-only markdown.
- REQUIRED sections present -- in particular a `## Constraints` section carrying a
  `Release model:` line.
- `## Open items` is EMPTY at approval (no unresolved question ships in a plan).
- Every `file:line` citation in the plan RESOLVES against the tree (grep loop:
  each `path:NN` reference exists and the file has at least NN lines).

## critic rubric (subjective -- taste + conformance, not a test command)

The plan-document-reviewer gates
(`sw-go/.claude/skills/writing-plans/plan-document-reviewer-prompt.md`):
completeness, task decomposition, buildability, principle conformance, and
signal-to-noise (strip authoring/process narration -- the pre_verify markers are
the floor; the reviewer catches the rest). PLUS:

- COHESION -- reads as one document; phases in dependency order; no section
  contradicts another.
- COVERAGE -- every goal/constraint in the intent has a phase or step; nothing
  named is left unplanned.
- REINVENTION -- the plan does not propose building what already exists (a skill,
  helper, or doc); it cites the existing thing. The reuse search the author
  skipped.
- DOC-CLAIMS-VS-CODE -- every "the code does X" and every `file:line` claim is
  verified against the actual tree, not asserted.

## human gate

The scratch -> committed promotion IS the loop's pause. A plan iterates in
`scratch/**/plans/` and graduates to committed `plans/` ONLY on the owner's
approval; the owner's promotion is the human gate. The loop never self-promotes.

## Hard stops (surface to the human)

- `## Open items` cannot be emptied without an owner decision (an unresolved
  question is the owner's, not the loop's).
- A reviewer FAIL that recurs after 3 generate iterations on one plan.
- A conformance/reinvention finding that would change the plan's scope (never
  decided unilaterally).
