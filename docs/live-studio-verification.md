# Live GenLayer Studio verification

DriftPermit was exercised end to end in GenLayer Studio using Normal (Full Consensus) mode on October 3, 2026. The test used a public, commit-pinned baseline and a mutable `main` fixture so validators evaluated a real external change rather than injected test strings.

## Deployments

- DriftPermit: [`0x4Bdb443424bEe8dd22809dd0B76755109aC89615`](https://explorer-studio.genlayer.com/address/0x4Bdb443424bEe8dd22809dd0B76755109aC89615)
- GuardedConsumer: [`0x4cEdAc7a470d81EFC151564d5887Ba616bF56C31`](https://explorer-studio.genlayer.com/address/0x4cEdAc7a470d81EFC151564d5887Ba616bF56C31)
- Operator: `0x1DcB045123730e606A88380BCe534332F50332d2`

Deployment transactions:

- DriftPermit: `0x05626f1132f5b9827232dd95bacbca07c07f771a369da7ce8f5e51ddc45de047`
- GuardedConsumer: `0x28cd7a0fd63c016844e0d4ab316a3e4ce288f8869f42700e326c435726a65fb4`

Both deployments reached `FINALIZED` and emitted `Contract deployed` without a GenVM error.

## Authenticated fixture

- Baseline commit: [`61239b2b003c5e2f389e6c09c385e8801cdd8d06`](https://github.com/ShalyX/drift-permit-genlayer/blob/61239b2b003c5e2f389e6c09c385e8801cdd8d06/examples/dependency_policy.md)
- Baseline SHA-256: `0840ddf0270cbf175d19b97d7c3ff05876bb1c9bde83b0b7190fbf7f9e1159c7`
- Live URL: [`main/examples/dependency_policy.md`](https://github.com/ShalyX/drift-permit-genlayer/blob/main/examples/dependency_policy.md)
- Breaking revision commit: [`06f10be55226da770f191346268b69c134bda554`](https://github.com/ShalyX/drift-permit-genlayer/commit/06f10be55226da770f191346268b69c134bda554)
- Breaking live SHA-256: `68254a2651025c9fec32a800f779594a66ef564f985a2c49b214a47fa85e5edf`

The declared invariants required OAuth for every request, capped a single charge at 100 USD, and prohibited provider retention of customer data after settlement.

## Compatible path

1. `register_dependency` finalized in transaction `0xf34909e6b58248531ff06f68502cd1cf4d782dcffa0fbb217637c6d914b24d85`.
2. `assess_dependency` finalized in transaction `0xc413a9eff131e9273112136a2a163b1d31f92a840ac34145d3b6d6934fdbcbab`.
3. The finalized dependency state reported:
   - `decision: compatible`
   - `status: compatible`
   - `confidence_band: high`
   - `sources_healthy: true`
   - `baseline_hash_matches: true`
   - `auth`, `charge_limit`, and `data_retention`: `preserved`
4. Permit nonce `1` was active, unconsumed, and bound to dependency version `1` plus the exact two-source snapshot.
5. The operator started `charge-paper-20261003`, payload `Charge 25 USD for printer paper.`, through GuardedConsumer. Transaction `0xe28aa1d0eb2333abbfb48a8bae8deaee3f902868cf04c8aa7cc33dd686b59e5a` finalized.
6. The finalized consumer action entered `awaiting_permit`; DriftPermit simultaneously reported nonce `1` as `consumed: true` and `active: false`.
7. `finalize_action` transaction `0x093fdf7875557769c88d4d096671b1dd9a7c767982398a36a0ea4b2fb8c2fa60` finalized. The action then reported `status: executed` with nonzero start and finalization timestamps.

This proves the ordinary user/agent path crossed the real contract-to-contract finality boundary and consumed the authorization exactly once.

## Breaking-drift path

The live fixture was then changed so OAuth became optional, the charge ceiling rose to 500 USD, and customer data could be retained for training. The commit-pinned baseline remained unchanged.

1. A second `assess_dependency` finalized in transaction `0xfcdd4b43adc0bee8017942aa6036f1cb1a9d783bc5094830e72d0f4c517ccf2b`.
2. Validators fetched both sources with HTTP 200 and recorded the distinct baseline and live hashes above.
3. The finalized dependency state reported:
   - `decision: breaking`
   - `status: suspended`
   - `confidence_band: high`
   - all three invariant IDs: `violated`
   - change types: `AUTHORIZATION`, `DATA_HANDLING`, `FINANCIAL_LIMIT`, and `POLICY`
4. No replacement permit was issued; nonce `1` remained consumed and inactive.
5. A second consumer action, `charge-retry-20261003`, attempted to reuse nonce `1`. Transaction `0x717ac23be48e41aa3cff8681be92ce5ea4fb27485389a514ae9cbeb7312b9947` finalized with a GenVM rollback:

   ```text
   [EXPECTED] No valid dependency permit
   ```

6. A finalized `get_action("charge-retry-20261003")` read failed, confirming rollback left no action record.

## Local verification

The same revision passes all 16 direct and simulator tests. Both contract files also pass the GenVM linter; the only linter notice is the tool's informational newer-runner warning.
