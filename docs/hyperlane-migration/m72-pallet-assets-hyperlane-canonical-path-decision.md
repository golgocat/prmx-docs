# M72 — Canonical Hyperlane / `pallet-assets(1)` Path

PRMX's capital architecture is fixed in this direction:

- `pallet-assets(1)` is the **only bookkeeping source of truth** for settlement-asset balances on PRMX.
- **Inbound** deposits traverse Hyperlane end-to-end before mutating PRMX balances.
- **Outbound** exits start from the pallet path and traverse PRMX EVM → Hyperlane → Base EVM before Base collateral is released.

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

`warpAccount.request_exit`
→ escrow / burn against `pallet-assets(1)`
→ `EvmDispatchHook`
→ `PRMX EVM dispatch surface`
→ `Hyperlane Mailbox`
→ `ISM verification`
→ `relayer / validators`
→ `Base HypERC20Collateral release`

The concrete outbound dispatcher subdesign is in [m73 — Exit Dispatcher design](/docs/hyperlane-migration/m73-exit-dispatcher-design).

## What this means concretely

- PRMX does **not** treat the PRMX EVM synthetic balance as an independent ledger.
- PRMX does **not** accept an oracle-attested mint path.
- PRMX does **not** accept direct `HypERC20Synthetic.transferRemote()` exits.

Repository defaults reflect this:

- `oracle-service` only runs in observe mode for inbound (`HYPERLANE_DEPOSIT_ATTEST_MODE=observe`); canonical inbound expects `SettlementBridgeRecipient` to mint before any DB row is promoted.
- `hyperlane-deposit-watcher` requires `HYPERLANE_DEST_SETTLEMENT_BRIDGE_RECIPIENT_ADDRESS` and does not consume PRMX synthetic `ReceivedTransferRemote` logs for live delivery tracking.
- `scripts/hyperlane-smoke/{deposit,exit}.mjs` reject any non-canonical transport override.
- Zero-start config clears `HYPERLANE_DEST_SYNTHETIC_ADDRESS`, so a fresh generation does not preserve that stale env surface.

## Monitoring

The bridge-net counter is `warpAccount.bridgeMintedTotal`. The Tier 2 invariant is `B ≈ EVM_total = collateral + PolicyVaults + settlement-transient` (see [Capital Invariants](/docs/architecture/CAPITAL-INVARIANTS)). Direct PRMX EVM synthetic balances are not part of this audit — they are not a settlement ledger.
