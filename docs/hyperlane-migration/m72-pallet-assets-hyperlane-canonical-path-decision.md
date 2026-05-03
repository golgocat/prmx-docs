# M72 — Canonical Hyperlane / `pallet-assets(1)` Path Decision

**Status**: Accepted architecture decision on 2026-04-21.

## Decision

PRMX fixes the capital architecture in the following direction:

- `pallet-assets(1)` is the **only bookkeeping source of truth** for settlement-asset balances on PRMX testnet.
- **Inbound** deposits must traverse Hyperlane end-to-end before mutating PRMX balances.
- **Outbound** exits must start from the pallet path and then traverse PRMX EVM → Hyperlane → Base EVM before Base collateral is released.

This replaces the prior ambiguous hybrid state where some flows used pallet bookkeeping and others used direct PRMX EVM synthetic calls.

## Canonical routes

### Inbound

`Base HypERC20Collateral lock`
→ `Hyperlane Mailbox`
→ `ISM verification`
→ `relayer / validators`
→ `PRMX EVM SettlementBridgeRecipient`
→ `SettlementAssetERC20`
→ `0x0800` pallet-assets precompile
→ `pallet-assets(1)::mint`

### Outbound

`warpAccount.request_exit` (or its direct successor extrinsic)
→ escrow / burn against `pallet-assets(1)`
→ `EvmDispatchHook`
→ `PRMX EVM dispatch surface`
→ `Hyperlane Mailbox`
→ `ISM verification`
→ `relayer / validators`
→ `Base HypERC20Collateral release`

## What this means concretely

- PRMX does **not** treat the PRMX EVM synthetic balance as an independent ledger.
- PRMX does **not** accept an oracle-attested mint path as the final design.
- PRMX does **not** accept direct `HypERC20Synthetic.transferRemote()` exits as the final design.

Those paths are now disabled or observe-only in the repo. They are not
precedent for future work.

## 2026-04-24 addendum

The repository has now been hardened further in the canonical direction:

- `oracle-service` no longer accepts `HYPERLANE_DEPOSIT_ATTEST_MODE=attest`.
  Canonical inbound is observe-only and expects
  `SettlementBridgeRecipient` to mint before the DB row is promoted.
- `hyperlane-deposit-watcher` now requires
  `HYPERLANE_DEST_SETTLEMENT_BRIDGE_RECIPIENT_ADDRESS` and no longer consumes
  PRMX synthetic `ReceivedTransferRemote` logs for live delivery tracking.
- `scripts/hyperlane-smoke/{deposit,exit}.mjs` now reject
  `legacy|synthetic` transport overrides; canonical deposit / pallet-first exit
  are the only accepted smoke paths.
- zero-start config sync / audit now clear
  `HYPERLANE_DEST_SYNTHETIC_ADDRESS` when canonical inbound is requested, so a
  fresh generation does not preserve that stale env surface by default.

Live testnet follow-through on 2026-04-24:

- synced rebuilt `oracle-service/dist/` to DO
- backed up `/etc/prmx/secrets.env` to
  `/etc/prmx/secrets.env.pre-canonical-only-cleanup-20260424T002043Z`
- cleared `HYPERLANE_DEST_SYNTHETIC_ADDRESS=` on DO while preserving
  `HYPERLANE_DEPOSIT_ATTEST_MODE=observe`
- restarted `prmx-oracle`
- re-ran `api-lifecycle-hyperlane` against the DO manifest and got PASS in
  `381050ms`

## Historical gap vs this decision

This was the 2026-04-21 snapshot that motivated the decision:

- `pallet-assets(1)` precompile exists.
- `SettlementAssetERC20` exists.
- `warpAccount.request_exit` already debits `pallet-assets(1)` into escrow.
- But inbound still relied on `hyperlane-deposit-attest-worker` +
  `warpAccount.attest_deposit`.
- And smoke / some operational exits still called PRMX EVM `transferRemote()`
  directly.
- Runtime `EvmDispatchHook` is still wired to `()`, so pallet outbound does not yet dispatch Hyperlane messages by itself.

Current live state on 2026-04-24 is narrower:

- canonical inbound is `SettlementBridgeRecipient` only
- deposit attestation is observe-only DB promotion, not a mint path
- repo smoke / manual scripts reject `legacy|synthetic` deposit and exit modes
- canonical zero-start config clears `HYPERLANE_DEST_SYNTHETIC_ADDRESS`

## Required next implementation steps

1. Complete SBP-403 C3:
   completed live on zero-start r2. `SettlementBridgeRecipient` now receives
   Hyperlane inbound deposits, calls `SettlementAssetERC20`, and mints
   `pallet-assets(1)` through the `0x0800` precompile. Runtime spec `488`
   also updates `bridgeMintedTotal` atomically with those precompile
   mints/burns.
2. Complete SBP-403 C4:
   replace the oracle-service visible dispatcher sidecar with a permanent
   relayer-visible PRMX EVM dispatch surface so pallet outbound exits dispatch
   through Hyperlane without transport substitution.
   Concrete subdesign is fixed in [m73-exit-dispatcher-design.md](/Users/satorubito/Codes/PRMX/docs/hyperlane-migration/m73-exit-dispatcher-design.md).
3. Remove temporary paths:
   keep `hyperlane-deposit-attest-worker` in observe mode only until DB
   lifecycle consumers are migrated, and stop using direct PRMX EVM
   `transferRemote()` as the canonical exit route.
4. Align monitors and smoke harnesses:
   update invariant docs, runtime readers, oracle-service, and smoke tests so they audit the canonical path rather than the transitional hybrid.

## Consequences for SBP-390 / monitoring

The recent `400 USDC` mismatch investigation showed that the old alerts were mixing two ledgers:

- pallet-side `bridgeMintedTotal`
- direct PRMX EVM synthetic exits

Under this decision, that mixed model is explicitly invalid. The fix is to complete the canonical cutover, not to normalize the hybrid as a permanent architecture.
