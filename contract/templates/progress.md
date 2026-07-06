# progress -- append-only run log

<!-- One block per iteration. Append only; never rewrite. -->

## <YYYY-MM-DDThh:mm> item:<id> iter:<n>
- generator: <one-line summary of the change + the conformance manifest>
- critic: PASS | FAIL
  - correctness: <verify command result>
  - conformance: <principle checks>
  - reinvention: <symbols checked vs catalog/codebase>
  - evidence: <the specific failing assertion + location, on FAIL>
- decision: retry | advance | stop(<reason>)
