# DriftPermit threat model

## Assets and authority

- The external dependency assumptions approved by the operator.
- The authenticated baseline document and its exact byte hash.
- The live document fetched during each assessment.
- The short-lived, one-use permit consumed by a designated contract.
- The downstream effect that the consumer performs after consumption.

The dependency owner controls configuration. GenLayer consensus controls the semantic compatibility result. Only the configured consumer address can consume a permit. The consumer, not DriftPermit, controls the final protected action.

## Threats and controls

| Threat | Control | Remaining limit |
| --- | --- | --- |
| Owner substitutes a different baseline after registration | The baseline response bytes must match the stored SHA-256 on every assessment | The owner selected the original baseline and invariants |
| Live policy changes without changing its URL | Validators fetch the live URL and compare every invariant against the authenticated baseline | Detection occurs only when an assessment is called |
| Model omits, duplicates, or contradicts an invariant | Exact full ID partition is required; the decision is derived from invariant states and contradictory output reverts | Validators can still agree on an incorrect interpretation |
| Leader and validator observe different content or conclusions | Equivalence binds both source snapshots and every consequential decision field | Highly dynamic sources may fail consensus rather than produce a decision |
| Unavailable, empty, oversized, or tampered source authorizes use | The result is deterministically inconclusive and any active permit is revoked | Fail-closed behavior can reduce availability |
| Permissionless caller repeatedly rechecks compatible evidence to rotate a nonce | A still-valid compatible permit is preserved unchanged | Rechecks still consume network resources |
| Unauthorized caller consumes a permit | `consume_permit` requires the exact configured executor address | A compromised executor can consume its own permit |
| One permit authorizes multiple consumer actions | The example consumer claims each dependency/nonce once and finalizes only after exact consumption | Production consumers must retain an equivalent replay guard |
| Consumer records an action before child-message finality | The example uses a two-stage `awaiting_permit` → `executed` flow | A different consumer can integrate incorrectly |
| Owner silently changes configuration while a permit is active | Reconfiguration increments the dependency version and invalidates the permit | Owner governance itself is out of scope |
| Source changes immediately after assessment | Permits expire after a bounded 60–86,400 seconds and are one-use | No snapshot can guarantee future external behavior |
| Prompt injection inside fetched documents | Source blocks are explicitly marked untrusted and output is tightly structured | Prompt isolation is not a formal sandbox |

## Deliberate exclusions

DriftPermit does not prove that a provider follows its published terms, establish legal enforceability, execute off-chain API calls, inspect private evidence, or authorize the business purpose of a downstream action. It answers one bounded question: does the currently fetched public dependency document still preserve every declared assumption from an authenticated baseline strongly enough to issue a short on-chain compatibility lease?
