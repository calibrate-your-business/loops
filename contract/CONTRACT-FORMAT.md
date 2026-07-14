# Contract format -- the gradable target (LOOPS.md III/IV)

The contract is what the CRITIC grades against. It lives in the TARGET project
(on its branch), not in loops. Three files + a per-item manifest; crash-resume
from these three files alone.

## contract.md -- the testable assertions

Grouped by feature item. Each assertion has an ID, the claim, and HOW to check it
(a test name, a command, or a greppable fact). CONFORMANCE assertions are
first-class, alongside correctness -- they are the ones tests miss.

**Every assertion's `check:` MUST be falsifiable** -- it names a test, a command,
or a greppable fact that could actually go RED. A `check:` that only restates the
claim (nothing that could fail) is INVALID. A contract that can't fail you hasn't
constrained anything. (run-loop's CONTRACT-SANITY step flags these before building.)

    ## item: <id> -- <title>

    ### correctness
    - [C1] <claim> -- check: <test name / command / grep>

    ### conformance (graded against the project's PRINCIPLES, not tests)
    - [K1] conforms to <[[principle]]> -- check: <the prohibition it must not violate>
    - [K2] reuses <existing primitive@loc> instead of reinventing -- check: <symbol>

    ### provenance / auth
    - [P1] <provenance / source / RLS claim> -- check: <test / grep>

## feature_list.json -- machine-readable status (orchestration + resume)

    {
      "target": "<branch>",
      "verify": "<command>",
      "rubric_context": "<principles path>",
      "items": [
        {"id":"...","title":"...","status":"todo|doing|done|blocked",
         "assertions":["C1","K1","P1"], "iters":0}
      ]
    }

## progress.md -- append-only run log (LOOPS.md VII trace)

One block per iteration: timestamp, item, generator summary, critic verdict +
evidence, decision (retry | advance | stop). NEVER rewrite; only append. This is
the debugging surface when a verdict looks wrong (grep it, do not re-run blind).

## conformance manifest -- the generator declares, the critic verifies

Every generator turn ENDS with a manifest. The critic checks it claim-by-claim;
an unverifiable or false claim is a FAIL. This turns "did you read the docs" into
a gradable artifact.

    principles_consulted: [<[[principle]]> -> <how this change conforms>]
    primitives_reused:    [<symbol@location>]
    primitives_new:       [<symbol> -> <why NO existing equivalent (critic re-checks)>]
