# contract: <feature> -- <branch>

<!-- The gradable target. The spec is the boundary; THIS is what the critic
grades. Conformance assertions are first-class. -->

## item: <id> -- <title>

### correctness
- [C1] <claim> -- check: <test name / command / grep>
- [C2] <claim> -- check: <...>

### conformance (graded against the brain, not tests)
- [K1] conforms to [[<principle>]] -- check: <prohibition it must not violate>
- [K2] reuses <existing primitive@loc> instead of reinventing -- check: <symbol>

### provenance / auth
- [P1] <provenance / RLS / source claim> -- check: <test / grep>
