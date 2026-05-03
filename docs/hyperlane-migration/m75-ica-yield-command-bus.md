# M75 — ICA Command Bus and Yield Accrual Flow

ICA (Hyperlane Interchain Accounts) is PRMX's operational control bus to Base. It is **not** the deposit / exit settlement transport. Settlement transport is the pallet-first canonical Hyperlane route described in [m72](/docs/hyperlane-migration/m72-pallet-assets-hyperlane-canonical-path-decision) and [m73](/docs/hyperlane-migration/m73-exit-dispatcher-design).

ICA is used for:

- policy lifecycle operations (`VaultFactory.deployVault`, `PolicyVaultManager.rebalance`, `PolicyVaultManager.returnSettlementCapital`, `PolicyVaultManager.topUpReserve`)
- testnet mock-yield accrual (`MockLendingVault.accrueYield`)

## Route

```text
PRMX operator
→ PRMX EVM InterchainAccountRouter
→ Hyperlane Mailbox / ISM / relayer / validators
→ Base InterchainAccountRouter
→ deterministic Base ICA proxy
→ IcaReceiver.execute(bytes)
→ IcaCommandRouter.execute(target, calldata)
→ allowlisted Base target
```

The Base target sees `msg.sender == IcaCommandRouter`. Operator wiring on Base must therefore set:

- `PolicyVaultManager.setOperator(IcaCommandRouter)`
- `VaultFactory.setOperator(IcaCommandRouter)`
- `IcaReceiver.setTarget(IcaCommandRouter)`

After this wiring, direct Base writes to `PolicyVaultManager` / `VaultFactory` are expected to fail by design. New operation code must either use the ICA dispatch path or submit PRMX-native / pallet operations that later produce relayer-visible Hyperlane messages.

## Components

- Standard Hyperlane `InterchainAccountRouter` is included in the PRMX EVM deployment script.
- Base `IcaReceiver` is the deterministic-proxy and gas-floor gate. It is project-owned and safe to redeploy when the expected ICA owner / proxy tuple changes; do not modify Hyperlane's standard router contracts for this.
- Base `IcaCommandRouter` is the project-owned allowlist layer behind `IcaReceiver`.
- `MockLendingVault` emits `YieldAccrued` when testnet yield is physically minted.
- `oracle-service/src/capital/ica-dispatch.ts` builds PRMX-router ICA calls with 2M gas metadata and quotes the native IGP fee through PRMX `Mailbox.quoteDispatch(...)` when `ICA_DISPATCH_NATIVE_FEE_WEI=0`.
- `oracle-service/src/capital/policy-vault-creation-worker.ts` is ICA-aware: with `ICA_DISPATCH_ENABLED=true`, `VaultFactory.deployVault(...)` and `PolicyVaultManager.rebalance(...)` are dispatched from PRMX and the worker polls Base state for the resulting vault / funding effect.
- `oracle-service/src/capital/settlement-dispatch-worker.ts` is ICA-aware: with `ICA_DISPATCH_ENABLED=true`, `PolicyVaultManager.returnSettlementCapital(...)` is dispatched from PRMX and the worker waits for Base `SettlementCapitalReturned` before finalizing PRMX with the Base tx hash.
- `oracle-service/src/rebalancer/executor/` is ICA-aware: with a complete ICA config, autonomous `PolicyVaultManager.rebalance(...)` decisions are submitted as PRMX-side ICA dispatch transactions.
- `oracle-service/src/capital/yield-accrual-driver.ts` scans PolicyVault strategy assets and dispatches `MockLendingVault.accrueYield(strategy, amount)` via ICA.
- `policyVaultReporter.reportVaultAssets` is wired into `prmxPolicyV4.applyVaultAssetsReport`. `CapitalApiV4Adapter` credits / debits canonical `pallet-assets(1)` plus `warpAccount.bridgeMintedTotal` for yield / loss deltas.

ICA is independent from yield-report transport. The yield accrual driver causes Base-side movement; [m76](/docs/hyperlane-migration/m76-yield-report-hyperlane-transport) describes how Base-side vault totals and rebalance acknowledgements are reported back to PRMX through Hyperlane.

## Initial Allowlist

Council allowlists only the selectors needed for the current goal:

- `VaultFactory.deployVault(bytes32)`
- `PolicyVaultManager.rebalance(bytes32[],int256[])`
- `PolicyVaultManager.topUpReserve(int256)`
- `PolicyVaultManager.returnSettlementCapital(bytes32,uint256)`
- `MockLendingVault.accrueYield(address,uint256)`

Admin setters remain Council motions unless explicitly delegated.

## Deployment / Wire-up

For a clean zero-start:

1. Deploy PRMX EVM core, including `core.icaRouter`.
2. Enroll PRMX ICA router and Base ICA router symmetrically with router + ISM.
3. Derive the Base local ICA proxy for `(origin=PRMX_DOMAIN, owner=PRMX router caller, router=PRMX_ICA_ROUTER, ism=Base ISM)`.
4. Deploy Base stack with `EXPECTED_ICA_PROXY` set to the derived proxy.
5. Council wires `IcaReceiver.target = IcaCommandRouter`.
6. Council allowlists the initial selectors on `IcaCommandRouter`.
7. Council rotates `PolicyVaultManager.operator` and `VaultFactory.operator` to `IcaCommandRouter`.
8. Enable oracle-service ICA / yield env only after steps 1–7 complete.

For an already-live generation, use the incremental scripts instead of rerunning the whole Hyperlane deployment:

```bash
scripts/hyperlane/deploy-prmx-ica-router.sh
scripts/hyperlane/deploy-base-ica-command-bus.sh
node scripts/hyperlane/sync-zero-start-configs.mjs \
  --base-manifest evm/deployments/base-sepolia/hyperlane-manifest.json \
  --prmx-manifest evm/deployments/prmx-evm/hyperlane-manifest.json \
  --smoke-manifests scripts/hyperlane-smoke/manifest.do.json \
  --canonical-inbound
GAS_PRICE_WEI=1000000000 scripts/hyperlane/execute-ica-precutover-wireup.sh
```

`execute-ica-precutover-wireup.sh` intentionally stops before the two operator rotations on Base; those motions are submitted manually so they remain a deliberate gate.

`ica-dispatch.ts` submits to the PRMX EVM router from the operator key. The architectural target is a chain-governed authority (Substrate extrinsic) rather than EOA-governed dispatch, but a pallet calling the PRMX Mailbox through `pallet_evm::Runner::call(...)` is intentionally avoided: runtime-internal EVM calls emit `pallet_evm::Log` events visible in `system.events` but **not** in Frontier `eth_getLogs`, so Hyperlane relayers cannot index them. The current relayer-visible PRMX EVM transaction is the safe scaffold.

## Wire-up Verification Harness

Before enabling any ICA / yield env flag, run:

```bash
node scripts/hyperlane/ica-wireup-check.mjs \
  --manifest scripts/hyperlane-smoke/manifest.do.json \
  --base-rpc https://sepolia.base.org \
  --prmx-rpc http://<prmx-host>:9944 \
  --phase cutover \
  --strict
```

If the PRMX owner EOA is not encoded in the deployed `IcaReceiver.prmxOwner()`, pass it explicitly:

```bash
node scripts/hyperlane/ica-wireup-check.mjs \
  --manifest scripts/hyperlane-smoke/manifest.do.json \
  --prmx-owner 0x... \
  --phase cutover \
  --strict
```

Phase semantics:

- `--phase deploy`: immediately after deployment
- `--phase wireup`: after router enrollment, target, and allowlist motions; operator checks are reported as `INFO` so wireup can be verified without breaking direct-operator workers
- `--phase cutover`: full operator rotation expected to be complete

The script never sends transactions. It checks router enrollment in both directions, ICA proxy derivation, `IcaReceiver.target`, `IcaCommandRouter.icaReceiver`, `PolicyVaultManager.operator`, `VaultFactory.operator`, and initial command allowlist entries. With `--strict`, `TODO` ("wire-up still missing") and `FAIL` ("unexpected state or stale address") exit non-zero. `INFO` is advisory for the selected phase. `SKIP` is used only when an optional surface or required manifest field is absent.

The script also prints candidate calldata for owner / Safe actions:

- `InterchainAccountRouter.enrollRemoteRouterAndIsm(...)` on PRMX and Base
- `IcaReceiver.setTarget(IcaCommandRouter)`
- `IcaCommandRouter.setCommandPermissions(...)`
- `PolicyVaultManager.setOperator(IcaCommandRouter)`
- `VaultFactory.setOperator(IcaCommandRouter)`

## Yield Accrual Operator Surface

To make the 24h driver testable without waiting a full day, oracle-service exposes:

- `GET /capital/yield-accrual/status`
- `POST /capital/operator/run-yield-accrual`

The status endpoint reports whether the driver is enabled / running, whether a cycle is in flight, config pins (`intervalMs`, `routeId`, `annualYieldBps`, min/max accrual, max policies), `nextCycleAtMs`, last cycle timestamps and elapsed window, last cycle summary (`scanned`, `eligible`, `dispatched`, `totalAccrued`, optional `skipped`), and last error.

The operator endpoint executes a single yield-accrual cycle immediately and returns the same summary with `totalAccrued` serialized as a string. On a fresh process where `lastCycleAtMs` is unset, the first manual run uses the configured interval as the effective elapsed window — with the default `YIELD_ACCRUAL_INTERVAL_MS=86400000` it acts as a manual 24h-equivalent validation run.

The operator endpoint does not authorize leaving `YIELD_ACCRUAL_DRIVER_ENABLED=true` permanently. The intended cadence is:

1. keep the autonomous driver disabled
2. use `POST /capital/operator/run-yield-accrual` for an explicit cadence probe
3. follow with a report cycle and invariant readback
4. only then decide whether to run a recurring soak

## Env Surface

ICA dispatch:

```dotenv
ICA_DISPATCH_ENABLED=true
PRMX_ICA_ROUTER_ADDRESS=0x...
PRMX_ICA_MAILBOX_ADDRESS=0x...
BASE_ICA_RECEIVER_ADDRESS=0x...
BASE_ICA_COMMAND_ROUTER_ADDRESS=0x...
BASE_ICA_ROUTER_ADDRESS=0x...
BASE_ICA_ISM_ADDRESS=0x...
PRMX_ICA_DISPATCH_HOOK_ADDRESS=0x...
ICA_BASE_DOMAIN=84532
ICA_DISPATCH_GAS_LIMIT=2000000
ICA_DISPATCH_FEE_BPS=12000
ICA_DISPATCH_PRMX_TX_GAS=2500000
```

Yield accrual:

```dotenv
YIELD_ACCRUAL_DRIVER_ENABLED=true
YIELD_ACCRUAL_ROUTE_ID=2
YIELD_ACCRUAL_INTERVAL_MS=86400000
YIELD_ACCRUAL_ANNUAL_BPS=500
```

Autonomous rebalancing:

```dotenv
REBALANCER_EXECUTOR_ENABLED=true
REBALANCER_DECISION_ENABLED=true
REBALANCER_MONITOR_ENABLED=true
```

The executor prefers ICA whenever `ICA_DISPATCH_ENABLED=true` and the full ICA config is present. If ICA is explicitly enabled but incomplete, the executor stays dormant instead of silently falling back to direct Base writes.

`ICA_DISPATCH_PRMX_TX_GAS=2500000` is the code default. PRMX EVM `eth_estimateGas` can over-estimate ICA router dispatches above the block gas limit; the explicit cap prevents that.
