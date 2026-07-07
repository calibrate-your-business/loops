# loops-orchestrator spec

## Intent

Add an orchestrator to loops so ONE parent session drives a multi-wave program
for any cyb project, first target **cyb-www** (the blog build, which today cannot
run loops at all). The parent audits the plan into a manifest, spawns child
`run-loop`s wave by wave, gates each wave, and escalates only owner decisions. It
writes zero code. The FIRST CUT IS SERIAL (children run one at a time -- the
already-proven run-loop flow); parallel fan-out is a later, separately-proven
capability. Driven by a per-project conf + a plan. Done = point loops at cyb-www
with a committed conf + the blog-system plan, and it runs the blog's phases as
owner-gated waves.

## Constraints

- **Serial-first (load-bearing).** The first orchestrator runs children ONE AT A
  TIME on a SINGLE program branch (children commit sequentially; a checkpoint tag
  before each wave gives rollback) -- exactly how run-loop works today. NO
  per-child worktrees, NO naming convention needed in the serial cut. Worktrees
  exist only to isolate CONCURRENT children, which is a PARALLEL problem: the
  critic's verify is the FULL suite, so parallel children sharing one tree/dev-DB
  produce non-attributable verdicts. So parallel fan-out is a LATER phase and is
  the ONLY place that needs per-child worktrees + a branch naming convention
  (`<program>-w<wave>-<item-id>`) + a parent merge-and-re-verify. Single-writer
  manifest holds throughout.
- **Honest scope: the orchestrator is mostly NEW infra.** Reused as-is: the
  contract format, the critic skill, the pre_verify convention, the brain glue,
  and `run-loop` AS THE CHILD PROCEDURE. Built new: the audit, wave state,
  spawn/monitor, the wave gates (safety_gate/UAT), escalation, registration/
  discovery, the conf generator. `run-loop` itself WILL change (its hardcoded
  `loops/profiles/<project>.md` path; its human-pause behaviour under a parent).
- **Real field names.** The conf uses the EXISTING fields `generator`, `verify`,
  `pre_verify`, `rubric_context`, `mode`; the only NEW fields are `safety_gate`
  and `uat`. There is no `implementor`/`tester`/`critic` field.
- **Copy the automations placement technique faithfully.** Conf committed
  in-project; ONE gitignored registration per repo (`cyb.loop`, NAME+BRANCH, no
  paths) + glob discovery; `git add -f` to publish. Definition committed, only the
  registration gitignored.
- **The manifest is AUTHORED by an audit step and OWNER-APPROVED before wave 1.**
  The five safety fields (wave/risk/files/irreversible/unknown) are only as good
  as whoever set them, so a human approves the manifest as the program's first
  gate.
- **Human gates that work TODAY.** Manifest approval, per-wave UAT, and
  inventory-approval for destructive items all work in-terminal now (parent
  pause + an inventory file the owner approves, recorded in the manifest).
  Telegram is a later upgrade, never a dependency.
- **AI infra outside cyb.** Orchestrator + skills + registration in loops; the
  conf + plan + manifest live with the product.
- loops worktree discipline; ASCII-only committed markdown.

## Spec

### Placement (automations technique, faithful)

- **Conf committed:** `<owning-level>/loops/<name>.conf`, parallel to
  `<owning-level>/automations/*.autojob`. (ad-campaigns already uses a project
  `loops/` dir for contract artifacts; the conf sits alongside -- keep names
  distinct, `<name>.conf`.)
- **Registration:** ONE gitignored `loops/registrations/cyb.loop` (`NAME` +
  `BRANCH`, NO paths), exactly like `automations/registrations/*.repo`. Discovery
  = glob `**/loops/*.conf` in the registered repo. `git add -f` publishes
  participation, never the conf contents.
- Framework (orchestrate skill + registration) in loops; conf + manifest with the
  product.

### The conf (extended profile) -- REAL field names

```
project:        <name>
repo:           <owning dir>            # loops reads this; never inspects layout
generator:      <build skill>           # EXISTING
verify:         <FULL-suite command>    # EXISTING (never a subset)
pre_verify:     <deterministic floor>   # EXISTING
rubric_context: <brain context>         # EXISTING (this binds the critic)
mode:           objective | subjective  # EXISTING (hybrid = unproven extension; not first cut)
safety_gate:    <rehearsal/regression cmd (+ backup audit on high-risk)>  # NEW, between waves
uat:            human                   # NEW, per-wave acceptance
model/test_model                         # EXISTING; parent holds a program budget ledger
```

### The manifest -- authored by AUDIT, owner-approved

`feature_list.json` items gain: `wave`, `risk`, `files`, `irreversible`,
`status` (+`unknown`), and a `rulings` note field.

- **AUDIT step (new):** a planner-role pass reads the plan + the ACTUAL repo,
  drafts a verdict per item, assigns wave/risk/`files`, and defaults anything
  uninspectable to `unknown`. Then the **owner approves the manifest** -- the
  program's first human gate, before wave 1.
- **Single-writer:** the PARENT owns `feature_list.json` + `progress.md`; children
  emit per-item report files the parent merges. (No concurrent writers even later.)
- **Location:** in the target project on its branch (loops' existing rule) -- the
  site's `loops/` dir (ad-campaigns precedent), open item below.

### `orchestrate` mode (parent) -- serial-first

Reads conf + plan -> AUDIT -> owner-approves the manifest -> for each wave,
safest to most irreversible:

1. Run each item's child `run-loop` ONE AT A TIME on the single program branch
   (commit per item); per-item gate = `pre_verify` -> `critic`, as today. Under
   the parent the child's interactive human-pause is SUSPENDED
   (contract-failure-only); the human gates move to the wave boundary.
2. After the wave's items: the parent runs the full `verify` + the `safety_gate`
   (rehearsal; on high-risk waves a backup audit + a checkpoint tag first), then
   **UAT** (owner accepts). Only then advance. (No merge step in the serial cut --
   there is one branch. Merge-and-re-verify appears only in the parallel phase.)
3. Escalate: a child whose reality disagrees with the contract writes
   `status:blocked` + a report file; the PARENT re-waves it (records the ruling in
   the manifest). `unknown`/`irreversible` items escalate to the owner; a
   destructive item emits a dry-run INVENTORY and the owner approves the exact
   list.

The parent writes no code. It holds a **wave-state ledger** (wave N: audited /
approved / running / merged / gate-PASS / UAT-granted-at-T) so the program is
crash-resumable. Recovery: item `doing` + no live child + clean tree -> reset
`todo`; dirty tree -> escalate.

### The gate stack (per wave)

`pre_verify` (per item) -> `critic` (per item) -> [wave] parent-merge + full
`verify` + `safety_gate` -> `UAT` (owner) -> advance. Wave gates only on
ALL-GREEN; a partially-failed wave blocks, or is deferred by an explicit owner
ruling recorded in the manifest.

### Rollback / restore

Per-wave **checkpoint** (tag/branch) before the wave runs. A UAT reject, or a
later wave exposing an earlier break, reverts to the checkpoint + re-verifies.
`safety_gate`'s backup audit names the artifact + a restore test on high-risk
waves. (Reversibility is what makes "sequence by risk" pay off.)

### conf generator (onboarding)

Inspect a project (test runner, rehearsal path, `.claude/skills`, CLAUDE.md) ->
draft `<project>/loops/<name>.conf` -> owner confirms -> register. Sequenced AFTER
the first real run (which hand-writes the conf).

### Invocation

From the project base dir: "Read `loops.conf` and initiate the orchestrator for
the plan at `<plan>`." The orchestrate skill is discovered via the project
CLAUDE.md / shared skills.

## Migration phases

**Phase 1 -- Placement + registration + run-loop path fix.** Add
`loops/registrations/` (gitignored, one `cyb.loop`) + glob discovery; define the
committed `<project>/loops/<name>.conf`. Migrate the sw-go + ad-campaigns profiles
into their projects (strip machine-local paths). Update the hardcoded refs:
`run-loop/SKILL.md:16` and `critic/SKILL.md:20` (both point at
`loops/profiles/<project>.md`), plus loops-spec.md + README. RE-SMOKE plain sw-go
run-loop after the move -- Phase 1 is a regression risk to the only proven flow.

**Phase 2 -- Manifest schema.** Extend `feature_list.json`
(wave/risk/files/irreversible/unknown/rulings); the new fields are OPTIONAL for
plain run-loop (no break to existing contracts).

**Phase 3 -- AUDIT step + manifest approval.** The planner-role pass that authors
the manifest from plan + repo, defaults uninspectable items to `unknown`, and the
owner-approval gate.

**Phase 4 -- `orchestrate` mode, SERIAL.** One child at a time on ONE program
branch (no per-child worktrees); per-item gates; wave = full verify + safety_gate
+ UAT (checkpoint tag before each wave for rollback); escalation; wave-state
ledger + crash-resume; program budget ledger; in-terminal UAT + inventory file.

**Phase 5 -- First real program: cyb-www.** Hand-write cyb-www's conf; run the
blog-system plan's phases as SERIAL gated waves. PREREQ (not built here): the
blog's subjective/hybrid critic must first be made to work -- that is
blog-system Phase 2, done with PLAIN run-loop, no orchestrator; the orchestrator
consumes it once green. (So the blog is unblocked independently; this phase then
drives it under the orchestrator.)

**Phase 6 -- Parallel fan-out (LATER, separately proven).** The ONLY phase that
introduces per-child worktrees + the branch naming convention
(`<program>-w<wave>-<item-id>`) + parent merge-and-re-verify + a concurrency cap.
Proven on a wave-shaped program. NOT required for cyb-www.

**Phase 7 -- conf generator.**

Each phase updates the loops docs as it ships.

## Open items

- Child substrate: Agent subagent vs a detached session for the serial child (and
  later the parallel ones).
- cyb-www manifest location: the site's `loops/` dir (ad-campaigns precedent) vs
  `cyb/plans/blog-system/`.
- `cyb.loop` registration format (confirm NAME+BRANCH, glob discovery).
- Model tier per wave (assigned at audit; the parent budget-ledger cap).
- iOS (preschool-app) verify/safety_gate specifics -- later.
