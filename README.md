# DriftPermit

DriftPermit is a reusable GenLayer Intelligent Contract for keeping autonomous systems from silently relying on external APIs, policies, schemas, or service terms after those dependencies materially change.

An operator registers an authenticated baseline document, a live document URL, a bounded set of human-readable invariants, a designated consumer contract, and a short lease duration. GenLayer validators independently fetch the baseline and live documents, compare every declared invariant, and agree on the complete structured result. A one-use permit is issued only when every invariant is preserved with high confidence. Breaking drift, unclear evidence, source failure, or baseline-hash mismatch suspends authorization.

This repository is intentionally contract-focused. It contains the reusable contract, a guarded consumer example, direct behavioral tests, an end-to-end simulator test, deployment guidance, a live-test fixture, and a threat model. It has no frontend.

## The trust problem

Agents often depend on external services whose behavior is governed by natural-language documentation and policies. A URL staying the same does not mean its authentication rules, spending limits, data handling, interfaces, or availability commitments stayed the same. A normal smart contract can compare hashes, but it cannot decide whether a changed document still preserves the assumptions a human approved.

DriftPermit gives downstream contracts a narrow guarantee: the exact authenticated baseline and the currently fetched live document were assessed against every declared invariant, the result was consensus-compatible at high confidence, and the resulting lease has not expired, been consumed, or been invalidated by a configuration revision.

## Why GenLayer is central

The contract fetches both sources inside its non-deterministic flow. The leader classifies every invariant as `preserved`, `violated`, or `unclear`; validators independently refetch and re-evaluate the same sources. Equivalence binds all consequential fields:

- exact baseline and live source snapshots, including status and SHA-256;
- baseline hash authentication and source-health flags;
- `compatible`, `breaking`, or `unclear` decision;
- the complete, disjoint invariant partition;
- normalized change-type set; and
- confidence band.

Free-form rationale is stored for auditability but cannot authorize a permit. Deterministic contract logic requires healthy sources, an authenticated baseline, all invariants preserved, and `high` confidence before issuing a lease.

This follows GenLayer's guidance to use [authoritative web evidence](https://docs.genlayer.com/understand-genlayer-protocol/core-concepts/web-data-access), structured validator output, and a real [consensus-critical state change](https://docs.genlayer.com/developers/intelligent-contracts/when-to-use-genlayer).

## Lifecycle

1. The operator publishes an immutable baseline document and records its exact SHA-256.
2. The operator deploys DriftPermit and a downstream consumer contract.
3. The operator registers the dependency, its live URL, 1–8 explicit invariants, the consumer address, and a 60–86,400 second lease.
4. Anyone may call `assess_dependency`. Permissionless checks let monitoring services refresh or revoke safety without holding owner authority.
5. Validators fetch both sources and classify every invariant.
6. A high-confidence compatible result issues a one-use permit. A still-valid permit is preserved across compatible rechecks, preventing nonce-rotation denial of service.
7. The named consumer queues `consume_permit` as a finalized internal message. It records the protected effect only after consumption is visible.
8. Breaking or unclear drift, source failure, baseline tampering, expiry, consumption, or owner reconfiguration prevents further use.

## Contract surface

### DriftPermit

- `register_dependency(...)` — creates a source-bound dependency policy.
- `assess_dependency(dependency_id)` — permissionlessly fetches and evaluates current evidence through consensus.
- `consume_permit(dependency_id, nonce)` — one-use, executor-only consumption.
- `reconfigure_dependency(...)` — owner-only revision that increments the policy version and invalidates any lease.
- `get_dependency(id)`, `get_permit(id)`, `is_permit_valid(id, nonce)` — public audit and integration views.

### Guarded consumer example

`examples/guarded_consumer.py` demonstrates the full finality boundary. The configured operator starts an action only when the consumer contract is the dependency's named executor. The consumer claims that dependency/nonce pair, queues permit consumption from its own address, and refuses to finalize the action until the permit contract reports the exact nonce consumed.

## Deployment example

Deploy `contracts/drift_permit.py` with no constructor arguments. Then deploy `examples/guarded_consumer.py` with:

- `drift_permit`: the deployed DriftPermit address;
- `operator`: the account allowed to start protected example actions.

Register a dependency with:

```text
dependency_id: payments-api
name: Acme Payments API
description: External payment API used by an autonomous purchasing agent.
baseline_url: immutable HTTPS URL for the reviewed document
baseline_sha256: lowercase SHA-256 of the exact response bytes
live_url: stable HTTPS URL whose current contents govern the dependency
invariants_json: [{"id":"auth","rule":"OAuth remains mandatory for every payment request."}, ...]
executor: deployed GuardedConsumer address
lease_seconds: 3600
```

Use stable, public, non-personalized sources. `examples/dependency_policy.md` is the repository's live-test fixture: a Studio demonstration can authenticate a commit-pinned raw URL as the baseline and use the `main` raw URL as the live source. Updating that file after the compatible path provides a real public drift event while Git history preserves the exact baseline.

## Live Studio verification

The complete Normal (Full Consensus) demonstration is recorded in [`docs/live-studio-verification.md`](docs/live-studio-verification.md). It includes the deployed contract addresses, finalized transaction hashes, authenticated source hashes, compatible permit lifecycle, contract-to-contract consumption, breaking-drift verdict, and the expected on-chain rollback when an old permit is reused.

## Run checks

```powershell
python -m pip install -r requirements.txt
python -m pytest -p no:cacheprovider tests -q
python -m genvm_linter.cli check --json contracts/drift_permit.py
python -m genvm_linter.cli check --json examples/guarded_consumer.py
```

The direct tests cover source authentication, source failure, complete invariant partitioning, contradictory model output, high-confidence gating, validator disagreement, permissionless recheck safety, one-use consumption, owner authority, lease bounds, and duplicate permit claims. The simulator deploys both contracts and executes the protected action through the real finalized internal-message path before proving that breaking drift blocks a second action.

## Limits

- DriftPermit is a compatibility gate, not a security audit, legal opinion, or guarantee that an external service behaves as documented.
- The owner chooses the baseline, live URL, invariants, executor, and lease duration. Integrators must review that configuration.
- A source may change immediately after assessment. Keep leases short enough for the dependency's risk level.
- Stable source delivery matters. Personalized, region-dependent, highly dynamic, or intermittently unavailable pages may fail consensus or suspend permits.
- Permissionless checks deliberately fail closed on unavailable evidence. This protects safety at the cost of availability.
- The guarded consumer demonstrates correct permit consumption; production consumers must enforce their own operator, asset, and action-specific controls.
