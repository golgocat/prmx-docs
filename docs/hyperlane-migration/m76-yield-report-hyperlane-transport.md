# M76 — Yield Report Hyperlane Transport

Vault yield / loss reports from Base PolicyVaults flow back to PRMX through Hyperlane: Base `YieldReporter` → Hyperlane Mailbox → PRMX `YieldReportRecipient` → PRMX EVM precompile `0x0802` → `pallet-prmx-policy-vault-reporter` → `pallet-policy-v4` → canonical `pallet-assets(1)` bookkeeping.

This is independent from the ICA command bus described in [m75](/docs/hyperlane-migration/m75-ica-yield-command-bus). ICA causes Base-side yield/loss movement; this transport reports the result back.

## Route

```text
oracle-service VaultReporter
→ Base YieldReporter.dispatchYieldBatch / dispatchYieldSingle / dispatchRebalanceAck
→ Base Hyperlane Mailbox
→ Hyperlane ISM / relayer / validators
→ PRMX Hyperlane Mailbox
→ PRMX YieldReportRecipient.handle(origin, sender, envelope)
→ PRMX EVM precompile 0x0000000000000000000000000000000000000802
→ pallet-prmx-policy-vault-reporter apply_hyperlane_*
→ pallet-policy-v4 VaultAssetsReportHook
→ pallet-assets(1) canonical bookkeeping update
```

`YieldReportRecipient` performs Mailbox, origin-domain, trusted-sender, envelope-version, kind, and fixed-length checks before calling the precompile. The precompile is the runtime boundary and accepts only allowlisted EVM callers (the `YieldReportRecipient` address).

In `hyperlane` transport mode, the pre-settlement single-policy refresh waits for PRMX `policyVaultReporter.policyVaultAssets(policyId).reported_at` to become strictly greater than the pending settlement `since_block` before it returns success. If delivery does not land before timeout, it fails closed and leaves the pending settlement entry on-chain for retry.

## Contracts and Runtime Surface

Base — `evm/src/hyperlane/YieldReporter.sol`:

- dispatches fixed-schema envelopes through `IMailbox.dispatch`
- quotes dispatch fee through `quoteYieldBatch`, `quoteYieldSingle`, `quoteRebalanceAck`
- operator-gated; owner can rotate operator and remote recipient

PRMX EVM — `evm/src/hyperlane/YieldReportRecipient.sol`:

- handles Mailbox-delivered messages from the trusted Base `YieldReporter`
- calls `IYieldReportPrecompile.handleYieldReport(bytes)` at `0x0802` with a bounded gas stipend
- exposes `interchainSecurityModule()` for Hyperlane Mailbox ISM lookup

Runtime — `pallets/pallet-evm-precompile-prmx-yield-report`:

- runtime address: `0x0000000000000000000000000000000000000802`
- ABI selector: `handleYieldReport(bytes)` = `0x281684ec`
- `PrmxYieldReportAuthorizedCallers` is intentionally empty in source; each live deployment must pin it to the deployed `YieldReportRecipient` address before `hyperlane` mode can mutate state

## Envelope Schema

All integers are big-endian and the first byte is `version = 1`.

| Kind | Byte 1 | Length | Body |
| --- | ---: | ---: | --- |
| Batch | `1` | `20 + n * 32` | `epoch u64`, `batchIndex u32`, `batchCount u32`, `entriesLen u16`, then `policyId bytes16 + totalAssets u128` repeated |
| Single | `2` | `42` | `policyId bytes16`, `totalAssets u128`, `reportedAt u64` |
| Rebalance ack | `3` | `82` | `policyId bytes16`, `credit u128`, `debit u128`, `sourceMessageId bytes32` |

- Batch replay is idempotent by `(epoch, batchIndex)`.
- Single replay is idempotent by `(policyId, reportedAt)`.
- Rebalance acknowledgements use `messageId` idempotency.

## Gas Controls

Two gas controls are required together:

- `YieldReportRecipient.PRECOMPILE_GAS_STIPEND = 8_000_000` bounds the recipient → `0x0802` precompile call.
- PRMX relayer config `chains.prmx.transactionOverrides.gasLimit = 15000000` prevents Frontier `Mailbox.process(...)` estimation from chasing the destination block limit.

If delivery regresses, use `scripts/hyperlane/profile-yield-report-gas.mjs` to separate direct precompile, recipient handle, and full Mailbox process costs before changing either cap.

The recipient must call the precompile with low-level `address(precompile).call{gas: PRECOMPILE_GAS_STIPEND}(...)`. High-level Solidity interface calls revert because Frontier precompile addresses have no EVM bytecode, and unbounded low-level calls can make `eth_estimateGas` chase the destination block gas limit.

## Accounting Guards

- `record_vault_funded` calibrates `initialVaultAssets` immediately, so the first post-funding report is real yield / loss rather than an implicit baseline reset.
- The `prmxPolicyV4.update_vault_assets` direct extrinsic is restricted to no-op / downward reports. Upward vault growth must arrive through the trusted `pallet-prmx-policy-vault-reporter` path (Hyperlane `0x0802`).
- Vault-backed settlement fails closed if `latestVaultAssets` is missing. The watcher returns the full latest vault snapshot from Base to Reserve before PRMX finalization.
- `oracle-service` excludes `fundsStatus=Settled` policies from the report submission path while still keeping their vault balances inside the cached EVM total used by invariants / reconciliation. Settled vault balances can legitimately be `0` or tiny dust and must not go through the yield / loss path.

## Oracle-Service Transport Flag

```text
VAULT_REPORTER_ENABLED=true
VAULT_REPORTER_PERIODIC_ENABLED=true
VAULT_REPORTER_PERIODIC_DISPATCH_ENABLED=true
VAULT_REPORTER_TRANSPORT=hyperlane
VAULT_REPORTER_PRE_SETTLEMENT_TRANSPORT=hyperlane
VAULT_REPORTER_YIELD_REPORTER_ADDRESS=0x...
VAULT_REPORTER_HYPERLANE_FEE_WEI=0
VAULT_REPORTER_HYPERLANE_DELIVERY_TIMEOUT_MS=900000
VAULT_REPORTER_HYPERLANE_DELIVERY_POLL_MS=5000
VAULT_REPORTER_HYPERLANE_BATCH_SIZE=1
```

`hyperlane` is the live transport. `both` is the rollback / next-generation soak mode (mirrors deliveries through both the Hyperlane and reporter-pallet direct paths). `direct` is the rollback safe mode used only when the Hyperlane path is being redeployed.

When `VAULT_REPORTER_HYPERLANE_FEE_WEI=0`, the reporter reads the appropriate `YieldReporter.quote*` function and sends the quoted native fee. A non-zero value overrides the quote for emergency / operator testing.

The delivery timeout / poll settings apply to targeted pre-settlement `dispatchYieldSingle(...)` refreshes when the effective pre-settlement transport includes Hyperlane. The 24h active-vault batch path is scheduler-driven by `VAULT_REPORTER_INTERVAL_MS`. Keep `VAULT_REPORTER_HYPERLANE_BATCH_SIZE=1` on the current PRMX process-gas cap.

## Deployment and Wire-Up

Deployment scripts emit the relevant surfaces:

- Base `DeployHyperlane.s.sol` deploys `YieldReporter`.
- PRMX `DeployPrmxEvmHyperlane.s.sol` / `scripts/hyperlane/deploy-prmx-evm.sh` deploy `YieldReportRecipient` and record precompile `0x0802`.
- For an already-live generation, use the incremental scripts:
  - `scripts/hyperlane/deploy-yield-report-base.sh`
  - `scripts/hyperlane/deploy-yield-report-prmx-evm.sh`
- `scripts/hyperlane/execute-zero-start-wireup.sh` wires:
  - `YieldReporter.setRemoteRecipient(bytes32(prmxYieldReportRecipient))`
  - `YieldReportRecipient.setTrustedSender(bytes32(baseYieldReporter))`
- `scripts/hyperlane/sync-zero-start-configs.mjs` exports `VAULT_REPORTER_YIELD_REPORTER_ADDRESS` and smoke manifest fields when the manifests contain these addresses.
- After `YieldReportRecipient` is known, prepare the allowlist runtime:

```bash
node scripts/hyperlane/apply-yield-report-runtime-allowlist.mjs \
  --recipient "$(jq -r '.prmx.yieldReportRecipient' evm/deployments/prmx-evm/hyperlane-manifest.json)"
```

Then build and submit the runtime upgrade using the Council runtime-upgrade runbook before changing `VAULT_REPORTER_TRANSPORT`.

Activation order for a new generation or replacement recipient:

1. Deploy Base `YieldReporter` and PRMX `YieldReportRecipient` incrementally.
2. Run `execute-zero-start-wireup.sh` so both trusted-sender pointers match.
3. Patch / build / upgrade the runtime allowlist for precompile `0x0802`.
4. Sync configs / manifests, set `VAULT_REPORTER_TRANSPORT=both`, restart the oracle, and soak until Hyperlane reports match the reporter-pallet direct path. Keep `VAULT_REPORTER_PERIODIC_DISPATCH_ENABLED=true` for active-vault freshness once the recipient stipend and PRMX relayer process-gas cap are in place.
5. Run the targeted `yield-pre-settlement-refresh` smoke to prove a single pre-settlement refresh can wait for Hyperlane delivery and PRMX freshness.
6. Run `settlement-pre-refresh` to prove a real `pendingPreSettlementReport` queue can be cleared through Hyperlane refresh and settlement can finalize.
7. Only after `both` mode is clean, return `VAULT_REPORTER_TRANSPORT=hyperlane`.

Configuration constraints:

- Base `YieldReporter` must use the same active Base MerkleTreeHook as `HypERC20Collateral`, otherwise validator checkpoints may not reach quorum even though the Mailbox dispatch tx succeeded.
- In `both` mode, `oracle-service` treats the reporter-pallet direct path as the safety gate. Failed direct batches are bisected, failing single entries are quarantined for one hour, and only direct-accepted entries are mirrored through Hyperlane.

## Redeploy / Rollback Gates

After any future runtime, `YieldReportRecipient`, `0x0802` allowlist, or relayer gas-cap redeploy, do not return to `VAULT_REPORTER_TRANSPORT=hyperlane` until all are true:

- Runtime deployed with `PrmxYieldReportAuthorizedCallers` containing the current PRMX `YieldReportRecipient`.
- Base `YieldReporter.remoteRecipient` equals that PRMX recipient.
- PRMX `YieldReportRecipient.trustedSender` equals the Base `YieldReporter`.
- Hyperlane mock-flow test passes for batch, single, and rebalance ack.
- Targeted pre-settlement Hyperlane single-refresh polling is enabled, fail-closed, and passes live.
- `both` mode has been soaked long enough to prove Hyperlane-delivered snapshots match the reporter-pallet direct path.
- A full settlement smoke exercises the actual `pendingPreSettlementReport` queue and clears it through Hyperlane refresh.
- The relayer queue audit (`scripts/hyperlane/relayer-queue-audit.mjs`) classifies the queue as `empty` or `historical_backlog`, not `active_or_unclassified`, and the cleanup planner (`scripts/hyperlane/relayer-queue-cleanup-plan.mjs`) does not identify current-epoch / current-recipient messages requiring targeted retry.

`pallet-prmx-policy-vault-reporter` lets a newer epoch supersede an older incomplete epoch instead of wedging on `YieldEpochIncomplete`. This avoids permanent stalls when a split epoch never finishes and newer single-batch epochs would otherwise be rejected.

## Combined Backend Gate

`run-smoke --only=api-lifecycle-hyperlane` runs health → deposit → active policy → rebalance → `settlement-pre-refresh` → exit → health, exercising the Hyperlane report path inside a broader API-first lifecycle. The gate also includes pre/post relayer queue audits: historical backlog is tolerated, but the run fails if the queue becomes `active_or_unclassified`, grows in count, or advances to newer stale nonces during the lifecycle.

## Active-Vault Monitor

`scripts/prmx-monitor/new-policy-monitor.mjs` checks Base `bridgedIn - returnedOut` against `maxPayout`, treats missing / stale report evidence as a delivery anomaly only when periodic dispatch is enabled, and distinguishes active Escrowed policies from settled dust.
