---
name: run-loop
description: The loop orchestrator. Use to drive a target project's feature_list.json to all-green -- DISCOVER (brain) -> GENERATE (project's skill) -> CRITIQUE (critic) -> advance/retry/stop, with hard stops and a human gate. Deterministic mechanics; the model drives inside.
---

# run-loop -- the orchestrator (LOOPS.md I/V)

You drive the loop. The mechanics here are deterministic; the judgment lives in
the generator and critic you invoke. You hold the state (`feature_list.json`,
`progress.md`) and enforce the stops.

**Announce:** "I'm running the loop on <target>; <N> items, human-gated."

## Setup (from the project profile + the contract)

Read `loops/profiles/<project>.md` (planner, generator, verify command,
`pre_verify` checks, `rubric_context`, `knowledge_context`, reinvention catalog,
brain query, `model` + `test_model`) and the target's `contract.md` +
`feature_list.json`. The two context fields go to different roles:
`knowledge_context` (the brain) is handed to DISCOVER; `rubric_context` (the
project's principles dir) is handed to the critic. Never cross them -- the
brain informs what to build; it cannot fail a diff.

**PROFILE-SANITY (every run start, before anything else).** Profiles rot; the
orchestrator checks, never assumes. Re-validate the profile's facts: the repo
path exists, the verify command runs, the `rubric_context` path exists, and the
contract files are present at their declared home. Any stale fact = STOP and
surface to the human before the item loop.

**CONTRACT-SANITY (one-time, before the item loop).** Scan every assertion in
`contract.md` and inspect its `check:`. Flag any that is NON-falsifiable -- one
with no test, command, or greppable target that could actually FAIL; a check that
just restates the claim instead of pinning it to a fact that could go red. Surface
the flagged assertions to the human at the existing contract boundary (below)
BEFORE building. A contract that can't fail you hasn't constrained anything --
weak goal conditions are one level up from the slop the critic catches in a
transcript, and they let a "pass" mean nothing.

## The loop body -- per item, until all-green or a hard stop

1. **PICK** the next `todo` item in `feature_list.json`; set `doing`.
2. **DISCOVER** (memory, not re-derivation): query the brain for the domain
   knowledge AND the existing primitives tied to this item's files
   (`<brain query> "<area>" --context <knowledge_context>`). Hand this to the
   generator as required reading -- this is PULL, not "please read the docs."
3. **GENERATE**: invoke the project's generator skill on this item, with the
   discovered context, the item's assertions, and the requirement to end with a
   CONFORMANCE MANIFEST. The generator MAY NOT grade itself.
4. **PRE-VERIFY** (the cheap floor -- runs BEFORE the expensive critic): run the
   profile's `pre_verify` checks (deterministic, zero model spend). If ANY fail,
   it is an INSTANT FAIL: append the failing output to `progress.md` and re-invoke
   GENERATE on the SAME item with that failure as the fix list -- WITHOUT invoking
   the critic. Never spend a critic subagent on something `gofmt`/`vet`/`build`
   would catch. Only when pre-verify is fully green do you proceed to CRITIQUE.
   `iters += 1`. (Pre-verify is the cheap floor; the critic is the expensive
   ceiling.)
5. **CRITIQUE**: invoke the `critic` skill (its own context) on the diff.
6. **DECIDE** from the verdict:
   - PASS -> mark the item `done`; append to `progress.md`; WRITE-BACK to the
     brain (note any primitive reused/added so run N+1 knows); then PAUSE for the
     human "go" (default gate) before the next item.
   - FAIL -> append the verdict to `progress.md`; if a hard stop trips, STOP and
     surface; else re-invoke GENERATE on the SAME item with the critic's specific
     failures as the fix list. `iters += 1`.

## Hard stops (from the profile; defaults)

- `max_iters` per item (default 20).
- no-progress: 2 consecutive iterations with no diff or no newly-passing
  assertion -> STOP.
- budget ceiling (token/$ cap) -> STOP.
Any stop: pause, surface `progress.md` + the blocking assertion to the human.

## Human gate (default: human-gated)

Pause after each item passes; resume on "go". Loosen to contract-failure-only
(run straight through, stop only when the CONTRACT itself is wrong) ONLY when
the owner says so.

## Model tiers (the generator/critic Agent calls cost real tokens)

Pass the model to each generator/critic invocation via the Agent tool's `model`
option (or Workflow `opts.model`):

- `test_model` (the profile's cheapest tier -- Haiku) when TESTING the harness
  PLUMBING: does the loop move, does pre-verify fire on a bad diff, does the critic
  gate a fail, does crash-resume work off the three files. Mechanics only.
- `model` (the profile's strong tier) for REAL work.

CRITICAL: Haiku is fine to confirm the MECHANICS move, but the adversarial critic's
real value -- conformance and reinvention judgment on actual code -- needs the
strong tier. `test_model` is for testing mechanics, NOT for trusting the critic's
VERDICTS on real code.

## HEARTBEAT (the coordinator's liveness duty -- owner-mandated 2026-07-05)

Whenever ANY delegated work is in flight (a generator, a critic, a suite gate),
the coordinator checks in on a ~5-minute cadence and gives the human a BRIEF
one-sentence status. **Every heartbeat the human sees starts with a timestamp**
(`HH:MM:SS` local) -- both the raw watchdog line and the coordinator's relayed
sentence -- so the human can see cadence and staleness at a glance without
trusting the narration. Each heartbeat VERIFIES before it reports -- one cheap
process-table + log-mtime check (`ps -C run-tests` / newest log mtime / ok-count),
never a repeat of the delegate's last claim:

- **Verified progressing** -> one sentence: what is running, how far, ETA shape.
- **Stale** (no process, or log mtime older than ~5 min while a run is claimed)
  -> do NOT report "holding": that is the parked-agent trap ("running" !=
  "progressing"). Act immediately: stop the parked delegate, take the step over
  (launch the gate yourself, commit yourself), THEN report what you did.

Mechanism when the coordinator is itself event-driven (an agent that sleeps
between notifications): arm a WATCHDOG background task that exits every ~300s
with the status line -- its completion wakes the coordinator, which verifies,
reports one sentence, and re-arms until the in-flight work lands. A silent gap
longer than ~10 minutes between reports is a coordinator failure, not a delegate
failure.

**SINGLE-CHAIN rule (cadence hygiene).** Exactly ONE pending watchdog exists at
any time. Every beat the coordinator emits -- whether the scheduled watchdog
fired or an unrelated event prompted an ad-hoc status -- RETIRES the pending
watchdog (TaskStop) and arms the next one at +5 min from THAT beat. Never arm a
new watchdog without retiring the old: overlapping chains bunch beats seconds
apart and then leave gaps, which reads as noise and hides real staleness. (The
completion-waiter on the in-flight work is separate and exempt -- it is the end
signal, not a heartbeat.) Note: notification delivery can still bunch behind a
long coordinator turn; the chain keeps the EMISSION cadence honest.

## Trace (LOOPS.md VII)

Pipe each generator/critic invocation's output to a per-run log; `progress.md` is
the human-readable index into it. When a verdict looks wrong, grep the trace --
do not re-run blind.
