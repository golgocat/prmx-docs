# M75 — ICA Command Bus and Yield Accrual Flow

Date: 2026-04-22

## Fixed Scope

ICA is for operational control, not settlement transport.

- Deposits stay on the normal Hyperlane Warp/canonical inbound path.
- Exits stay on the normal pallet-first Hyperlane outbound path.
- ICA is used for policy lifecycle operations, capital allocation, reserve/yield operations, and testnet mock-yield accrual.

## Route

Operational command route:

```text
PRMX operator / future Substrate extrinsic
-> PRMX EVM InterchainAccountRouter
-> Hyperlane Mailbox / ISM / relayer / validators
-> Base InterchainAccountRouter
-> deterministic Base ICA proxy
-> IcaReceiver.execute(bytes)
-> IcaCommandRouter.execute(target, calldata)
-> allowlisted Base target
```

The Base target should see `msg.sender == IcaCommandRouter`. Therefore post-deploy operator wiring must set:

- `PolicyVaultManager.setOperator(icaCommandRouter)`
- `VaultFactory.setOperator(icaCommandRouter)`
- `IcaReceiver.setTarget(icaCommandRouter)`

## Components

- Standard Hyperlane `InterchainAccountRouter` is now included in the PRMX EVM deployment script.
- Base `IcaReceiver` remains the deterministic-proxy and gas-floor gate. It is
  safe to redeploy this project-owned contract when the expected ICA owner/proxy
  tuple changes; do not modify Hyperlane's standard router contracts for this.
- New Base `IcaCommandRouter` is the project-owned allowlist layer behind `IcaReceiver`.
- `MockLendingVault` now emits `YieldAccrued` when testnet yield is physically minted.
- `oracle-service/src/capital/ica-dispatch.ts` builds PRMX-router ICA calls with 2M gas metadata and quotes native IGP fee through PRMX `Mailbox.quoteDispatch(...)` when `ICA_DISPATCH_NATIVE_FEE_WEI=0`.
- `oracle-service/src/capital/policy-vault-creation-worker.ts` is now ICA-aware: with `ICA_DISPATCH_ENABLED=true`, `VaultFactory.deployVault(...)` and `PolicyVaultManager.rebalance(...)` are dispatched from PRMX and the worker polls Base state for the resulting vault/funding effect.
- `oracle-service/src/capital/settlement-dispatch-worker.ts` is now ICA-aware: with `ICA_DISPATCH_ENABLED=true`, `PolicyVaultManager.returnSettlementCapital(...)` is dispatched from PRMX and the worker waits for Base `SettlementCapitalReturned` before finalizing PRMX with the Base tx hash.
- `oracle-service/src/rebalancer/executor/` is now ICA-aware: with a complete ICA config, autonomous `PolicyVaultManager.rebalance(...)` decisions are submitted as PRMX-side ICA dispatch txs. The Base effect is asynchronous and is later observed by `VaultReporter` through `RebalanceExecuted` logs.
- `oracle-service/src/capital/yield-accrual-driver.ts` scans PolicyVault strategy assets and dispatches `MockLendingVault.accrueYield(strategy, amount)` via ICA.
- Runtime spec `491` wires `policyVaultReporter.reportVaultAssets` into
  `prmxPolicyV4.applyVaultAssetsReport`, and `CapitalApiV4Adapter` now
  credits/debits canonical `pallet-assets(1)` plus
  `warpAccount.bridgeMintedTotal` for yield/loss deltas.
- SBP-421 adds an optional Hyperlane transport for the report path itself:
  Base `YieldReporter` dispatches fixed yield-report envelopes through the
  Hyperlane Mailbox to PRMX `YieldReportRecipient`, which calls precompile
  `0x0802` and then `pallet-prmx-policy-vault-reporter`. Default remains
  `VAULT_REPORTER_TRANSPORT=direct`; see
  `docs/hyperlane-migration/m76-yield-report-hyperlane-transport.md`.

## Initial Allowlist

The Council should allow only the selectors needed for the current goal:

- `VaultFactory.deployVault(bytes32)`
- `PolicyVaultManager.rebalance(bytes32[],int256[])`
- `PolicyVaultManager.topUpReserve(int256)`
- `PolicyVaultManager.returnSettlementCapital(bytes32,uint256)`
- `MockLendingVault.accrueYield(address,uint256)`

Admin setters remain Council motions unless we explicitly decide to delegate them.

## Deployment / Wire-Up Notes

For a clean zero-start:

1. Deploy PRMX EVM core, including `core.icaRouter`.
2. Enroll PRMX ICA router and Base ICA router symmetrically with router + ISM.
3. Derive the Base local ICA proxy for `(origin=PRMX_DOMAIN, owner=PRMX router caller, router=PRMX_ICA_ROUTER, ism=Base ISM)`.
4. Deploy Base stack with `EXPECTED_ICA_PROXY` set to the derived proxy.
5. Council wires `IcaReceiver.target = IcaCommandRouter`.
6. Council allowlists the initial selectors on `IcaCommandRouter`.
7. Council rotates `PolicyVaultManager.operator` and `VaultFactory.operator` to `IcaCommandRouter`.
8. Enable oracle-service ICA/yield env only after steps 1-7 are complete.

For an already-live generation, use the incremental scripts instead of
rerunning the whole Hyperlane deployment:

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

`execute-ica-precutover-wireup.sh` intentionally stops before
`PolicyVaultManager.setOperator(icaCommandRouter)` and
`VaultFactory.setOperator(icaCommandRouter)`. Those two motions are final
cutover, because they move live `onlyOperator` authority away from the direct
operator EOA.

Current implementation note: `ica-dispatch.ts` can submit to the PRMX EVM router from the operator key as a controlled testnet scaffold. The desired final SBP-404 D3 authority model is still chain-governed rather than EOA-governed, but the exact runtime-to-visible-EVM mechanism remains pending.

Important runtime note: do not implement D3 by having a Substrate extrinsic call
the PRMX Mailbox through `pallet_evm::Runner::call(...)` until the Frontier log
visibility issue is solved. The C4 investigation proved that runtime-internal
EVM calls can create mailbox activity visible in `system.events` but invisible
to Hyperlane relayer `eth_getLogs`. The current safe scaffold is therefore a
relayer-visible PRMX EVM transaction to the ICA router.

## Wire-Up Verification Harness

Use the read-only harness before enabling any ICA/yield env flags:

```bash
node scripts/hyperlane/ica-wireup-check.mjs \
  --manifest scripts/hyperlane-smoke/manifest.do.json \
  --base-rpc https://sepolia.base.org \
  --prmx-rpc http://159.223.218.107:9944 \
  --phase cutover \
  --strict
```

If the PRMX owner EOA is not already encoded in the deployed
`IcaReceiver.prmxOwner()`, pass it explicitly:

```bash
node scripts/hyperlane/ica-wireup-check.mjs \
  --manifest scripts/hyperlane-smoke/manifest.do.json \
  --prmx-owner 0x... \
  --phase cutover \
  --strict
```

Use `--phase deploy` immediately after deployment, `--phase wireup` after router
enrollment / target / allowlist motions, and `--phase cutover` only when
operator rotation is meant to be complete. In `wireup`, operator checks are
reported as `INFO` so pre-cutover can be verified without breaking the current
direct-operator workers.

The script never sends transactions. It checks:

- PRMX ICA router enrollment for Base router + Base ISM.
- Base ICA router enrollment for PRMX router + PRMX ISM.
- Base deterministic ICA proxy derivation against `IcaReceiver.expectedIcaProxy`.
- `IcaReceiver.target == IcaCommandRouter`.
- `IcaCommandRouter.icaReceiver == IcaReceiver`.
- `PolicyVaultManager.operator == IcaCommandRouter`.
- `VaultFactory.operator == IcaCommandRouter`.
- Initial `IcaCommandRouter.allowedCommand(target, selector)` entries.

It also prints candidate calldata for the required owner/Safe actions:

- `InterchainAccountRouter.enrollRemoteRouterAndIsm(...)` on PRMX.
- `InterchainAccountRouter.enrollRemoteRouterAndIsm(...)` on Base.
- `IcaReceiver.setTarget(icaCommandRouter)`.
- `IcaCommandRouter.setCommandPermissions(...)`.
- `PolicyVaultManager.setOperator(icaCommandRouter)`.
- `VaultFactory.setOperator(icaCommandRouter)`.

Treat `TODO` as "wire-up still missing" and `FAIL` as "unexpected state or stale
address." With `--strict`, either status exits non-zero. `INFO` is advisory for
the selected phase. `SKIP` is used only when an optional surface or required
manifest field is absent, so a clean live cutover should be all `PASS` except
explicitly optional mock-yield entries.

## Historical Testnet Pre-Cutover Record — zero-start r2 (2026-04-22)

The zero-start r2 generation completed SBP-404 pre-cutover wire-up, but these
addresses are superseded by the r3 record below:

- PRMX `InterchainAccountRouter`: `0x53460E2F6203C7a7D87Cbdd2836D4F707cB1970b`
- Base `IcaReceiver`: `0x11F73402697026876EAa515fB1D7217F6CE3EAC8`
- Base `IcaCommandRouter`: `0xbEf7d43DbD642Cadf84a45166009a7cF7afa0c9f`
- Base expected ICA proxy: `0xAdF1ff04766e5649C1B3B5cd870945A3469A22dc`
- PRMX ICA owner scaffold: `0x18B80F1F9ab9B27492c222c81F6932080f97f2fd`
- ICA owner bytes32:
  `0x00000000000000000000000018B80F1F9ab9B27492c222c81F6932080f97f2fd`

Important: the earlier Base `IcaReceiver`
`0x4aa1A23798f73bB998d3e15f06927BCBBDA95802` is superseded for ICA command bus
use. It was deployed with the raw Alice Substrate AccountId as `prmxOwner`,
while the standard Hyperlane `InterchainAccountRouter` derives ownership from
the PRMX EVM caller address. The current scaffold therefore uses the operator
EOA encoded as bytes32.

Executed transactions:

- PRMX router enrollment tx:
  `0x2d9710ac96cfc89b952ab61276e2cd543bfef94fa35d9ae7b364c515d27ff5ab`
- Base router enrollment Safe exec:
  `0x8d1574ed6f49539b3e441cbfbef07997ccb4f0b4ba200c23fd8b0a3a6392e0c6`
- `IcaReceiver.setTarget(icaCommandRouter)` Safe exec:
  `0x4d096eb488b514a8c6cf0bd42905e8e56532b7fcfbdaf0b6c286cabfe9914d80`
- `IcaCommandRouter.setCommandPermissions(...)` Safe exec:
  `0x996f9a60ee6d371d6959445d9930ac3762a6a6140ed726b53ffa89bdc2228d3f`

Verification:

```bash
node scripts/hyperlane/ica-wireup-check.mjs \
  --manifest scripts/hyperlane-smoke/manifest.do.json \
  --prmx-owner 0x18B80F1F9ab9B27492c222c81F6932080f97f2fd \
  --phase wireup \
  --strict
```

Result: `PASS`.

The command-path smoke is now available as an opt-in scenario:

```bash
SMOKE_BASE_RPC_URL=https://base-sepolia.drpc.org \
ICA_COMMAND_SMOKE_AMOUNT_ATOMS=1 \
node scripts/hyperlane-smoke/run-smoke.mjs \
  --manifest=./scripts/hyperlane-smoke/manifest.do.json \
  --only=ica-command
```

Live result on 2026-04-22: `PASS`.

- PRMX dispatch tx:
  `0x56f558454c48b427f21dff0334ab4c85e7920772e31ef682cb3adfc784d04b10`
- Base execution tx:
  `0xd0b11f46e504307ec2eebddc59c5ef23511f0a314a5675a6b8243edf870f1956`
- Base route observed: `IcaReceiver.IcaCallForwarded` and
  `IcaCommandRouter.CommandExecuted`
- `MockLendingVault.balanceOf(mockYieldStrategy)` delta: `11` atoms for a
  requested `1` atom command. The extra delta is expected pending-yield
  settlement in the mock lending vault.
- `yieldEventObserved=false` on the live contract. This means the command path
  is working. SBP-405 E2 now deliberately uses command/balance evidence and
  PRMX accounting deltas; `YieldAccrued` remains diagnostic only on this live
  generation.

The yield-flow E2 scenario is available as an opt-in pre-soak gate:

```bash
SBP405_YIELD_POLICY_ID=0x... \
SMOKE_BASE_RPC_URL=https://base-sepolia.drpc.org \
node scripts/hyperlane-smoke/run-smoke.mjs \
  --manifest=./scripts/hyperlane-smoke/manifest.do.json \
  --only=yield-event
```

Live result on 2026-04-22: `PASS` after runtime upgrade `490 -> 491` and
rewiring `PolicyVaultManager.defaultStrategy` to the r3 `MockYieldStrategy`.

- Runtime upgrade code hash:
  `0x7970f33bbdd6e704d7231f724a38647e5e706d09b4ce9d6a3bb66dc44498c1a9`
- Default strategy Safe exec:
  `0x83cd9eb76ecff850cd246ed974c4cf7c361484de0b54a73d1a8353ea8c2738b1`
- Policy:
  `0x305c542443ccf7df7bb2751298678a72`
- Vault:
  `0x570029646D76B5CebC46d32b038a13C12ABe01fe`
- PRMX ICA dispatch:
  `0x2bb16e78d9ae05db169e0ee07f24c9b50c60b5b118d839a09a5ed55ad590ac22`
- Base command execution:
  `0x17e77124b9ec5da91761d8425ab9c032e6a2256913cd35e457ef4312cb200627`
- Observed deltas:
  - Base PolicyVault totalAssets `+166669`
  - reporter snapshot `+166668`
  - `prmxPolicyV4.policyPoolBalance` `+166668`
  - `pallet-assets(1)` supply `+166668`
  - `warpAccount.bridgeMintedTotal` `+166668`
- `yieldEventObserved=false`; accepted for this generation because command,
  strategy balance, vault totalAssets, reporter, pool, supply, and bridge
  liability evidence all moved consistently.

## SBP-405 E1 operational surface (2026-04-24)

To make the dormant 24h driver testable without waiting a full day, oracle-service
now exposes a small operator/readback surface:

- `GET /capital/yield-accrual/status`
- `POST /capital/operator/run-yield-accrual`

The status endpoint reports:

- whether the driver is enabled/running
- whether a cycle is currently in flight
- config pins (`intervalMs`, `routeId`, `annualYieldBps`, min/max accrual, max policies)
- `nextCycleAtMs`
- the last cycle timestamps and elapsed window
- the last cycle summary (`scanned`, `eligible`, `dispatched`, `totalAccrued`, optional `skipped`)
- the last error, if any

The operator endpoint executes a single yield-accrual cycle immediately and
returns the same summary with `totalAccrued` serialized as a string. On a fresh
process where `lastCycleAtMs` is still unset, the first manual run uses the
configured interval as the effective elapsed window, so with the default
`YIELD_ACCRUAL_INTERVAL_MS=86400000` it acts as a "manual 24h-equivalent"
validation run.

This does **not** authorize leaving `YIELD_ACCRUAL_DRIVER_ENABLED=true`
permanently yet. The intended next gate is:

1. keep the autonomous driver disabled on DO
2. use `POST /capital/operator/run-yield-accrual` for an explicit cadence probe
3. follow it with a report cycle / invariant readback
4. only then decide whether to run a shortened soak or the real 24h cadence

Live result on 2026-04-24: `PASS` on the current `hyperlane` generation.

- `GET /capital/yield-accrual/status` before the probe:
  - `enabled=false`
  - `running=false`
  - `lastResult=null`
- `POST /capital/operator/run-yield-accrual`:
  - `scanned=91`
  - `eligible=41`
  - `dispatched=1`
  - `totalAccrued=27569`
- `GET /capital/yield-accrual/status` after the probe:
  - `lastCycleElapsedMs=86400000`
  - `lastResult={ scanned=91, eligible=41, dispatched=1, totalAccrued=27569 }`
  - `lastError=null`
- Follow-up `POST /capital/operator/run-vault-report`:
  - `reported=8`
- Follow-up invariant readback:
  - `/capital/invariants?routeId=2 => overallStatus=ok`
  - Tier 3 unsafe gap `0`

Short recurring soak result on 2026-04-24: `PASS`.

- Temporary live config on DO:
  - `YIELD_ACCRUAL_DRIVER_ENABLED=true`
  - `YIELD_ACCRUAL_INTERVAL_MS=60000`
- New smoke scenario:
  - `scripts/hyperlane-smoke/yield-recurring-soak.mjs`
- Live command:

```bash
SBP405_RECURRING_CYCLES=3 \
SBP405_RECURRING_MAX_INTERVAL_MS=180000 \
SBP405_RECURRING_POST_CYCLE_SETTLE_MS=15000 \
node scripts/hyperlane-smoke/run-smoke.mjs \
  --manifest=./scripts/hyperlane-smoke/manifest.do.json \
  --only=yield-recurring-soak
```

- Result: `PASS` in `302625ms`
- Observed cycles:
  - cycle 1: `dispatched=1`, `totalAccrued=78`
  - cycle 2: `dispatched=1`, `totalAccrued=96`
  - cycle 3: `dispatched=1`, `totalAccrued=84`
- After every cycle:
  - `run-vault-report => reported=8`
  - `vault/status.transport=hyperlane`
  - `vault/status.lastError=null`
  - `/capital/invariants?routeId=2 => overallStatus=ok`
  - Tier 3 unsafe gap `0`
- After the soak, DO config was restored to:
  - `YIELD_ACCRUAL_DRIVER_ENABLED=false`
- Current project decision on 2026-04-24:
  - treat SBP-405 as confirmed on the current generation
  - do not make the true 24h autonomous cadence run the next backend blocker
  - keep the autonomous driver disabled until that longer soak is explicitly
    requested

## Historical Testnet Cutover Record — zero-start r2 (2026-04-22)

Final SBP-404 operator cutover completed on the r2 generation. This remains
audit history only; use the r3 record below for live operations.

- `PolicyVaultManager.operator == IcaCommandRouter`
- `VaultFactory.operator == IcaCommandRouter`
- `ICA_DISPATCH_ENABLED=true` on the DO oracle-service
- `REBALANCER_EXECUTOR_ENABLED=true` on the DO oracle-service
- `YIELD_ACCRUAL_DRIVER_ENABLED=false` until SBP-405 E2 is deliberately enabled
  for continuous operation. The manual `yield-event` smoke is green; keep the
  autonomous driver disabled until the soak plan explicitly includes recurring
  yield accrual.

Council Safe executions:

- `PolicyVaultManager.setOperator(icaCommandRouter)`:
  `0x5d15da47171cb26ccb154955f6d67affdaaac776c03a96ae1b23f52abbb33e39`
- `VaultFactory.setOperator(icaCommandRouter)`:
  `0x7f498800c4564b5fb1b2f2c0fe3c59d3eb1cb973785195c86c4a6bd2cfcfe03b`

Verification:

```bash
node scripts/hyperlane/ica-wireup-check.mjs \
  --manifest scripts/hyperlane-smoke/manifest.do.json \
  --prmx-owner 0x18B80F1F9ab9B27492c222c81F6932080f97f2fd \
  --phase cutover \
  --strict
```

Result: `PASS`, including both Base operator checks and all command allowlist,
router enrollment, receiver target, and expected ICA proxy checks.

DO rollout notes:

- First safety action disabled the stale direct executor:
  `/etc/prmx/secrets.env.pre-rebalancer-executor-disable-20260422T075707Z`
- ICA-aware executor dist was deployed with backups:
  `/opt/PRMX/oracle-service/dist.pre-rebalancer-ica-20260422T080030Z` and
  `/opt/PRMX/oracle-service/dist.pre-rebalancer-ica-20260422T080041Z`
- Final enable backup:
  `/etc/prmx/secrets.env.pre-rebalancer-ica-enable-20260422T080125Z`
- Startup evidence:
  `[RebalancerExecutor] Starting ... "dispatchMode":"ica"`
- Stabilization evidence:
  two initial decision cycles returned `noDrift=8`, so no automatic rebalance
  was submitted during enablement.
- Post-cutover ICA command smoke: `PASS`
  - PRMX dispatch tx:
    `0x8af74f5a67f3cd1f59810d801068259acf38eed7e1f92ec09331a935a5e3d620`
  - Base command execution tx:
    `0x7a93f77e72c3ba261f074a77297409a358e922dd91b378c63c8cf803e0ab1b82`
  - MockLendingVault balance delta: `12` atoms for a requested `1` atom command
  - `yieldEventObserved=false` remains unchanged on the deployed mock surface
- Post-cutover API policy/rebalance smoke: `PASS`
  - Command:
    `SMOKE_POLICY_SKIP_SETTLEMENT=1 SMOKE_BASE_RPC_URL=https://base-sepolia.drpc.org node scripts/hyperlane-smoke/run-smoke.mjs --manifest=./scripts/hyperlane-smoke/manifest.do.json --only=policy-lifecycle,rebalance-stress`
  - Policy: `0xdb572d320eb9026257853cac36107406`
  - Vault: `0xBB3A4464BeF0F6149E3D3dC9f0d5E602B97c580d`
  - ICA deploy dispatch: `0x504f7e6d1540fbcdbbfadb7c7ff8fa1e98f95a6279a45220377b4eae37255265`
  - ICA funding dispatch: `0xe65dc869cd744207920e900bbebad2e94d0070f81a886a4d4e6935bb5dbf2b26`
  - ICA autonomous rebalance dispatch:
    `0x9dd5d4e7fbcb61b82bebd290ec94e1a4ed4cc152690deaa29db19111af661990`
  - Rebalance recovery: `99.000001 -> 100.000004` USDC-equivalent vault assets

Follow-up fix discovered by the smoke:

- Set `ICA_DISPATCH_PRMX_TX_GAS=2500000` on DO and made `2500000` the code
  default. PRMX EVM `eth_estimateGas` can over-estimate ICA router dispatches
  above the block gas limit.
- Expanded operator EVM nonce-collision retry matching to include
  `already known` and `lower than the current nonce`.
- DO backups:
  - `/etc/prmx/secrets.env.pre-ica-prmx-tx-gas-20260422T080926Z`
  - `/opt/PRMX/oracle-service/dist.pre-ica-gas-nonce-20260422T081150Z`
- A small testnet reserve top-up restored the strict WarpInvariant pool sample:
  `0x79a476066ade61fa86d35336583e7bd853d3d64fbd5cf473514528efa34ae815`.
  `/capital/invariants?routeId=2` returned `overallStatus=ok`.

Post-cutover stabilization tests:

- `oracle-service/src/capital/__tests__/evm-send.test.ts` pins nonce collision
  matching for the provider phrases observed around the cutover, including
  `already known` and `lower than the current nonce`.
- `oracle-service/src/capital/__tests__/ica-dispatch.test.ts` pins the
  `ICA_DISPATCH_PRMX_TX_GAS=2500000` default and verifies it is propagated into
  the PRMX router transaction.
- `oracle-service/src/rebalancer/executor/__tests__/submit.test.ts` covers the
  Base RPC current-nonce wording for the legacy direct executor path.
- `oracle-service/src/rebalancer/executor/__tests__/executor.test.ts` covers
  ICA-mode circuit-breaker draining so repeated PRMX dispatch failures do not
  retry-storm.

Verification:

```bash
npx --prefix oracle-service tsx --test \
  oracle-service/src/capital/__tests__/evm-send.test.ts \
  oracle-service/src/capital/__tests__/ica-dispatch.test.ts \
  oracle-service/src/rebalancer/executor/__tests__/submit.test.ts \
  oracle-service/src/rebalancer/executor/__tests__/executor.test.ts
npm --prefix oracle-service run build
```

Result: `PASS` (`29` targeted tests) and TypeScript build `PASS`.

Important: after this cutover, direct Base writes to `PolicyVaultManager` /
`VaultFactory` are expected to fail by design. New operation code must either
use the ICA dispatch path or submit PRMX-native/pallet operations that later
produce relayer-visible Hyperlane messages.

## Env Surface

Required for ICA dispatch:

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

Required for yield accrual:

```dotenv
YIELD_ACCRUAL_DRIVER_ENABLED=true
YIELD_ACCRUAL_ROUTE_ID=2
YIELD_ACCRUAL_INTERVAL_MS=86400000
YIELD_ACCRUAL_ANNUAL_BPS=500
```

Required for autonomous rebalancing after cutover:

```dotenv
REBALANCER_EXECUTOR_ENABLED=true
REBALANCER_DECISION_ENABLED=true
REBALANCER_MONITOR_ENABLED=true
```

The executor will prefer ICA whenever `ICA_DISPATCH_ENABLED=true` and the full
ICA config is present. If ICA is explicitly enabled but incomplete, the executor
stays dormant instead of silently falling back to direct Base writes.

## Current Live Testnet Record — zero-start r3 (2026-04-22)

The r2 ICA addresses above are retained as audit history only. The current live
generation uses the following r3 ICA command bus:

- PRMX `InterchainAccountRouter = 0xBa3531D5D3883fa4B4B68299E231A4aC3cEc565F`
- Base `IcaReceiver = 0xcD92d3F728A95BeEee52C7E45Be073269Dcd0166`
- Base `IcaCommandRouter = 0xcE0950B274C23cDB0568A47aBC9D6c963Dd178F6`
- Base expected ICA proxy = `0x932C34B7F9b06c00fC4D44D9Ca83386Bf1a9Df3B`
- PRMX ICA owner scaffold = `0x18B80F1F9ab9B27492c222c81F6932080f97f2fd`

Wire-up evidence:

- PRMX IAR enrollment tx:
  `0xc5a91469323346bf706666b34a14ca7c89fedaf38c82f2a772bc1ff469d595d7`
- `IcaReceiver.setTarget(IcaCommandRouter)` Safe exec:
  `0x1b857c828cba2780330fc9b99f7292794dfdd16441f9c09bb848e343d9cc6b6d`
- `IcaCommandRouter.setCommandPermissions(...)` Safe exec:
  `0x888fa3ebc5cf46e60b39d072af7c65a14c3acc1220398fb0f251b8a3ed6819dd`
- `VaultFactory.setOperator(IcaCommandRouter)` Safe exec:
  `0x334a3666cd51b67dd78a93c30951f25f03a921c121137611ed0c27cbdd863400`
- `PolicyVaultManager.setOperator(IcaCommandRouter)` Safe exec:
  `0xde884e54401a99893834fc81e19f3586c75435d64a52eb2e49b7d026c7f588f3`

Verification:

```bash
node scripts/hyperlane/ica-wireup-check.mjs \
  --manifest scripts/hyperlane-smoke/manifest.do.json \
  --phase cutover \
  --strict
```

Result: `PASS`.

Base IAR's default PRMX enrollment is a stale immutable default on the reused
standard router. This is advisory only for the current route, because PRMX
dispatch uses explicit `callRemoteWithOverrides(...)` router / ISM overrides.
Do not block cutover on that default, and do not attempt to overwrite it.

ICA command smoke result:

- PRMX dispatch tx:
  `0xb26309d0ecdf0095ab390b7f8c1f2cb196bfbaa9477251f680f1c69ce0bb482b`
- Base command execution tx:
  `0xa2ec8ca80bb33e67fe793a1fcee035b66aa0323d9b650b2706c3ecb563c95103`
- Balance delta: `18`
- `yieldEventObserved=false`; keep SBP-405 E2 blocked until we refresh the
  deployed mock-yield event surface or deliberately switch the watcher to
  command/balance evidence.

No-skip API smoke result:

- policy `0x8a2fdec7ba9a225f093ff056491894a3`
- vault `0x46cA902E5906eC711e7CD81D9f9030e5b84A2746`
- rebalance mode `ica`
- rebalance PRMX dispatch:
  `0xb8617f4fa07b58eaefd1ade88adee9143732d3c951bcd103fa223803bedf598e`
- rebalance Base tx:
  `0x4f487befc552d2a2543300f36e21a119e59dff6e03edbcaf26674c0ed00d6071`
