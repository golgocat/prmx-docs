# M73 — Canonical Outbound Exit Design

PRMX outbound exits use a **dispatcher + recipient + bridge** split that keeps the PRMX EVM synthetic token out of the user-balance ledger and keeps Hyperlane / collateral custody contracts as small as possible.

## Route

```text
warpAccount.request_exit
→ pallet escrow on pallet-assets(1)
→ pallet PendingExits
→ keeper-submitted, relayer-visible PRMX EVM tx
→ PRMX EVM SettlementExitDispatcher
→ Hyperlane Mailbox / ISM / relayer / validators
→ Base EVM SettlementExitRecipient
→ Base EVM SettlementExitBridge
→ PrmxHypERC20Collateral allowance-backed release
→ watcher-driven warpAccount.finalize_exit
```

The runtime `EvmDispatchHook` is intentionally a no-op. Pallet exits do not create Frontier-invisible Hyperlane dispatches; transport authority is the pallet `PendingExits` state, surfaced through the `0x0801` read-only precompile and validated inside the dispatcher.

## Why this shape

- **No dual ledger on PRMX.** User balances stay authoritative only in `pallet-assets(1)`.
- **No hidden runtime-internal mailbox dispatches.** All Hyperlane sends are EVM transactions emitted through Frontier RPC logs, so the relayer indexes them normally.
- **No transport authority on the keeper EOA.** The dispatcher rejects any payload that does not match `PendingExits[exitId]`, so an oracle-service keeper cannot release funds for an exit the pallet did not create.
- **Custody contract stays small.** Outbound release logic lives in a PRMX-owned `SettlementExitBridge`, not inside the Hyperlane-derived collateral subclass.

## Contracts and responsibilities

### 1. PRMX EVM — `SettlementExitDispatcher`

Responsibilities:

- accept dispatch submissions
- validate the supplied request against `PendingExits[exitId]` via the `0x0801` precompile
- format the outbound `ExitMessageV1` payload
- pay Hyperlane dispatch gas from its own prefunded native balance
- call PRMX `Mailbox.dispatch(...)`
- emit `ExitDispatched(exitId, messageId, recipient, amount, routeId)`

Submission policy:

- `authorizedCaller == address(0)`: permissionless / keeper-driven (intended deployment mode)
- `authorizedCaller != address(0)`: emergency throttle that restricts submission to one address

External surface:

```solidity
function dispatchExit(
    uint32 routeId,
    uint64 exitId,
    address recipient,
    uint256 amount
) external returns (bytes32 messageId);

event ExitDispatched(
    uint64 indexed exitId,
    bytes32 indexed messageId,
    address indexed recipient,
    uint256 amount,
    uint32 routeId
);
```

If dispatcher native balance is insufficient, `dispatchExit(...)` reverts. The pallet exit remains pending and retryable.

### 2. Base EVM — `SettlementExitRecipient`

Implements `IMessageRecipient`. Verifies:

- `msg.sender == Base Mailbox`
- `origin == PRMX_DOMAIN`
- `sender == trustedSender` (padded PRMX dispatcher address)
- payload `version`
- payload `routeId == expectedRouteId`

Computes a replay/audit commitment and forwards to `SettlementExitBridge.releaseExit(...)`.

```solidity
function handle(uint32 origin, bytes32 sender, bytes calldata message) external payable;
function setTrustedSender(bytes32 next) external;
function setExitBridge(address next) external;

event ExitMessageHandled(
    uint64 indexed exitId,
    bytes32 indexed commitment,
    address indexed recipient,
    uint256 amount,
    bool released
);
```

The recipient is explicitly ISM-aware so the Base Mailbox does not fall back to the default DomainRouting ISM for PRMX-origin messages.

### 3. Base EVM — `SettlementExitBridge`

PRMX-owned executor + replay guard. Not a reserve holder.

- holds `processedExitCommitments[exitId]`
- pulls USDC from the active collateral via `transferFrom`
- emits the canonical `ExitReleased` event that PRMX watchers finalize against

```solidity
function releaseExit(
    uint64 exitId,
    bytes32 commitment,
    address recipient,
    uint256 amount
) external returns (bool released);

function setRecipientContract(address next) external;

event ExitReleased(
    uint64 indexed exitId,
    address indexed recipient,
    uint256 amount,
    bytes32 commitment
);
```

Release semantics:

- `msg.sender` must equal `recipientContract`
- if `processedExitCommitments[exitId] == 0`: store `commitment`, `transferFrom(collateral, recipient, amount)`, emit `ExitReleased`, return `true`
- if `processedExitCommitments[exitId] == commitment`: idempotent duplicate, no transfer, return `false`
- if `processedExitCommitments[exitId] != commitment`: revert `CommitmentMismatch`

One-time owner wiring (Council motions):

- `PrmxHypERC20Collateral.approveTokenForBridge(USDC, SettlementExitBridge)`
- `SettlementExitRecipient.setExitBridge(SettlementExitBridge)`
- `SettlementExitBridge.setRecipientContract(SettlementExitRecipient)`

### 4. Base EVM — `PrmxHypERC20Collateral`

Custody contract. Holds the active reserve USDC, enforces source-side deposit gating, and grants allowance to `SettlementExitBridge` via the inherited `approveTokenForBridge(...)` surface. Outbound release logic does not live here.

## Payload schema

`ExitMessageV1`:

```solidity
abi.encode(
    uint8(1),          // version
    uint32 routeId,
    uint64 exitId,
    address recipient,
    uint256 amount     // 6-decimal USDC amount
)
```

Commitment:

```solidity
bytes32 commitment = keccak256(
    abi.encode(
        uint8(1),
        origin,
        sender,
        routeId,
        exitId,
        recipient,
        amount
    )
);
```

`IMessageRecipient.handle(...)` does not receive Hyperlane `messageId`, so `origin + sender + decoded payload` is the replay/audit fingerprint.

## Runtime hook

`HyperlaneExitDispatchHook` is the runtime side of `WarpDispatchHook<AccountId, Balance, ExitId>`. Its current implementation returns `Ok(())` and does not invoke `pallet_evm::Runner::call(...)`.

Reason: runtime-internal `Runner::call` emits `pallet_evm::Log` events visible in `system.events` but **not** returned by Frontier `eth_getLogs`. Hyperlane relayers index PRMX as an Ethereum chain via RPC logs, so a hidden dispatch advances mailbox / merkle sequence counters without giving the relayer the logs it needs, which poisons contiguous sequence indexing.

The visible PRMX EVM transaction to `SettlementExitDispatcher` is therefore the canonical transport submission point. Any keeper / user / relayer helper may submit it; pallet state remains the authority.

## Sequence

1. **Request.** User submits `warpAccount.request_exit(routeId, amount, evmRecipient)`. Pallet checks balance, transfers to escrow, inserts `PendingExits[exitId]`, emits `ExitRequested`.
2. **Eligible.** Pallet calls the no-op `EvmDispatchHook`. The exit remains in `PendingExits[exitId]` waiting for a visible PRMX dispatcher tx.
3. **Dispatch.** A keeper submits `SettlementExitDispatcher.dispatchExit(...)`. Dispatcher reads `0x0801.pendingExit(exitId)` and rejects missing pending exits, route/recipient/amount mismatches, and duplicate dispatches. On success it pays Hyperlane gas, calls `Mailbox.dispatch(...)`, and emits `ExitDispatched`.
4. **Deliver.** Validators sign checkpoints, relayer submits Base `Mailbox.process(...)`, `SettlementExitRecipient.handle(...)` runs, computes the commitment, calls `SettlementExitBridge.releaseExit(...)`, which `transferFrom`s USDC out of the active collateral and emits `ExitReleased`.
5. **Finalize.** Watcher observes `ExitReleased`, extracts the Base EVM tx hash, and calls `warpAccount.finalize_exit(exitId, evm_tx_hash)`. Pallet removes `PendingExits[exitId]`, burns the escrowed `pallet-assets(1)` balance, decrements `BridgeMintedTotal`, inserts `FinalizedExitTxHashes[evm_tx_hash]` and `ProcessedExits[exitId]`.

If the active reserve is short, the `transferFrom` reverts, `handle(...)` reverts, and the message remains undelivered until reserve is refilled and the relayer retries.

## Oracle-service / watcher

- `getExitReleaseState()` reads `SettlementExitBridge.processedExitCommitments(exitId)` and scans `SettlementExitBridge.ExitReleased`.
- An oracle-service submitter may act as a keeper for `dispatchExit(...)` but treats `authorizedCaller() == address(0)` as the expected mode and skips the operator-equality check.
- Event/log scanning is idempotent; the keeper does not spam already-dispatched exits.

## Security properties

- **No dual ledger on PRMX.** `pallet-assets(1)` is authoritative.
- **No EOA transport authority.** The dispatcher cannot emit a Hyperlane message unless pallet state matches.
- **Atomic failure on underfunded dispatch.** The dispatcher reverts; the pallet exit stays retryable.
- **Atomic failure on insufficient Base reserve.** Delivery stays pending until reserve is refilled.
- **Idempotent Base release.** Duplicate same-commitment deliveries do not double-release.
- **Replay guard on PRMX finalize.** `evm_tx_hash` cannot be reused across `finalize_exit` calls.
- **Custody contract stays small.** Release logic lives in `SettlementExitBridge`, not in the Hyperlane-derived collateral subclass.

## Intentional non-goals

- No attempt to make Base release succeed when the reserve is short.
- No attempt to burn PRMX supply before Base release is confirmed.
- No support for direct PRMX EVM `HypERC20Synthetic.transferRemote()` as a permanent exit path.
