# M76 — SBP-421 Yield Report Hyperlane Transport

Date: 2026-04-22

## Decision

SBP-421 was implemented as plumbing first and has now completed live cutover on
the current zero-start generation.

- `VAULT_REPORTER_TRANSPORT=direct` was the original safe mode before
  activation.
- `VAULT_REPORTER_TRANSPORT=both` was the first live soak mode for mirrored
  comparison and remains the rollback / next-generation soak mode.
- `VAULT_REPORTER_TRANSPORT=hyperlane` is now live on DO after the runtime
  allowlist, trusted-sender wire-up, both-mode gates, and short post-cutover
  soak all passed.
- In `hyperlane` mode, the SBP-401 pre-settlement single-policy refresh waits
  for PRMX `policyVaultReporter.policyVaultAssets(policyId).reported_at` to
  become strictly greater than the pending settlement `since_block` before it
  returns success. If delivery does not land before timeout, it fails closed
  and leaves the pending settlement entry on-chain for retry.
- Current Route 2 active-vault freshness is Hyperlane-first. A
  `VAULT_REPORTER_PRE_SETTLEMENT_TRANSPORT=direct` override may still be used
  as the conservative maturing-policy fallback; do not confuse that rollback
  rail with periodic active-vault report delivery.

This keeps SBP-405 yield accrual separate from SBP-421 report transport:

- SBP-405: PRMX ICA command causes Base-side yield/loss movement.
- SBP-421: Base-side vault totals and rebalance acknowledgements are reported back to PRMX through Hyperlane Mailbox delivery instead of the legacy policy-v4 direct vault-assets extrinsic.

## Current Operational State

As of 2026-05-01, the live testnet generation is on runtime
`spec_version=510`. The current manifests pin:

- Base `YieldReporter`: `0xdFDcE3567dE0c74AF4DA283b1f13faA6cD90Ce6E`
- PRMX `YieldReportRecipient`: `0x3d23e0fb68F283fcCcf760D6a164Dabb6e147F3a`
- PRMX yield-report precompile: `0x0000000000000000000000000000000000000802`

The current recipient is the replacement deployed after the first cutover
recipient. The runtime allowlist, smoke manifests, and PRMX-EVM deployment
manifest must all point to the same recipient before `hyperlane` delivery is
considered writable.

Two gas controls are required together:

- `YieldReportRecipient.PRECOMPILE_GAS_STIPEND = 8_000_000`
- PRMX relayer config `chains.prmx.transactionOverrides.gasLimit = 15000000`

The stipend bounds the recipient -> `0x0802` precompile call; the relayer cap
prevents Frontier `Mailbox.process(...)` estimation from chasing the destination
block limit. If delivery regresses, use
`scripts/hyperlane/profile-yield-report-gas.mjs` to separate direct precompile,
recipient handle, and full Mailbox process costs before changing either cap.

Accounting guards after the 2026-05-01 follow-up:

- `record_vault_funded` calibrates `initialVaultAssets` immediately in spec
  510, so the first post-funding report is real yield/loss rather than an
  implicit baseline reset.
- The legacy `prmxPolicyV4.update_vault_assets` direct extrinsic cannot apply
  upward deltas. Upward vault growth must arrive through the trusted
  `pallet-prmx-policy-vault-reporter` path, including Hyperlane `0x0802`.
- Vault-backed settlement fails closed if `latestVaultAssets` is missing, and
  the watcher returns the full latest vault snapshot from Base to Reserve before
  PRMX finalization.

## Route

```text
oracle-service VaultReporter
-> Base YieldReporter.dispatchYieldBatch / dispatchYieldSingle / dispatchRebalanceAck
-> Base Hyperlane Mailbox
-> Hyperlane ISM / relayer / validators
-> PRMX Hyperlane Mailbox
-> PRMX YieldReportRecipient.handle(origin, sender, envelope)
-> PRMX EVM precompile 0x0000000000000000000000000000000000000802
-> pallet-prmx-policy-vault-reporter apply_hyperlane_*
-> pallet-policy-v4 VaultAssetsReportHook
-> pallet-assets(1) canonical bookkeeping update
```

The recipient performs Mailbox, origin-domain, trusted-sender, envelope-version,
kind, and fixed-length checks before calling the precompile. The precompile is
the runtime boundary and accepts only allowlisted EVM callers.

## Contracts and Runtime Surface

Base:

- `evm/src/hyperlane/YieldReporter.sol`
- Dispatches fixed-schema envelopes through `IMailbox.dispatch`.
- Quotes dispatch fee through `quoteYieldBatch`, `quoteYieldSingle`, and `quoteRebalanceAck`.
- Operator-gated; owner can rotate operator and remote recipient.

PRMX EVM:

- `evm/src/hyperlane/YieldReportRecipient.sol`
- Handles Mailbox-delivered messages from the trusted Base `YieldReporter`.
- Calls `IYieldReportPrecompile.handleYieldReport(bytes)` at `0x0802`.
- Exposes `interchainSecurityModule()` for Hyperlane Mailbox ISM lookup.

Runtime:

- `pallets/pallet-evm-precompile-prmx-yield-report`
- Runtime address: `0x0000000000000000000000000000000000000802`
- ABI selector: `handleYieldReport(bytes)` = `0x281684ec`
- Runtime allowlist is live on the current zero-start testnet at
  `spec_version=510`, pinned to the deployed PRMX `YieldReportRecipient`
  `0x3d23e0fb68F283fcCcf760D6a164Dabb6e147F3a`.
- The generic source intentionally leaves `PrmxYieldReportAuthorizedCallers`
  empty; each live deployment must pin it to the fresh
  `YieldReportRecipient` address before `hyperlane` mode can mutate state.

## Envelope Schema

All integers are big-endian and the first byte is `version = 1`.

| Kind | Byte 1 | Length | Body |
| --- | ---: | ---: | --- |
| Batch | `1` | `20 + n * 32` | `epoch u64`, `batchIndex u32`, `batchCount u32`, `entriesLen u16`, then `policyId bytes16 + totalAssets u128` repeated |
| Single | `2` | `42` | `policyId bytes16`, `totalAssets u128`, `reportedAt u64` |
| Rebalance ack | `3` | `82` | `policyId bytes16`, `credit u128`, `debit u128`, `sourceMessageId bytes32` |

Batch replay is idempotent by `(epoch, batchIndex)`. Single replay is
idempotent by `(policyId, reportedAt)`. Rebalance acknowledgements keep the
existing `messageId` idempotency.

## Oracle-Service Transport Flag

Environment:

```text
VAULT_REPORTER_ENABLED=true
VAULT_REPORTER_PERIODIC_ENABLED=true
VAULT_REPORTER_PERIODIC_DISPATCH_ENABLED=true
VAULT_REPORTER_TRANSPORT=direct|both|hyperlane
VAULT_REPORTER_PRE_SETTLEMENT_TRANSPORT=direct|both|hyperlane
VAULT_REPORTER_YIELD_REPORTER_ADDRESS=0x...
VAULT_REPORTER_HYPERLANE_FEE_WEI=0
VAULT_REPORTER_HYPERLANE_DELIVERY_TIMEOUT_MS=900000
VAULT_REPORTER_HYPERLANE_DELIVERY_POLL_MS=5000
VAULT_REPORTER_HYPERLANE_BATCH_SIZE=1
```

When `VAULT_REPORTER_HYPERLANE_FEE_WEI=0`, the reporter reads the appropriate
`YieldReporter.quote*` function and sends the quoted native fee. A non-zero
value overrides the quote for emergency/operator testing.

The delivery timeout/poll settings apply to targeted pre-settlement
`dispatchYieldSingle(...)` refreshes when the effective pre-settlement
transport includes Hyperlane. The 24h active-vault batch path remains scheduler
driven by `VAULT_REPORTER_INTERVAL_MS`; keep `VAULT_REPORTER_HYPERLANE_BATCH_SIZE=1`
on the current PRMX process-gas cap.

## Deployment and Wire-Up

Deployment scripts now emit the new surfaces:

- Base `DeployHyperlane.s.sol` deploys `YieldReporter`.
- PRMX `DeployPrmxEvmHyperlane.s.sol` / `scripts/hyperlane/deploy-prmx-evm.sh`
  deploy `YieldReportRecipient` and record precompile `0x0802`.
- For an already-live generation, use the incremental scripts instead of full
  redeploy:
  - `scripts/hyperlane/deploy-yield-report-base.sh`
  - `scripts/hyperlane/deploy-yield-report-prmx-evm.sh`
- `scripts/hyperlane/execute-zero-start-wireup.sh` optionally wires:
  - `YieldReporter.setRemoteRecipient(bytes32(prmxYieldReportRecipient))`
  - `YieldReportRecipient.setTrustedSender(bytes32(baseYieldReporter))`
- `scripts/hyperlane/sync-zero-start-configs.mjs` exports
  `VAULT_REPORTER_YIELD_REPORTER_ADDRESS` and smoke manifest fields when the
  manifests contain these addresses.
- After `YieldReportRecipient` is known, prepare the allowlist runtime with:

```bash
node scripts/hyperlane/apply-yield-report-runtime-allowlist.mjs \
  --recipient "$(jq -r '.prmx.yieldReportRecipient' evm/deployments/prmx-evm/hyperlane-manifest.json)"
```

Then build and submit the runtime upgrade using the Council runtime-upgrade
runbook before changing `VAULT_REPORTER_TRANSPORT`.

Live activation order for a new generation or replacement recipient:

1. Complete SBP-395 24h soak without changing runtime behavior.
2. Deploy Base `YieldReporter` and PRMX `YieldReportRecipient` incrementally.
3. Run `execute-zero-start-wireup.sh` so both trusted-sender pointers match.
4. Patch/build/upgrade the runtime allowlist for precompile `0x0802`.
5. Sync configs/manifests, set `VAULT_REPORTER_TRANSPORT=both`, restart the
   oracle, and soak until Hyperlane reports match the reporter-pallet direct path. Keep
   `VAULT_REPORTER_PERIODIC_DISPATCH_ENABLED=true` for active-vault freshness
   once the recipient stipend and PRMX relayer process-gas cap are in place.
6. Run the targeted `yield-pre-settlement-refresh` smoke to prove a single
   pre-settlement refresh can wait for Hyperlane delivery and PRMX freshness.
7. Run `settlement-pre-refresh` to prove a real
   `pendingPreSettlementReport` queue can be cleared through Hyperlane refresh
   and settlement can finalize.
8. Only after `both` mode is clean and the historical quarantine / queue
   decision is resolved, return `VAULT_REPORTER_TRANSPORT=hyperlane`.

Live note from 2026-04-23:

- Base `YieldReporter` must use the same active Base MerkleTreeHook as
  `HypERC20Collateral`; otherwise validator checkpoints may not reach quorum
  even though the Mailbox dispatch tx succeeded.
- PRMX `YieldReportRecipient` must call precompile `0x0802` with low-level
  `address(precompile).call{gas: PRECOMPILE_GAS_STIPEND}(...)`; high-level
  Solidity interface calls can revert because Frontier precompile addresses have
  no EVM bytecode, while unbounded low-level calls can make Frontier
  `eth_estimateGas` chase the destination block gas limit.
- PRMX relayer delivery must also cap the inflated `Mailbox.process` gas
  estimate to the configured PRMX process gas limit. The fixed recipient brought
  direct `YieldReportRecipient.handle` estimates back to bounded gas, but
  Frontier can still overestimate full Mailbox/ISM `process(...)` calls; the
  relayer should submit with the tested PRMX cap instead of the block limit.
  The current non-KMS hexKey PRMX relayer template cap is `15000000`; the
  earlier `2000000` cap was too low for the replacement-recipient path.
- In `both` mode, `oracle-service` treats the reporter-pallet direct path as
  the safety gate.
  Failed direct batches are bisected, failing single entries are quarantined for
  one hour, and only direct-accepted entries are mirrored through Hyperlane.
  This keeps old anomalous vault data from creating stuck Hyperlane messages.
- For testnet-only historical contamination, `oracle-service` can pin explicit
  waivers with `VAULT_REPORTER_WAIVED_POLICY_IDS=0x...,0x...`. This is a
  temporary cutover aid, not a production mechanism: live routing stays in
  `both`, the waived IDs stay visible in status/logs, and the long-term action
  is to retire the affected policies rather than hide future anomalies.
- Targeted pre-settlement refresh probe passed without changing live
  `VAULT_REPORTER_TRANSPORT=both`: single Base `YieldReporter` dispatch,
  PRMX Mailbox delivery, and `reported_at > freshAfterBlock`.
- Full settlement pre-refresh gate passed without changing live
  `VAULT_REPORTER_TRANSPORT=both`: real `pendingPreSettlementReport`, Hyperlane
  refresh, settlement progression, Base reserve return, and PRMX finalization.

## Redeploy / Rollback Gates

After any future runtime, `YieldReportRecipient`, `0x0802` allowlist, or relayer
gas-cap redeploy, do not return to `VAULT_REPORTER_TRANSPORT=hyperlane` until
all are true:

- Runtime deployed with `PrmxYieldReportAuthorizedCallers` containing the
  current PRMX `YieldReportRecipient`.
- Base `YieldReporter.remoteRecipient` equals that PRMX recipient.
- PRMX `YieldReportRecipient.trustedSender` equals the Base `YieldReporter`.
- Hyperlane mock-flow test passes for batch, single, and rebalance ack.
- Targeted pre-settlement Hyperlane single-refresh polling is enabled,
  fail-closed, and passes live.
- `both` mode has been soaked long enough to prove the Hyperlane-delivered
  snapshot matches the reporter-pallet direct path.
- A full settlement smoke exercises the actual `pendingPreSettlementReport`
  queue and clears it through the Hyperlane refresh path. This passed on
  2026-04-23.
- Historical quarantined policies are no longer bouncing in and out of
  in-memory quarantine after reporter restarts. For testnet this can be
  satisfied either by explicit waiver plus documented retirement follow-up, or
  by direct retirement/repair before cutover.
- SBP-420 / real-accrual baseline decision is either complete or explicitly
  waived for mock-yield-only testnet coverage.
- The relayer queue audit is `empty` or historical-only and the cleanup planner
  does not identify current-epoch / current-recipient messages requiring
  targeted retry.

## Verification

Local verification on 2026-04-22:

- `FOUNDRY_PROFILE=hyperlane forge test --match-path test/hyperlane/YieldReportFlow.t.sol`: PASS, 6 tests.
- `FOUNDRY_PROFILE=hyperlane forge build`: PASS; existing lint warnings remain.
- `cargo test -p pallet-prmx-policy-vault-reporter --lib`: PASS, 23 tests.
- `cargo check -p pallet-evm-precompile-prmx-yield-report`: PASS.
- `cargo check -p prmx-runtime`: PASS.
- `npm run build` in `oracle-service`: PASS.
- `npx tsx --test src/capital/__tests__/vault-reporter.test.ts`: PASS, 13 tests.

Live verification on 2026-04-23:

- `VAULT_REPORTER_TRANSPORT=both` enabled on DO with reporter-pallet direct retained as the
  safety rail. This was the first cutover recipient; see Current Operational
  State for the replacement recipient now pinned in manifests.
- Runtime allowlist for `0x0802` live at `spec_version=494`.
- Base `YieldReporter`:
  `0xdFDcE3567dE0c74AF4DA283b1f13faA6cD90Ce6E`
- PRMX `YieldReportRecipient`:
  `0x82cAf5a3e7217e46833D45E0fE5186252499D4A2`
- `yield-pre-settlement-refresh` smoke: PASS.
  - policy `0xd8cc726a06b18ae71d71140310c556f1`
  - vault `0x7E6Cd9DA5d375ef47080666aA46c79AA7C60A6fe`
  - Base dispatch tx
    `0x7a0a0a76bdd6aa90bce5ddb86d38e0a66d72ebbaacdd2215650d77a5a45b87d7`
  - Hyperlane message
    `0xc1c83a04cf753b9f84901c65f4fb42684a07ddb44208c9ac0d01bb6963741bcf`
  - PRMX Mailbox `delivered=true`
  - PRMX freshness moved from block `17425` to `reported_at=17428`.
- `settlement-pre-refresh` smoke: PASS.
  - policy `0xc625b73607abb2904245dc334a15a780`
  - vault `0xb337217533fb0ba4a51435aCdEbf17372604a191`
  - pending pre-settlement `sinceBlock=17591`
  - Base dispatch tx
    `0xf40b5ff78e4baa7fd201a23ef10a9cd8ecbab355ae8bbe214a291546609aa968`
  - Hyperlane message
    `0x0230819d9034a8d4f0f87d603ea69cf13d12a48121f88188754d7d1119cc0e51`
  - PRMX freshness moved to `reported_at=17594`
  - Base reserve-return tx
    `0xa953e37871d221775d5a599cfc04f05fafb8a0cd2af8003396eac9a1adcaeb72`
  - collateral delta `100000000`, matching returned amount
  - PRMX finalized the Base tx hash for the policy.
- Historical waiver mode: implemented on 2026-04-23.
  - `VAULT_REPORTER_WAIVED_POLICY_IDS` now accepts explicit lower-cased
    `policyId` values and skips only those entries before quarantine.
  - Current testnet waiver set:
    - `0x9642904e5ca8d43c8373ca91f5769319`
    - `0xc2e0a38cf12e7d15a09ed8f9a0b09b0b`
  - Reporter restarts no longer need to re-discover and re-quarantine these
    same two historical bad policies before healthy batches resume.
  - DO live rollout confirmed the waiver is loaded through
    `/capital/vault/status` and restart-time logs.
  - The same restart surfaced additional direct-path retirement candidates not
    covered by the waiver, including:
    - `0x8b0f10976492514632c1aeb035b41b5a` with `totalAssets=0`
    - `0xc625b73607abb2904245dc334a15a780` with `totalAssets=30`
  - Retirement of the two waived policies remains the follow-up before any
    `hyperlane`-only cutover decision, and the newly surfaced zero/settled
    policies now join that blocker set.

Live follow-up on 2026-04-23:

- All four blocker policies above were classified from on-chain / Base reads:
  - `retire`: `0x9642904e5ca8d43c8373ca91f5769319`
  - `retire`: `0xc2e0a38cf12e7d15a09ed8f9a0b09b0b`
  - `expected settled-zero edge case`: `0x8b0f10976492514632c1aeb035b41b5a`
  - `expected settled-dust edge case`: `0xc625b73607abb2904245dc334a15a780`

Live cutover on 2026-04-24:

- `VAULT_REPORTER_TRANSPORT=hyperlane` is now live on DO.
- First cutover attempt exposed two separate blockers:
  - Base -> PRMX relayer `Retry(ErrorEstimatingGas)` on `Mailbox.process` until
    the PRMX-side relayer config included
    `chains.prmx.transactionOverrides.gasLimit = 2000000`.
  - After that fallback engaged, the deeper blocker was a stale incomplete
    yield epoch on PRMX:
    `epoch=29615738, batchCount=2, receivedCount=1, closed=false`.
    Newer single-batch epochs were being rejected even though the old split
    epoch was never going to finish.
- Runtime fix:
  - `pallet-prmx-policy-vault-reporter` now lets a newer epoch supersede an
    older incomplete epoch instead of wedging forever on
    `YieldEpochIncomplete`.
  - Council runtime upgrade `494 -> 495` code hash:
    `0x2336487d9852663a8fd844ea1a4f11f14fc5d456b18a6b67865ca25f1ca7bed7`
- Post-upgrade live readback:
  - `policyVaultReporter.yieldReportEpochState` moved to
    `epoch=29615988, batchCount=1, receivedCount=1, closed=true`.
  - All 8 active Escrowed policies advanced to `latestVaultReportBlock=18342`.
- Hyperlane-only validation after the fix:
  - `yield-event`: PASS on active policy
    `0xab978a3e88030d1682c4c85eb6f4203f`
    - PRMX ICA tx:
      `0xf5162ebac63fa5cb4249413edacb1c81bba60dc74cb0787760b7cb8edf68e0d1`
    - Base command tx:
      `0x44b191a835efa4f4904a31d14d901d8deb538da8ca7ade878ef8ab0bc0f664b2`
    - PRMX reporter snapshot advanced from `18364` to `18371`
  - `yield-pre-settlement-refresh`: PASS on the same policy
    - Base dispatch tx:
      `0x978a37065db9443e187e1c44f4fde71903420f50ef506dec5934e2917c7e089c`
    - Hyperlane message:
      `0xf11d1d3a5fcf261031e862355bbb2f6fdb525e442df398c5fb8a08cd8230332e`
    - PRMX freshness moved from `freshAfterBlock=18367` to `reportedAt=18371`
  - Fresh `policy-lifecycle + settlement-pre-refresh`: PASS
    - policy:
      `0x12a85a3d5a814fbe388c66466ff7e6cc`
    - vault:
      `0x90ee7089B1eC6688Bb541E60BdB0C37a97C38695`
    - Hyperlane refresh tx:
      `0x64b38ffa4e44f2d64b7e2429b7023f7525ad39ae7471c08481068442b48617a3`
    - Hyperlane message:
      `0xa9bbbe21dd31fad5cc959ce128c03d418f36b7aaf9433429facec9e55f52c9fb`
    - Base reserve-return tx:
      `0x6af0672ce6a15b8fca8cc794ae6461bcc044ed073a5ef9690db0ad69ad835443`
    - PRMX finalized that Base tx hash for the policy.
- Common root cause: all four policies were already `Settled`, but
  `VaultReporter` was still attempting to mirror their post-settlement vault
  balances. That can legitimately be `0` or tiny dust and should not go back
  through the live yield/loss path.
- `oracle-service` now excludes `fundsStatus=Settled` policies from the report
  submission path while still keeping their vault balances inside the cached
  EVM total used by invariants/reconciliation.
- After deploying that fix to DO, `VAULT_REPORTER_WAIVED_POLICY_IDS` was
  removed from `/etc/prmx/secrets.env`; live status now shows
  `waivedPolicyIds=[]`.
- Post-removal live result:
  - startup / both-mode cycle: PASS
  - `reportVaultAssets(n=7) success` on two consecutive cycles
  - no restart-time `YieldLossExceedsThreshold`
  - `/capital/invariants?routeId=2`: `overallStatus=ok`, unsafe gap `0`

Short live soak on 2026-04-24 after the cutover:

- Three consecutive live samples stayed healthy in `transport=hyperlane`:
  - `/capital/vault/status`: `lastReportedCount=8`, `lastError=null`,
    `waivedPolicyIds=[]`
  - `/capital/invariants?routeId=2`: `overallStatus=ok`, unsafe gap `0`
  - `policyVaultReporter.yieldReportEpochState` advanced
    `29616005 -> 29616007 -> 29616008` and stayed
    `{ batchCount=1, receivedCount=1, closed=true }`
- The relayer admin queue still contains historical Base -> PRMX pending
  entries:
  - `/list_operations?destination_domain=1337090`: `360` ops
  - nonce range stayed fixed at `95501 .. 95868`
  - all current entries remain `PendingMessage` with
    `Retry(ErrorEstimatingGas)` and `submitted=false`
- This backlog did not grow during the soak, while relayer metrics showed the
  live pipeline moving ahead of it:
  - `hyperlane_last_known_message_nonce{origin="basesepolia",phase="message_processed",remote="prmx"} = 95882`
  - `hyperlane_last_known_message_nonce{origin="basesepolia",phase="processor_loop",remote="prmx"} = 95892`
- Running relayer config on DO confirms the required PRMX-side fallback is live:
  - `/opt/hyperlane/relayer/config/relayer.json` has
    `chains.prmx.transactionOverrides.gasLimit = 2000000`
  - Historical note: this was enough for the 2026-04-24 first cutover path, but
    the 2026-05-01 replacement-recipient path requires the current PRMX EVM
    relayer value `15000000`.

Interpretation:

- The Hyperlane transport cutover itself is healthy and current batches are not
  re-entering the old `ErrorEstimatingGas` failure loop.
- The remaining relayer admin queue is historical cleanup work, not an active
  blocker for SBP-421 on this generation.
- A dedicated audit helper now exists so this conclusion is repeatable instead
  of anecdotal:
  - `scripts/hyperlane/relayer-queue-audit.mjs`
  - live run on 2026-04-24 against
    `http://159.223.218.107:9190/{list_operations,metrics}` returned
    `classification=historical_backlog`
  - queue summary:
    - `count=360`
    - `nonce range 95501 .. 95868`
    - all `Retry(ErrorEstimatingGas)`, all `submitted=false`
  - relayer frontiers at the same time:
    - `messageProcessed=95930`
    - `processorLoop=95940`
  - interpretation:
    the newest queued nonce is already behind the live processed frontiers, so
    current traffic is flowing and the queue can be handled as cleanup work
- A dedicated cleanup planner now exists so SBP-424 does not accidentally
  retry stale yield batches:
  - `scripts/hyperlane/relayer-queue-cleanup-plan.mjs`
  - live run on 2026-04-24 against the same queue returned:
    - `classification=historical_backlog`
    - `currentYieldEpoch=29616100`
    - `buckets={ stale_batch_epoch_lt_current: 360 }`
    - `recommendedAction=do_not_retry_db_cleanup_only`
  - sampled entry:
    - message id:
      `0x035092a780f6dd7952ec17fa0bc185f650cd675464afa5bfbfae2ed060481ee0`
    - nonce:
      `95868`
    - decoded envelope:
      `{ kind=batch, epoch=29615986, batchCount=1, entries=8 }`
  - interpretation:
    every queued Base -> PRMX retry item is an old yield batch behind the live
    PRMX epoch. Retrying them risks replaying old vault totals because
    `apply_hyperlane_yield_single(...)` / `apply_hyperlane_yield_batch(...)`
    are replay-idempotent, not "stale value safe" by timestamp alone.
  - operational consequence:
    treat SBP-424 as operator-window queue-state cleanup only. Do not use
    `/message_retry` against the full backlog.
  - runbook:
    `docs/hyperlane-migration/m77-relayer-queue-cleanup-runbook.md`
  - local cleanup primitive:
    `scripts/hyperlane/relayer-queue-apply-skip.mjs`
- SBP-424 is now executed on live DO:
  - relayer rollout:
    `scripts/hyperlane/deploy-relayer-message-skip.sh`
    deployed image
    `prmx-message-skip-20260423t18211776968460z:latest`
  - `/message_skip` existence probe returned HTTP `400` with the expected
    `non-empty pattern` validation error
  - dry-run:
    `targetCount=360`, `matched=360`, `removed=0`, `failed=0`
  - apply:
    `targetCount=360`, `matched=360`, `removed=360`, `failed=0`
  - post queue:
    `classification=empty`, `count=0`
  - restart nuance:
    immediately after relayer restart, metrics-based audit may still show
    `active_or_unclassified` because `messageProcessed` resets to `0`. The
    planner is now restart-aware and still returns
    `do_not_retry_db_cleanup_only` when every decoded queued entry is already a
    stale batch behind the current PRMX epoch.
  - post validation:
    `api-lifecycle-hyperlane` PASS in `377629ms`, post relayer audit `empty`,
    post health `/capital/invariants?routeId=2 -> overallStatus=ok`
- A new combined backend gate now exists:
  `run-smoke --only=api-lifecycle-hyperlane`
  runs health -> deposit -> active policy -> rebalance ->
  `settlement-pre-refresh` -> exit -> health, so the Hyperlane report path is
  exercised inside a broader API-first lifecycle instead of only in isolated
  probes.
- As of 2026-04-24, that gate also includes pre/post relayer queue audit.
  Historical backlog is tolerated, but the run fails if the queue becomes
  `active_or_unclassified`, grows in count, or advances to newer stale nonces
  during the lifecycle. This keeps SBP-424 cleanup noise from hiding fresh
  regressions.

Live follow-up on 2026-05-01:

- Runtime is now `spec_version=510`.
- The active PRMX `YieldReportRecipient` is
  `0x3d23e0fb68F283fcCcf760D6a164Dabb6e147F3a`.
- The non-KMS hexKey relayer config template now uses
  `chains.prmx.transactionOverrides.gasLimit = 15000000` for PRMX.
- Active-vault monitor guidance changed with the overfunding fixes:
  `scripts/prmx-monitor/new-policy-monitor.mjs` checks Base
  `bridgedIn - returnedOut` against `maxPayout`, treats missing/stale report
  evidence as a delivery anomaly only when periodic dispatch is enabled, and
  distinguishes active Escrowed policies from settled dust.
