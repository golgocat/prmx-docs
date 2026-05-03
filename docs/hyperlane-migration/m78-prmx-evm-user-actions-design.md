# M78 — PRMX EVM User Actions Design

> **TL;DR**: The PRMX EVM exposes a write surface at `0x0808` (`PrmxUserActionsPrecompile`) so users can operate PRMX from an Ethereum wallet without moving the source of truth out of the runtime pallets. The runtime stays the authority; the EVM is the entry point.

## Goal

Let users operate PRMX from an Ethereum wallet while keeping capital accounting, policy lifecycle, and settlement authority inside the existing pallets. The companion `PrmxViewsPrecompile` at `0x0807` covers reads (policies, requests, holdings, orderbook); `0x0808` is the write counterpart.

## Identity model

The EVM caller is **not** converted into a spendable PRMX account. Resolution is:

1. Read `msg.sender` / `handle.context().caller` as the wallet `H160`.
2. Resolve through `pallet-prmx-account-link::WalletToPrmx`.
3. Execute the underlying pallet operation as the linked PRMX `AccountId`.
4. Revert if the wallet is not linked.

This makes the wallet-link pallet the identity boundary and avoids silently creating a second balance identity for the same person.

## Precompile Map

| Address | Name | Surface |
|---|---|---|
| `0x0000000000000000000000000000000000000800` | `SettlementAsset` | Mint (Hyperlane inbound) |
| `0x0000000000000000000000000000000000000801` | `WarpAccount` | Pending-exit reads |
| `0x0000000000000000000000000000000000000802` | `YieldReport` | Vault asset reports inbound |
| `0x0000000000000000000000000000000000000807` | `Views` | Read-only state |
| `0x0000000000000000000000000000000000000808` | `UserActions` | **This design** — the only public EVM entry point for state-mutating user actions |

## Non-goals

- `0x0807` stays read-only.
- Settlement finalization, oracle attestations, reserve-return, vault/yield commands, ICA commands, mint, and burn are **never** exposed to public EVM users.
- No direct PRMX EVM synthetic-token exits.
- EVM ERC20 balances are not the settlement ledger — `pallet-assets(1)` stays SSoT.
- No dependency on Moonbeam's asset precompile implementation. The user-action surface is PRMX-specific.

## User actions

Phase 1 covers the public user extrinsics that already have clear ownership checks:

| Action | Selector | Underlying pallet call |
|---|---|---|
| Exit | `requestExit` | `warpAccount.request_exit` |
| Create coverage request | `createUnderwriteRequest` | `marketV4.create_underwrite_request` |
| Cancel coverage request | `cancelUnderwriteRequest` | `marketV4.cancel_underwrite_request` |
| Accept underwriting | `acceptUnderwriteRequest` | `marketV4.accept_underwrite_request` |
| Place LP ask | `placeLpAsk` | `orderbookLpV4.place_lp_ask` |
| Cancel LP ask | `cancelLpAsk` | `orderbookLpV4.cancel_lp_ask` |
| Buy LP | `buyLp` | `orderbookLpV4.buy_lp` |

### Read helpers

```solidity
function version() external pure returns (uint32);

function linkedAccount(address wallet)
    external
    view
    returns (bool exists, bytes32 prmxAccount);

function linkedAccountOfCaller()
    external
    view
    returns (bool exists, bytes32 prmxAccount);
```

These helpers let the frontend detect that `0x0808` exists and check whether the connected wallet can submit user actions before signing. They do not replace the richer read model in `0x0807`.

### Exit

```solidity
function requestExit(
    uint32 routeId,
    uint128 amount,
    address baseRecipient
) external returns (uint64 exitId);
```

Effects (identical to `warpAccount.request_exit`): debit the linked account's `pallet-assets(1)` mUSDC into escrow, create `PendingExits`, emit the pallet event, then the existing dispatcher / watcher path releases collateral on Base and finalizes on PRMX. Safest first write action — established canonical route, no settlement-semantics change.

### Coverage request

```solidity
function createUnderwriteRequest(
    uint64 locationId,
    uint8 eventType,
    int64 thresholdValue,
    uint8 thresholdUnit,
    bool earlyTrigger,
    uint128 totalShares,
    uint128 premiumPerShare,
    uint64 coverageStart,
    uint64 coverageEnd,
    uint64 expiresAt
) external returns (bytes16 requestId);
```

The precompile decodes the compact ABI into `EventSpecV4` and reuses the pallet's validation rules: active location, threshold unit, premium, coverage window, expiry, and sufficient mUSDC for premium escrow.

### Request cancellation

```solidity
function cancelUnderwriteRequest(bytes16 requestId) external;
```

Linked account must be the original requester. The pallet returns unfilled premium from escrow.

### Underwrite acceptance

```solidity
function acceptUnderwriteRequest(
    bytes16 requestId,
    uint128 sharesToAccept
) external;
```

The linked account becomes the underwriter. The pallet handles collateral transfer, first-acceptance policy creation, LP minting, and capital allocation queueing.

### LP orderbook

```solidity
function placeLpAsk(
    bytes16 policyId,
    uint128 price,
    uint128 quantity
) external returns (bytes16 orderId);

function cancelLpAsk(bytes16 orderId) external;

function buyLp(
    bytes16 policyId,
    uint128 maxPrice,
    uint128 quantity
) external;
```

The linked account must own the LP tokens for sell/cancel paths. Existing orderbook status gates remain authoritative: policies outside `Escrowed` are not tradeable.

## Required runtime refactor

Public extrinsics today return only `DispatchResult`; EVM callers need deterministic IDs. Introduce internal helpers and make extrinsics thin wrappers over them so Substrate API behavior and EVM behavior stay in lockstep:

| Helper | Returns |
|---|---|
| `warpAccount::do_request_exit(who, routeId, amount, recipient)` | `Result<ExitId, DispatchError>` |
| `marketV4::do_create_underwrite_request(requester, ...)` | `Result<RequestId, DispatchError>` |
| `marketV4::do_cancel_underwrite_request(who, requestId)` | `DispatchResult` |
| `marketV4::do_accept_underwrite_request(who, requestId, shares)` | `DispatchResult` |
| `orderbookLpV4::do_place_lp_ask(policyId, seller, price, quantity)` | `Result<OrderId, DispatchError>` |
| `orderbookLpV4::do_cancel_lp_ask(who, orderId)` | `DispatchResult` |
| `orderbookLpV4::do_buy_lp(who, policyId, maxPrice, quantity)` | `DispatchResult` |

The precompile wraps each write in a runtime transaction boundary and maps `DispatchError` to an EVM revert. No partial storage mutation may survive a failure in decoding, identity resolution, validation, or downstream dispatch.

## Events and logs

Native pallet events remain the canonical audit trail. The precompile additionally emits Ethereum logs so EVM-native tooling can index user actions:

```solidity
event EvmExitRequested(
    address indexed wallet,
    bytes32 indexed prmxAccount,
    uint64 indexed exitId,
    uint32 routeId,
    address baseRecipient,
    uint256 amount
);

event EvmUnderwriteRequestCreated(
    address indexed wallet,
    bytes32 indexed prmxAccount,
    bytes16 indexed requestId
);

event EvmUnderwriteRequestAccepted(
    address indexed wallet,
    bytes32 indexed prmxAccount,
    bytes16 indexed requestId,
    uint256 shares
);

event EvmLpAskPlaced(
    address indexed wallet,
    bytes32 indexed prmxAccount,
    bytes16 indexed orderId,
    bytes16 policyId
);

event EvmLpTradeSubmitted(
    address indexed wallet,
    bytes32 indexed prmxAccount,
    bytes16 indexed policyId,
    uint256 quantity
);
```

Names are intentionally user-action oriented. Settlement, reserve, yield, oracle, and admin events stay on their existing operator surfaces.

## Gas and funding

The signer is the EVM wallet, so the wallet's PRMX EVM gas account must hold native gas. The linked PRMX account is used for mUSDC and LP state, **not** for paying EVM gas.

| Stage | Gas funding option |
|---|---|
| Testnet | Drip native gas at wallet-link finalization, or expose a faucet action in the ops UI for linked wallets |
| Production | Require users to keep a native gas balance, add a governed gas-sponsorship relayer for selected actions, or add account-abstraction sponsorship after the direct precompile path is proven |

Phase 1 must not hide the requirement: if the wallet has no gas, the EVM transaction cannot reach the precompile.

## Frontend behavior

EVM mode activates only when **all** prerequisites are true; otherwise the existing Substrate/API flow is shown.

| Prerequisite |
|---|
| Injected EVM wallet connected |
| Wallet on PRMX EVM chain |
| Wallet linked to a PRMX account |
| Wallet has PRMX EVM gas |
| Runtime exposes `0x0808` |

| Action | Path |
|---|---|
| Deposit | Always Base EVM / Hyperlane collateral lock |
| Reads | `0x0807` or existing API |
| Exit | `0x0808.requestExit` |
| Policy creation, request cancel, underwriting, LP trading | `0x0808` when all prerequisites are met |

## Security checks

Every selector must enforce:

- Linked wallet required (no arbitrary `bytes32 prmxAccount` argument — derive from `msg.sender`).
- Amount/quantity non-zero where applicable.
- Existing pallet pause/status gates and ownership checks.
- Existing location/event/coverage validation.
- Sufficient `pallet-assets(1)` balance where funds are moved.
- No privileged origin substitution.
- No direct access to mint/burn, settlement, oracle, vault, reserve, ICA, or governance calls.

## Test plan

**Runtime/precompile:**

- `0x0808` appears in `used_addresses`.
- Unknown selector reverts through existing malformed-call coverage.
- Unlinked wallet reverts for write selectors.
- Linked wallet resolves to the expected PRMX account.
- `requestExit` storage changes match the Substrate extrinsic (balance debit, escrow credit, `PendingExits` inserted, returned `exitId` matches storage).
- `createUnderwriteRequest` returns the request id and creates the same request/escrow state as the extrinsic.
- `cancelUnderwriteRequest` returns unfilled premium and marks the request cancelled.
- `acceptUnderwriteRequest` mutates policy state and LP holdings through existing pallet logic.
- LP ask placement, buy, and cancellation update orderbook, buyer/seller holdings, and buyer mUSDC balance through existing pallet logic.
- EVM logs are emitted for create/accept/place/buy/exit actions.
- A failure inside any downstream pallet rolls back all precompile-side mutations.

**Verification commands:**

```bash
cargo check -p pallet-evm-precompile-prmx-user-actions
cargo test -p pallet-market-v4 underwrite --lib
cargo test -p pallet-prmx-orderbook-lp-v4 lp --lib
SKIP_WASM_BUILD=1 cargo test -p prmx-runtime evm::tests --lib
npm --prefix frontend run test:run -- prmx-user-actions-precompile prmx-views-precompile
cd frontend && ./node_modules/.bin/tsc --noEmit
cargo build --release -p prmx-node
node scripts/hyperlane-smoke/prmx-views-readback.mjs --manifest=./scripts/hyperlane-smoke/manifest.do.json
node scripts/hyperlane-smoke/run-smoke.mjs --manifest=./scripts/hyperlane-smoke/manifest.do.json --only=evm-user-actions
```

**Frontend:** EVM mode is exposed only when chain, link, gas, and runtime support are all present; missing link/gas falls back with clear user state; calldata builders match the ABI; receipt parsing extracts returned ids/logs.

**Smoke/live:** optional `evm-user-actions` leg ensures wallet link + gas, then walks create → accept → place/buy/cancel LP ask → cancel a second request → request an exit → verify `PendingExit`. Full Base release / PRMX finalization stays covered by the existing canonical exit and API lifecycle Hyperlane gates.

## Open decisions

| Decision |
|---|
| Whether testnet gas is funded via automatic link-finalization drip or a manual faucet button |
| Whether account-link creation itself eventually gets a PRMX EVM surface, or stays on the current `AccountLinkRegistry` + PRMX pallet finalize path |
