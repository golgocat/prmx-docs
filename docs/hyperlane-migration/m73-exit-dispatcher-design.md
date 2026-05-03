# M73 — Canonical Outbound Exit Design (`EvmDispatchHook` → PRMX dispatcher → Base recipient/bridge)

**Status**: Accepted on 2026-04-21. Updated on 2026-04-22 after SBP-403 C4 live cutover. The first live generation exposed that runtime-internal `Runner::call(...)` mailbox dispatches are not surfaced through Frontier `eth_getLogs`, so the poisoned generation was wiped and replaced by zero-start r2. The runtime `EvmDispatchHook` remains a no-op; C4 replaces the trusted oracle-service dispatcher sidecar with a relayer-visible PRMX EVM dispatcher transaction whose payload is verified against pallet `PendingExits` through the read-only `0x0801` precompile. Any account may submit the EVM transaction when `authorizedCaller == address(0)`, but the dispatcher cannot emit a Hyperlane message unless the pallet state matches the supplied exit id, route id, recipient, and amount. This path is now deployed on PRMX runtime spec `489`.

## Goal

Implement the outbound canonical route without:

- re-introducing the PRMX EVM synthetic token as an independent user ledger
- extending Hyperlane / collateral custody code more than necessary
- coupling bookkeeping and transport into the same contract

Target route:

`warpAccount.request_exit`
→ pallet escrow on `pallet-assets(1)`
→ pallet `PendingExits`
→ keeper-submitted, relayer-visible PRMX EVM tx
→ PRMX EVM `SettlementExitDispatcher`
→ Hyperlane Mailbox / ISM / relayer / validators
→ Base EVM `SettlementExitRecipient`
→ Base EVM `SettlementExitBridge`
→ active `PrmxHypERC20Collateral` allowance-backed release
→ watcher-driven `warpAccount.finalize_exit`

## Implementation note

The current implementation fixes these concrete choices:

- runtime `EvmDispatchHook` is intentionally still a no-op in live spec `489`
  so pallet exits do not create Frontier-invisible Hyperlane dispatches
- the PRMX EVM deploy script deploys the dispatcher with `remoteRecipient = 0`
  so Council can wire the Base recipient after both chains are deployed
- the Base deploy script deploys `SettlementExitBridge` and `SettlementExitRecipient`
  with the recipient-side bridge reference populated, while `trustedSender`
  and `recipientContract` are finalized later by Council motion
- `scripts/hyperlane/generate-exit-cutover-motions.mjs` now emits the Base Safe
  batch for those final motions and prints the matching PRMX owner
  `setRemoteRecipient(...)` command
- oracle-service no longer treats `SettlementVault.processedExitIds` as the
  canonical release probe; it now watches `SettlementExitBridge.ExitReleased`
  and `processedExitCommitments(exitId)`
- C4 introduces `PrmxWarpAccountPrecompile` at
  `0x0000000000000000000000000000000000000801`.
  It exposes `pendingExit(uint64)` as a read-only ABI surface over
  `warpAccount.pendingExits`.
- `SettlementExitDispatcher` now validates each outbound payload against that
  precompile before calling the PRMX Mailbox.
- `authorizedCaller == address(0)` is the intended C4 deployment mode: dispatch
  submission is permissionless/keeper-driven, while pallet state remains the
  authority. A non-zero `authorizedCaller` remains available as an emergency
  throttle.
- oracle-service may still submit the visible PRMX-side dispatcher transaction,
  but it is now only a keeper. It no longer has transport authority because an
  incorrect amount/recipient/route cannot pass dispatcher validation.
- smoke/API exit helpers now prefer the pallet-first route in repo:
  `exit.mjs` builds `warpAccount.request_exit` through the Agent API and polls
  `/capital/exits/:exitId` when the manifest/environment indicates the
  dispatcher + bridge are deployed
- live zero-start r2 deployments/wire-up are now in place for the canonical
  exit contracts:
  - Base `SettlementExitBridge = 0xdCC3B33Abe4A7236f5b399e98632f7c1D8eA1813`
  - Base `SettlementExitRecipient = 0xD76F9dB783690fcD954c0A0d3e03aE2a981F3811`
  - PRMX `SettlementExitDispatcher = 0x2cfC9e176DA56f352f7Da636ec6529E826D3E701`
  and the required trusted-sender / allowance / remote-recipient motions have
  been executed on the live testnet
- latest full smoke after C4 cutover confirmed the canonical path:
  - `exitId = 6`
  - PRMX request tx `0xed3f39a9fe6af2d705967d9994ec7f8be86771741b24695ce8e7d66de122a815`
  - Base release tx `0xd0efdd75822e99b510c1e7a5c51103b84e0a9c650a9eeca67c904b057563a03a`
  - `/capital/exits/6?routeId=2` reached `finalized_on_prmx`

## Design choice

Use a **dispatcher + recipient + bridge** split.

- Do **not** route outbound through `SettlementAssetERC20`.
  It is the PRMX-side bookkeeping facade, not a transport contract.
- Do **not** normalize direct PRMX EVM `HypERC20Synthetic.transferRemote()` as the permanent exit path.
- Do **not** extend `PrmxHypERC20Collateral` with custom exit release logic if a smaller integration surface is available.

This keeps:

- PRMX pallet = bookkeeping / lifecycle authority
- PRMX EVM dispatcher = Hyperlane transport adapter
- Base recipient = verified message sink
- Base bridge = release executor + replay guard
- active collateral = physical custody only

## Contracts and responsibilities

### 1. PRMX EVM — `SettlementExitDispatcher`

New contract on PRMX EVM.

Responsibilities:

- receive authenticated local dispatch requests from the PRMX-side exit
  authority
- format the outbound exit payload
- pay Hyperlane dispatch gas from its own prefunded native balance
- call PRMX Mailbox `dispatch(...)`
- emit an audit event with `exitId` and `messageId`

Key properties:

- validate the supplied dispatch request against pallet `PendingExits` via the
  `0x0801` precompile before formatting a Hyperlane payload
- prevent duplicate Hyperlane dispatch for the same `exitId`
- optional local throttle:
  - `authorizedCaller == address(0)`: permissionless submit, intended C4 mode
  - `authorizedCaller != address(0)`: only that account may submit
- stores:
  - `mailbox`
  - `hook` / gas-paymaster hook (if explicit)
  - `destinationDomain` (Base)
  - `remoteRecipient` (`bytes32`-padded Base `SettlementExitRecipient`)
  - `expectedRouteId`
  - `pendingExitOracle` (`0x0000000000000000000000000000000000000801`)
  - `dispatchedExits(exitId)` / `dispatchedMessages(exitId)`
- `owner` (PRMX-side `HYPERLANE_OWNER`, not the Base Council Safe)
- has `receive()` so Council/operator can top up dispatch gas
- if dispatch funding is insufficient, it **reverts**
  - in a future fully on-chain visible hook, that would roll back
    `request_exit` atomically
  - in the current live oracle-service submit path, the pallet exit remains
    pending and retryable until dispatch funding is restored

Suggested external surface:

```solidity
function dispatchExit(
    uint32 routeId,
    uint64 exitId,
    address recipient,
    uint256 amount
) external returns (bytes32 messageId);
```

Suggested events:

```solidity
event ExitDispatched(
    uint64 indexed exitId,
    bytes32 indexed messageId,
    address indexed recipient,
    uint256 amount,
    uint32 routeId
);
```

### 2. Base EVM — `SettlementExitRecipient`

New contract on Base EVM implementing `IMessageRecipient`.

Responsibilities:

- accept Hyperlane `handle(...)` calls only from Base `Mailbox`
- verify:
  - `origin == PRMX_DOMAIN`
  - `sender == trustedSender` (the padded PRMX dispatcher address)
  - payload `version`
  - payload `routeId == expectedRouteId`
- compute a replay/audit commitment for the exit
- call `SettlementExitBridge.releaseExit(...)`
- emit a message-handled event for observability

Suggested external/admin surface:

```solidity
function handle(uint32 origin, bytes32 sender, bytes calldata message) external payable;
function setTrustedSender(bytes32 next) external;
function setExitBridge(address next) external;
```

Suggested events:

```solidity
event ExitMessageHandled(
    uint64 indexed exitId,
    bytes32 indexed commitment,
    address indexed recipient,
    uint256 amount,
    bool released
);
```

### 3. Base EVM — `SettlementExitBridge`

New PRMX-owned Base-side helper contract.

Responsibilities:

- hold the release replay/idempotency map keyed by `exitId`
- be the only contract allowed to spend from the active collateral for exit release
- pull USDC from the active collateral via `transferFrom`
- emit the canonical `ExitReleased` event that PRMX watchers finalize against

This contract is **not** a reserve holder. It is a stateless executor plus replay guard.

Required one-time owner wiring:

- Council calls `PrmxHypERC20Collateral.approveTokenForBridge(USDC, SettlementExitBridge)` so the bridge can spend from the collateral balance
- Council sets `SettlementExitRecipient` as the bridge's trusted caller

Suggested external/admin surface:

```solidity
function releaseExit(
    uint64 exitId,
    bytes32 commitment,
    address recipient,
    uint256 amount
) external returns (bool released);

function setRecipientContract(address next) external;
```

Suggested storage:

```solidity
mapping(uint64 => bytes32) public processedExitCommitments;
address public recipientContract;
address public collateral;
address public asset;
```

Suggested release semantics:

- `msg.sender` must equal `recipientContract`
- if `processedExitCommitments[exitId] == 0`:
  - store `commitment`
  - call `IERC20(asset).transferFrom(collateral, recipient, amount)`
  - emit `ExitReleased(exitId, recipient, amount, commitment)`
  - return `true`
- if `processedExitCommitments[exitId] == commitment`:
  - idempotent duplicate, no transfer
  - return `false`
- if `processedExitCommitments[exitId] != commitment`:
  - revert `CommitmentMismatch`

Suggested event:

```solidity
event ExitReleased(
    uint64 indexed exitId,
    address indexed recipient,
    uint256 amount,
    bytes32 commitment
);
```

Why this is preferred to extending `PrmxHypERC20Collateral`:

- keeps the custom logic out of the custody contract
- avoids growing the diff against the Hyperlane-derived subclass
- makes replay tracking and release events PRMX-owned without modifying the reserve holder
- if the design changes later, the bridge can be rotated with allowance / trusted-caller updates instead of redeploying the collateral

### 4. Base EVM — `PrmxHypERC20Collateral`

No new outbound release logic is added here.

Responsibilities remain:

- hold the active reserve USDC
- enforce PRMX's source-side deposit gating
- grant allowance to the release bridge via the existing owner-only `approveTokenForBridge(...)` surface inherited from `MovableCollateralRouter`

This keeps the custody contract as close as possible to the current already-reviewed shape.

## Payload schema

Use a versioned ABI payload.

### `ExitMessageV1`

```solidity
abi.encode(
    uint8(1),          // version
    uint32 routeId,    // pallet route id / operational route id
    uint64 exitId,     // pallet exit id
    address recipient, // Base EVM recipient
    uint256 amount     // 6-decimal USDC amount
)
```

Rationale:

- `uint64 exitId` matches pallet state directly
- `routeId` makes misrouting easier to detect and leaves room for future multi-route expansion
- `address recipient` matches the pallet input shape (`H160`)
- `version` allows future format evolution without redeploying every observer first

### Commitment formula

On the Base recipient, compute:

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

Use this as the replay/audit fingerprint stored in `SettlementExitBridge.processedExitCommitments(exitId)`.

Why not use `messageId` directly:

- `IMessageRecipient.handle(...)` does not receive Hyperlane `messageId`
- `origin + sender + decoded payload` is enough to detect duplicates vs mismatched replays

## Runtime hook design

### Target end state

The architectural target is still a real runtime-side `EvmDispatchHook` for
`pallet-prmx-warp-account`.

### Runtime implementation shape

`HyperlaneExitDispatchHook` implements:

```rust
impl WarpDispatchHook<AccountId, Balance, ExitId> for HyperlaneExitDispatchHook
```

Responsibilities:

- ABI-encode `SettlementExitDispatcher.dispatchExit(route_id, exit_id, recipient, amount)`
- invoke PRMX EVM with a transport that produces Frontier-visible logs
- map EVM revert / out-of-gas into `DispatchError`
- use a deterministic pallet-origin EVM caller address

### EVM submitter / caller

The steady C4 design does not rely on caller identity for authority.

- `authorizedCaller = address(0)` makes `dispatchExit(...)` permissionless.
- Any user, relayer helper, oracle-service keeper, or future API worker can
  submit the visible PRMX EVM transaction.
- The dispatcher reads `PendingExits[exitId]` through `0x0801` and rejects any
  payload mismatch.
- A non-zero `authorizedCaller` can be set by the PRMX owner as an emergency
  operational throttle, but it is not the protocol authority.

This avoids both bad alternatives:

- no hidden runtime-internal Mailbox logs
- no trusted oracle-service transport sidecar

### Gas funding model

The runtime hook call should send **zero** value.

The dispatcher contract itself holds native PRMX balance and pays `Mailbox.dispatch` / hook fees from its own balance.

Consequences:

- if dispatcher balance is too low, `dispatchExit(...)` reverts
- in the future fully on-chain visible-hook version, the pallet
  `request_exit` extrinsic would roll back and no orphan `PendingExit` would be
  left behind
- in the current live oracle-service submit model, insufficient dispatcher
  balance leaves a retryable `PendingExit`

This is still the preferred steady-state behavior, even though the current live
bridge period temporarily accepts retryable pending exits to avoid Frontier
invisible dispatches.

### Current live implementation note

The current runtime does **not** use `Runner::call(...)` anymore.

Reason:

- we confirmed on 2026-04-21 that `pallet_evm::Runner::call(...)` from inside a
  Substrate extrinsic emits `pallet_evm::Log` events visible in
  `system.events`, but those logs are **not** returned by Frontier
  `eth_getLogs`
- Hyperlane relayer indexes PRMX as an Ethereum chain via RPC logs
- hidden dispatches therefore advance mailbox / merkle sequence counters
  without giving relayer the logs it needs, which poisons contiguous sequence
  indexing

So the runtime hook currently stays side-effect free, and oracle-service emits
the visible PRMX dispatcher transaction after `request_exit`.

## Sequence of events

### 1. User requests exit on PRMX

- user/API submits `warpAccount.request_exit(routeId, amount, evmRecipient)`
- pallet:
  - checks balance in `pallet-assets(1)`
  - transfers amount to pallet escrow
  - inserts `PendingExits[exitId]`
  - emits `ExitRequested`

### 2. Exit becomes transport-eligible on PRMX

- pallet calls `T::EvmDispatchHook::dispatch_exit(...)`
- current live runtime hook is a no-op that returns `Ok(())`
- the exit remains recorded in `PendingExits[exitId]`
- oracle-service watcher later notices the pending exit and checks whether a
  visible PRMX dispatcher tx already exists for `exitId`

### 3. Keeper/user submits visible PRMX dispatcher tx

- any keeper/user submits a real PRMX EVM transaction to
  `SettlementExitDispatcher.dispatchExit(...)`
- dispatcher:
  - optionally validates `authorizedCaller` only if it is non-zero
  - queries `0x0801.pendingExit(exitId)`
  - rejects missing pending exits
  - rejects any route/recipient/amount mismatch
  - rejects duplicate dispatch for the same `exitId`
  - validates `routeId`
  - ABI-encodes `ExitMessageV1`
  - pays Hyperlane dispatch gas
  - calls PRMX `Mailbox.dispatch(...)`
  - emits `ExitDispatched(exitId, messageId, ...)`

If this submission fails, the pallet exit remains pending and can be retried by
any keeper/user after funding/config is fixed.

### 4. Hyperlane delivers to Base

- validators sign checkpoints
- relayer submits Base `Mailbox.process(...)`
- Base `SettlementExitRecipient.handle(...)` runs
- recipient:
  - verifies mailbox / origin / trusted sender / route id / version
  - computes `commitment`
  - calls `SettlementExitBridge.releaseExit(...)`
- bridge:
  - verifies trusted caller
  - stores / checks `processedExitCommitments[exitId]`
  - pulls USDC from active collateral via `transferFrom`
  - emits `ExitReleased(exitId, recipient, amount, commitment)`

If active reserve is short, the transfer reverts, `handle(...)` reverts, and the message remains undelivered until reserve is refilled and the relayer retries.

### 5. Oracle-service finalizes on PRMX

Keep the current two-phase pallet semantics.

- watcher observes Base `SettlementExitBridge.ExitReleased(exitId, recipient, amount, commitment)`
- watcher extracts the Base EVM tx hash that emitted that log
- watcher calls `warpAccount.finalize_exit(exitId, evm_tx_hash)` on PRMX
- pallet:
  - removes `PendingExits[exitId]`
  - burns escrowed `pallet-assets(1)` balance
  - decrements `BridgeMintedTotal`
  - inserts `FinalizedExitTxHashes[evm_tx_hash]`
  - inserts `ProcessedExits[exitId]`

This preserves the existing correctness boundary:

- no supply burn until Base release is observed
- no silent double-finalize because tx-hash replay guard remains in place

## Oracle-service / watcher changes

Replace the current settlement-vault-centric exit release probe.

Today:

- `getExitReleaseState()` reads `SettlementVault.processedExitIds(exitId)`
- then scans `SettlementVault.ExitReleased(...)`

Under this design:

- `getExitReleaseState()` must read `SettlementExitBridge.processedExitCommitments(exitId)`
  or an equivalent boolean helper
- then scan `SettlementExitBridge.ExitReleased(exitId, ...)`

Minimal migration strategy:

1. add new Base ABI for bridge-based exit release
2. switch route config for canonical exits to the active exit-bridge address
3. keep `finalizeExitOnPrmx(exitId, evmTxHash)` unchanged

For C4 dispatch submission:

- keep the existing oracle-service submitter as an optional keeper
- treat `SettlementExitDispatcher.authorizedCaller() == address(0)` as the
  expected permissionless mode and skip the old operator equality check
- keep the event/log scanning idempotency guard so the keeper does not spam
  already-dispatched exits

## Operational caveat on the current chain generation

The code above stops creating **new** hidden mailbox gaps, but it does not
repair the live PRMX chain generation that already contains them.

On 2026-04-21 we confirmed:

- canonical `request_exit` successfully records the pallet exit
- oracle-service successfully submits a visible PRMX dispatcher tx
- the visible dispatcher emits `ExitDispatched` through Frontier RPC
- but Hyperlane relayer on the current PRMX chain is already stuck expecting
  earlier mailbox / merkle sequences that were advanced by prior invisible
  runtime dispatches

Practical implication:

- the canonical design remains correct
- the current testnet generation is **not** a valid environment for flawless
  end-to-end canonical exit, unless we build special relayer/index repair
  tooling
- the simpler recovery path is a fresh PRMX + Base zero-start after preserving
  the corrected code path

## Security properties

### Strong points

- **No dual ledger on PRMX**: user balances remain authoritative only in `pallet-assets(1)`
- **No direct EOA transport dependency**: runtime-authenticated hook calls the dispatcher
- **Atomic failure on underfunded dispatch**: exit request rolls back if transport cannot be paid
- **Atomic failure on insufficient Base reserve**: Base delivery stays pending until reserve exists
- **Idempotent Base release**: duplicate same-commitment deliveries do not double-release
- **Replay guard on PRMX finalize**: existing `evm_tx_hash` guard remains valid
- **Custody contract stays smaller**: outbound release logic lives in a PRMX-owned bridge, not inside the Hyperlane-derived collateral subclass

### Intentional non-goals

- no attempt to make Base release succeed when reserve is short
- no attempt to burn PRMX supply before Base release is confirmed
- no attempt to preserve direct PRMX EVM `transferRemote()` semantics as a fallback path

## Why this beats the other wiring options

Compared with extending `PrmxHypERC20Collateral` further:

- smaller diff against the Hyperlane-derived custody contract
- easier review and narrower blast radius
- replay logic and release event become explicitly PRMX-owned

Compared with a new outbound precompile or runtime-native Hyperlane adapter:

- Hyperlane ABI / hook / gas logic stays in Solidity, close to the Mailbox
- deployment/rotation is easier than repeated runtime changes
- event surface is better for debugging
- design mirrors the inbound `SettlementBridgeRecipient` shape

Compared with reusing PRMX EVM `HypERC20Synthetic.transferRemote()`:

- no second user-balance ledger is reintroduced
- no need to mirror supply into an EVM synthetic token just to move messages
- monitors do not have to accept permanent mixed semantics

## Implementation checklist

1. Add `SettlementExitDispatcher.sol` on PRMX EVM.
2. Add `SettlementExitRecipient.sol` on Base EVM.
3. Add `SettlementExitBridge.sol` on Base EVM.
4. Wire the active collateral with `approveTokenForBridge(USDC, SettlementExitBridge)`.
5. Add runtime constants/config for:
   - dispatcher address
   - pallet-origin caller address / pallet id
6. Keep runtime `HyperlaneExitDispatchHook` side-effect free until a
   Frontier-visible on-chain transport exists.
   Status: landed in spec `486` and retained through live spec `489`.
7. Update oracle-service exit-release ABI/address watchers to the exit bridge.
   Status: landed in repo and live.
8. Add oracle-service visible PRMX dispatcher submission for pending exits.
   Status: landed in repo and live.
8a. Add `0x0801` read-only pending-exit precompile and have
    `SettlementExitDispatcher` validate against it before dispatch.
    Status: landed in repo on 2026-04-22; runtime spec bumped to `489`;
    deployed live on zero-start r2.
8b. Make dispatcher submission permissionless when
    `authorizedCaller == address(0)` and keep oracle-service as a keeper only.
    Status: landed in repo on 2026-04-22; deployed live on zero-start r2.
9. Update smoke harness so canonical exit uses pallet/API path, not direct PRMX EVM `transferRemote()`.
   Status: landed in repo; legacy / synthetic exit modes are now intentionally
   rejected.
10. Re-run end-to-end cutover smoke and confirm:
   - PRMX `request_exit`
   - PRMX dispatcher `ExitDispatched`
   - Base exit bridge `ExitReleased`
   - PRMX `finalize_exit`
   Status: PASS on 2026-04-22 with `exitId = 6`.
11. If relayer reports missing contiguous mailbox / merkle sequences on PRMX,
    treat the chain generation as poisoned by pre-fix invisible dispatches and
    prefer zero-start over bespoke recovery tooling.

## Zero-start r2 status (updated 2026-04-22)

The poisoned generation referenced above was abandoned and replaced by a fresh
PRMX + Base route generation.

Current live canonical exit route:

- PRMX `SettlementExitDispatcher`:
  - `0x2cfC9e176DA56f352f7Da636ec6529E826D3E701`
  - `pendingExitOracle() = 0x0000000000000000000000000000000000000801`
  - `authorizedCaller() = address(0)`
- Base `SettlementExitBridge`:
  - `0xdCC3B33Abe4A7236f5b399e98632f7c1D8eA1813`
- Base `SettlementExitRecipient`:
  - `0xD76F9dB783690fcD954c0A0d3e03aE2a981F3811`

The Base recipient is now explicitly ISM-aware. This avoids Base Mailbox
falling back to the default DomainRouting ISM, which previously caused
PRMX-origin exits to fail with `No ISM found for origin: 1337090`.

Smoke result:

- `request_exit -> visible PRMX dispatch -> Hyperlane -> Base release ->
  PRMX finalize_exit` passed on zero-start r2
- latest observed full-smoke exit:
  - `exitId = 6`
  - PRMX request tx
    `0xed3f39a9fe6af2d705967d9994ec7f8be86771741b24695ce8e7d66de122a815`
  - Base release tx
    `0xd0efdd75822e99b510c1e7a5c51103b84e0a9c650a9eeca67c904b057563a03a`
  - API stage: `finalized_on_prmx`

Operational note:

- Oracle-service may submit the PRMX EVM dispatcher transaction, but only as a
  keeper. Authority is the pallet `PendingExits` state surfaced through
  `0x0801`, and malformed payloads cannot pass dispatcher validation.
- The old direct PRMX EVM `transferRemote()` smoke path is disabled in repo and
  must not be revived as an operational fallback.
