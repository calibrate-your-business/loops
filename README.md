# loops -- the dev-loop engine

A reusable harness that runs Karpathy's **LOOPS.md** pattern: a PLANNER, a
GENERATOR, and an adversarial CRITIC arguing over an on-disk CONTRACT, with the
**brain** as shared memory -- so multi-hour build work runs with the model
driving and a human only at the contract boundary.

`loops` is a SYSTEM (peer to `brain`). It **uses** the brain (memory + rulebook)
and **operates on** a target project; it provides the missing role (the critic)
plus the orchestration. It does NOT replace the target project's planner/
generator skills, hold data (`brain-db` does), or hold business code (`cyb` does).

See `loops-spec.md` for the design. First target: sw-go's next MCP features (the
platform MCP for app-builders + the domain MCP for the Ctrl-K menu) on a
`platform-8.5` branch -- mechanical assembly work that is the ideal proving
ground for the conformance critic.
