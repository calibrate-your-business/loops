# loops-orchestrator: native-primitives reconciliation (2026-07-07)

Verdict from the adversarial review: the "shrink the orchestrator onto native
Claude Code primitives" idea was mostly WRONG. Applying the owner's test -- adopt
a vendor primitive only where it delivers low-level value you genuinely cannot
build as well yourself (the orca -> native-subagents precedent) -- the plan is
essentially UNCHANGED, with two small deltas.

## Per-primitive verdict

- **`/goal` -- REJECT.** Its evaluator is Haiku, cannot call tools, judges only
  the transcript. `run-loop` already iterates to contract-green AND grades with
  the adversarial full-suite critic, and already turn-bounds (max 20/item).
  `/goal` is a strictly weaker judge; zero value to us.
- **`/schedule` -- REJECT.** Research preview, 1-hour minimum, flaky; its only
  edge (survives machine-off) is moot on a 24/7 Mac. launchd is better and owned.
- **auto mode -- REJECT.** Preview; headless `claude -p` + a tool allowlist
  already runs unattended, under our control.
- **dynamic workflows -- OPPORTUNISTIC, not a dependency.** It is a convenience
  over subagents (already owned); the `orchestrate` skill spawns raw subagents
  itself. Its one real value: keeping intermediate results in script variables
  (not the parent's context) fights context-collapse. Use it where that earns its
  keep; drop to raw `agent()` with no rework otherwise.
- **`/usage` -- ADOPT.** Vendor-owned token/billing accounting we cannot compute
  ourselves. Closes the "program budget ledger" open item outright.

## Deltas to the plan (everything else stands as hardened)

1. Open item "program budget ledger" -> **RESOLVED: use `/usage`.**
2. Phase 6 (parallel fan-out) MAY use dynamic-workflows as the fan-out harness
   for its context-isolation, but only as a **droppable convenience**; the
   default remains raw subagents in worktrees.

Serial-first, the contract, the adversarial critic, brain grounding, the manifest
+ audit + owner-approval, risk-ordered waves, safety_gate, UAT, and the
committed-conf/gitignored-registration placement all stand unchanged.

## Why "we already built it" is the point

We already own the capability (subagents, adopted the orca way), the judgment
(contract + critic + brain, on Karpathy-grounded research), and vendor-
agnosticism. Higher-level built-ins would trade control + flexibility for
convenience and re-expose the "hit a limit, get screwed" lock-in. The win of
owning the judgment layer IS the flexibility; adopting-for-convenience forfeits it.

## Status

UNAPPROVED for implementation. Parked here on the loops `scratch` branch pending
the owner's go. The general lesson (when to adopt a vendor primitive vs. build/
keep your own) is staged separately as a brain-db candidate.
