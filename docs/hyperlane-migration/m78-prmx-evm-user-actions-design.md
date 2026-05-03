# M78 - PRMX EVM User Actions Design

**Status**: Implemented and deployed to the DO PRMX testnet on 2026-04-28.
Runtime spec `503` exposes `0x0808` with linked-account reads, exit, coverage
request, request cancellation, underwriting, and LP orderbook user actions.
TypeScript ABI helpers and frontend EVM routes are wired for exit, request
creation/cancellation, underwriting acceptance, and LP place/buy/cancel actions.
The live `evm-user-actions` smoke passed against testnet spec `503`.

**Linear**: SBP-419 follow-up.

## Goal

Let users operate PRMX from an Ethereum wallet without moving the source of
truth out of the runtime pallets.

The current SBP-419 surface at `0x0000000000000000000000000000000000000807`
solves the read side: Ethereum RPC clients can read policies, requests,
holdings, and orderbook state. This design adds the write side for user-owned
actions while keeping capital accounting, policy lifecycle, and settlement
authority in the existing pallets.

## Decision

Add a new PRMX custom precompile:

`0x0000000000000000000000000000000000000808`

Name: `PrmxUserActionsPrecompile`.

It is separate from `PrmxViewsPrecompile` so SBP-419 remains read-only and safe
for generic RPC/indexing consumers. The new precompile is the only public EVM
entry point for user actions that mutate PRMX runtime state.

The EVM caller is not converted into a spendable PRMX account by address
mapping. Instead:

1. Read `msg.sender` / `handle.context().caller` as the wallet `H160`.
2. Resolve that wallet through `pallet-prmx-account-link::WalletToPrmx`.
3. Execute the underlying pallet operation as the linked PRMX `AccountId`.
4. Revert if the wallet is not linked.

This makes the wallet-link pallet the identity boundary and avoids silently
creating a second balance identity for the same person.

## SBP420 Coordination Note

SBP420 should be based on the latest `main` or `codex/sbp-419-main-cleanup`
state, not an older SBP419 snapshot. The current frontend surfaces for
exit/history/vault/agents/dashboard and the runtime precompile map include
SBP419/SBP428 changes.

Pinned current-generation precompile map:

- `0x0000000000000000000000000000000000000802` = `YieldReport`
- `0x0000000000000000000000000000000000000807` = `Views`
- `0x0000000000000000000000000000000000000808` = `UserActions`

Yield baselines should assume the SBP421 Hyperlane transport path. Do not use
the older direct reporter baseline when designing or testing SBP420.

## Non-goals

- Do not make `0x0807` mutable.
- Do not expose settlement finalization, oracle attestations, reserve-return,
  vault/yield commands, ICA commands, mint, or burn to public EVM users.
- Do not re-enable direct PRMX EVM synthetic-token exits.
- Do not treat an EVM ERC20 balance as the settlement ledger. `pallet-assets(1)`
  remains the source of truth.
- Do not depend on Moonbeam's asset precompile implementation. PRMX can follow
  the same architectural pattern, but the user-action surface is PRMX-specific.

## User actions

Phase 1 should expose the operations that are already public user extrinsics
and have clear ownership checks.

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

These helpers let the frontend detect that `0x0808` exists and check whether
the connected wallet can submit user actions before asking the user to sign a
transaction. They do not replace the richer read model in `0x0807`.

### Exit

```solidity
function requestExit(
    uint32 routeId,
    uint128 amount,
    address baseRecipient
) external returns (uint64 exitId);
```

Maps to the existing `warpAccount.request_exit` behavior:

- debit the linked account's `pallet-assets(1)` mUSDC into escrow
- create `PendingExits`
- emit the existing pallet event
- let the existing dispatcher / watcher path release collateral on Base and
  finalize on PRMX

This is the safest first write action because it starts from an established
canonical route and does not change settlement semantics.

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

Maps to `marketV4.create_underwrite_request`.

The precompile should decode the compact ABI into `EventSpecV4` and reuse the
same validation rules as the pallet: active location, threshold unit, premium,
coverage window, expiry, and sufficient mUSDC for premium escrow.

### Request cancellation

```solidity
function cancelUnderwriteRequest(bytes16 requestId) external;
```

Maps to `marketV4.cancel_underwrite_request`.

The linked account must be the original requester. The existing pallet returns
unfilled premium from escrow.

### Underwrite acceptance

```solidity
function acceptUnderwriteRequest(
    bytes16 requestId,
    uint128 sharesToAccept
) external;
```

Maps to `marketV4.accept_underwrite_request`.

The linked account becomes the underwriter. The existing pallet handles
collateral transfer, first-acceptance policy creation, LP minting, and capital
allocation queueing.

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

Maps to the public LP orderbook calls:

- `place_lp_ask`
- `cancel_lp_ask`
- `buy_lp`

The linked account must own the LP tokens for sell/cancel paths. The existing
orderbook status gates remain authoritative: policies outside `Escrowed` are
not tradeable.

## Required runtime refactor

The current public extrinsics return only `DispatchResult`. EVM callers need
deterministic return values for IDs. Add internal helpers and let both the
extrinsics and the precompile call the same code:

- `warpAccount::do_request_exit(who, routeId, amount, recipient) -> Result<ExitId, DispatchError>`
- `marketV4::do_create_underwrite_request(requester, ...) -> Result<RequestId, DispatchError>`
- `marketV4::do_cancel_underwrite_request(who, requestId) -> DispatchResult`
- `marketV4::do_accept_underwrite_request(who, requestId, shares) -> DispatchResult`
- `orderbookLpV4::do_place_lp_ask(policyId, seller, price, quantity) -> Result<OrderId, DispatchError>`
- `orderbookLpV4::do_cancel_lp_ask(who, orderId) -> DispatchResult`
- `orderbookLpV4::do_buy_lp(who, policyId, maxPrice, quantity) -> DispatchResult`

Each existing extrinsic should become a thin wrapper around the helper. That
keeps Substrate/API behavior and EVM behavior in lockstep.

The precompile should wrap each write in a runtime transaction boundary and map
any `DispatchError` to an EVM revert. No partial storage mutation should remain
if decoding, identity resolution, validation, or downstream dispatch fails.

## Events and logs

The native pallet events remain the canonical audit trail. The precompile should
also emit Ethereum logs so EVM-native tooling can index the actions:

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

The event names are intentionally user-action oriented. Settlement, reserve,
yield, oracle, and admin events remain on their existing operator surfaces.

## Gas and funding

The transaction signer is the EVM wallet, so the wallet's PRMX EVM gas account
must hold enough native gas. The linked PRMX account is used for mUSDC and LP
state, not for paying EVM gas.

For testnet, the practical path is:

- when a wallet link is finalized, drip a small native-gas amount to the EVM
  sender address; or
- expose a faucet action in the operations UI for linked wallets.

For production readiness, choose one of:

- require users to keep a native gas balance on PRMX EVM
- add a governed gas-sponsorship relayer for selected user actions
- add account-abstraction style sponsorship later, after the direct precompile
  path is proven

Phase 1 should not hide this requirement. If the wallet has no gas, the EVM
transaction cannot reach the precompile.

## Frontend behavior

The frontend should keep the current Substrate/API path as fallback and add an
EVM mode when all prerequisites are true:

- injected EVM wallet is connected
- wallet is on PRMX EVM chain
- wallet is linked to a PRMX account
- wallet has PRMX EVM gas
- the runtime exposes `0x0808`

Then:

- Deposit remains Base EVM / Hyperlane collateral lock.
- Read views can continue using `0x0807` or the existing API.
- Exit can use `0x0808.requestExit`.
- Policy creation, request cancellation, underwriting, and LP trading can use
  `0x0808` after the runtime containing this precompile is deployed; the local
  frontend now routes these actions through EVM when the connected EVM wallet is
  linked to the selected PRMX account.
- If any prerequisite fails, show the existing Substrate/API flow instead of
  attempting a transaction that will revert.

## Security checks

Every selector must enforce:

- linked wallet required
- amount/quantity non-zero where applicable
- existing pallet pause/status gates
- existing pallet ownership checks
- existing location/event/coverage validation
- sufficient `pallet-assets(1)` balance where funds are moved
- no privileged origin substitution
- no direct access to mint/burn, settlement, oracle, vault, reserve, ICA, or
  governance calls

The precompile must not accept an arbitrary `bytes32 prmxAccount` argument from
the caller. The linked account must be derived from `msg.sender` on-chain.

## Test plan

Runtime/precompile tests:

- `0x0808` appears in `used_addresses`.
- unknown selector reverts through the existing malformed-call coverage.
- unlinked wallet reverts for write selectors.
- linked wallet resolves to the expected PRMX account.
- `requestExit` produces the same storage changes as the Substrate extrinsic:
  user balance decreases, escrow balance increases, `PendingExits` is inserted,
  and the returned `exitId` matches storage.
- `createUnderwriteRequest` returns the request id and creates the same
  request/escrow state as the extrinsic.
- `cancelUnderwriteRequest` returns unfilled premium and marks the request
  cancelled.
- `acceptUnderwriteRequest` creates/adds policy state and LP holdings through
  the existing pallet logic.
- LP ask placement, buy, and cancellation update the orderbook, buyer/seller
  holdings, and buyer mUSDC balance through the existing pallet logic.
- emitted EVM logs are produced for create/accept/place/buy/exit actions.
- failure inside a downstream pallet rolls back all precompile-side mutations.

Current verification:

- `cargo check -p pallet-evm-precompile-prmx-user-actions`
- `cargo test -p pallet-market-v4 underwrite --lib`
- `cargo test -p pallet-prmx-orderbook-lp-v4 lp --lib`
- `SKIP_WASM_BUILD=1 cargo test -p prmx-runtime evm::tests --lib`
- `npm --prefix frontend run test:run -- prmx-user-actions-precompile prmx-views-precompile`
  (14 tests)
- `cd frontend && ./node_modules/.bin/tsc --noEmit`
- `cargo build --release -p prmx-node`
- Council runtime upgrade proposal `#13`: spec `502` -> `503`
- `node scripts/hyperlane-smoke/prmx-views-readback.mjs --manifest=./scripts/hyperlane-smoke/manifest.do.json`
- `node scripts/hyperlane-smoke/run-smoke.mjs --manifest=./scripts/hyperlane-smoke/manifest.do.json --only=evm-user-actions`
  passed live on DO testnet. Latest pass created policy
  `0x7ceb6dd91601d6e29e1abbf5f5788182`, LP order
  `0x3007c4f128097fac769b6eb89b07839e`, cancelled request
  `0xd7f9c9d1839a0093d61a03b66789c24b`, and EVM exit request `57`
  (`requestExit` tx `0x522a47374003ba1b749defbf485b553eec51d1752b9a9cc43cb796313500957a`).

Frontend tests:

- EVM mode is available only when chain, link, gas, and runtime support are
  present.
- missing link/gas falls back to the existing flow with clear user state.
- calldata builders match the ABI.
- transaction receipt parsing extracts returned ids/logs.

Smoke/live tests:

- optional `evm-user-actions` smoke leg is added for the current testnet
- ensure wallet link and gas before submitting writes
- create a request through `0x0808`
- accept shares through `0x0808`
- place, buy, and cancel an LP ask through `0x0808`
- create and cancel a second request through `0x0808`
- request an exit through `0x0808` and verify the PRMX `PendingExit`
- full Base release / PRMX finalization remains covered by the existing
  canonical exit and API lifecycle Hyperlane gates

## Rollout order

1. Add pallet helper refactors with no behavior change. Done for
   `warpAccount.request_exit`, market V4, and LP orderbook V4.
2. Add `pallet-evm-precompile-prmx-user-actions` and runtime address `0x0808`.
   Done for read helpers, `requestExit`, coverage requests, underwriting, and
   LP orderbook actions.
3. Add Rust tests for identity resolution and each selector. Done for read
   helpers, linked-account enforcement, `requestExit`, create/cancel request,
   accept request, place ask, buy LP, and cancel ask.
4. Add ABI snapshot and TypeScript calldata helpers. Done for all `0x0808`
   selectors, receipt waits, preflight return-id decoding, and V4 progress
   wrappers.
5. Add frontend EVM mode behind runtime capability detection. Done locally for
   exit, policy creation, request cancellation, underwriting acceptance, and LP
   place/buy/cancel. Substrate/API submission remains the fallback.
6. Deploy to testnet, run focused runtime/API tests, then run the optional live
   EVM-user-actions smoke leg. Done on DO testnet spec `503`.
7. Make the EVM path the preferred frontend path for users who have a linked EVM
   wallet connected. Done and deployed to the production Vercel alias.

## Open decisions

- Whether testnet gas should be funded by automatic link-finalization drip or a
  manual faucet button.
- Whether account-link creation itself should later get a PRMX EVM surface, or
  stay on the current AccountLinkRegistry + PRMX pallet finalize path.
