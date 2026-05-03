# Capital Invariants: PRMX mUSDC / Route 2 EVM USDC Parity

> **Values that must converge**: PRMX mUSDC supply (S, `pallet-assets(1)` SSoT),
> bridge-net counter (B), Base Route 2 collateral (C), PolicyVault assets (PV),
> EVM vault totalManagedAssets (V).
>
> Perfect real-time equality is structurally impossible in a cross-chain system — deposit relay latency,
> settlement-to-exit gaps, and yield reporter polling intervals create transient divergence.
> The correct approach is **invariant tiering**: hard rules that must never break, and soft rules
> with bounded convergence windows.

> **Architecture decision (2026-04-21)**: the canonical PRMX capital path is now explicitly fixed as
> Route 2 (Base Sepolia) only, with `pallet-assets(1)` as the settlement-balance
> source of truth and Hyperlane traversal in both directions.
> Inbound target: Base collateral lock → Hyperlane → PRMX EVM recipient/facade/precompile →
> `pallet-assets::mint`. Outbound target: pallet exit extrinsic → PRMX EVM dispatch surface
> validated against `warpAccount.pendingExits` through the `0x0801` precompile →
> Hyperlane → Base collateral release. Any oracle-attested mint path, direct PRMX EVM synthetic
> exit path, or non-Hyperlane deposit/exit path that bypasses this model is outside
> the final design.

## Definitions

| Symbol | Source | Description |
|--------|--------|-------------|
| **S** | `api.query.assets.asset(1).supply` | Total mUSDC circulating on PRMX (`pallet-assets(1)` settlement-balance SSoT). Fresh genesis must not pre-allocate balances; testnet initial balances are created by Base MockUSDC mint + Hyperlane deposit. |
| **B** | `api.query.warpAccount.bridgeMintedTotal` | Strict Route 2 bridge-net counter: canonical Hyperlane bridge-in mints minus `finalize_exit` burns. Genesis allocations should be zero by design. |
| **C** | `IERC20(USDC).balanceOf(HypERC20Collateral)` on Base | Idle USDC still sitting in the active Route 2 Hyperlane collateral router. Useful as a partial seam signal, but not the full route-backing picture once reserve has been deployed into PolicyVaults. |
| **ST** | Settlement / exit bridge transient custody, when present | Route-local USDC held outside the collateral router and outside active PolicyVaults for settlement or exit processing. Normally small or zero in steady state. |
| **PV** | Sum of all `PolicyVault.totalAssets()` | USDC held in individual policy vaults |
| **EVM_total** | C + PV + ST | Total Base backing across the active Route 2 contracts |

> **Historical note (SBP-403 Phase C5)**: prior to C5, there was a third on-chain
> value `M = warpAccount.totalMinted` that tracked every mint + burn of the
> synthetic mUSDC. Older testnet generations also mixed direct genesis/dev
> allocations into `S`, which made raw supply unsafe to treat as Base-backed.
> Phase C5 retired `TotalMinted` and `SyntheticBalance` storages and made
> `pallet-assets(1)` the single bookkeeping authority. Fresh generations must
> create initial balances through Hyperlane, so `S` and `B` start aligned for
> bridge-originated mints. `BridgeMintedTotal` was introduced in C4.1 as a
> narrower counter: it tracks only bridge-seam mints/burns, making it the
> correct operand for the "Base collateral backs the bridge" invariant.

## Tier 1: Bridge Seam Audit

**Rule**: `warpAccount.bridgeMintedTotal` on PRMX is the bridge-seam counter and must be audited against the canonical Hyperlane route, not against ad-hoc side paths or raw idle-collateral snapshots taken in isolation.

- **Enforcement target**: every canonical code path that changes B changes the canonical route-backing amount by the same amount (and vice-versa).
  - Canonical inbound target: Base `HypERC20Collateral` lock → Hyperlane delivery → PRMX EVM recipient/facade/precompile → `pallet-assets(1)` mint and `bridgeMintedTotal` increment.
  - Canonical outbound target: pallet exit extrinsic records `PendingExits`, PRMX EVM dispatcher validates the payload through `0x0801`, Hyperlane delivers, Base collateral router releases USDC, then `finalize_exit` burns escrowed `pallet-assets(1)` and decrements `bridgeMintedTotal`.
- **Why the old raw `B == C` check breaks**: once reserve is moved from the active collateral into PolicyVaults, `C` no longer contains the full bridge-backed principal. Direct PRMX EVM synthetic exits made this even noisier by mutating Base collateral without touching pallet exit state.
- **Detection**: `oracle-service/src/capital/warp-invariant-monitor.ts` reads the canonical bridge-net counter through `settlement-ledger.ts` and compares it to Base backing total (`collateral + PolicyVaults + settlement`) from the vault-reporter cache. Collateral-only drift is still exposed as a breakdown field, but it is not a critical invariant once reserve has been deployed into PolicyVaults.
- **Recovery**: Governance-gated `reconcile_bridge_minted_down(new_total)` (CouncilMajority origin) to realign B down to the actual backed amount. Cannot silently increase B — a mismatch in the other direction indicates unbacked mint, which must be investigated, not papered over.

> **Current-live status**: as of 2026-05-01, SBP-403 C3/C4, SBP-404 ICA cutover,
> SBP-405 E2, SBP-421, SBP-453/454/455, spec 510 baseline calibration, and
> the SBP-420 Morpho Blue testnet strategy are live or verified on the
> zero-start r3 Route 2 generation.
> Oracle-service may still submit the PRMX dispatcher transaction as a keeper,
> but authority comes from pallet `PendingExits` via `0x0801`. Yield reports now
> run through Hyperlane in live DO operation (`VAULT_REPORTER_TRANSPORT=hyperlane`)
> via Base `YieldReporter` -> PRMX `YieldReportRecipient` -> `0x0802`; upward
> direct `update_vault_assets` reports are rejected after SBP-453. `both`
> remains the rollback / next-generation soak mode after future redeploys. Base
> reserve, rebalance, and strategy operations are executed through Hyperlane ICA;
> direct Base writes to operational targets are cutover violations. See
> [m76-yield-report-hyperlane-transport.md](/Users/satorubito/Codes/PRMX/docs/hyperlane-migration/m76-yield-report-hyperlane-transport.md).

## Tier 2: B ~= EVM_total (1:1 Backing Invariant)

**Rule**: PRMX bridge-backed liability should match total EVM-locked USDC within
the configured tolerance. Every bridged mUSDC must be backed, and material
surplus backing is also drift that needs reconciliation.

- **Enforcement target**: a single authorized minting path — Hyperlane Warp Route ending in `pallet-assets(1)` via the PRMX EVM recipient/facade/precompile path. See [m72-pallet-assets-hyperlane-canonical-path-decision.md](/Users/satorubito/Codes/PRMX/docs/hyperlane-migration/m72-pallet-assets-hyperlane-canonical-path-decision.md).
- **Runtime guard**: `frame_system::Config::BaseCallFilter` blocks native
  `pallet-assets` create/mint/burn/team/status/destroy calls for
  `SETTLEMENT_ASSET_ID = 1`. External users, operators, and governance
  motions cannot issue or retire Assets(1) through the ordinary assets pallet
  extrinsics; the permitted supply-changing paths are the Hyperlane-backed
  runtime/precompile paths.
- **`force_credit_deposit` is permanently disabled** (spec 421, returns `ForceDepositDisabled`).
- **S remains the `pallet-assets(1)` settlement-balance SSoT**, and fresh chain generations must bootstrap user / LP / DAO balances through Hyperlane rather than genesis. It is exposed for observability; B is the bridge-backed liability used for Base-route safety checks.
- **Transient surplus backing (B `<` EVM_total)** can happen while collateral is
  locked before the matching PRMX bridge liability is minted, or when operational
  backing is added ahead of a Hyperlane bootstrap allocation. It is not as
  dangerous as under-backed liability, but material surplus must be reported as
  unhealthy drift rather than `OK`.
- **Over-backed surplus repair** never increases B to make the numbers match.
  Repair by waiting for the matching Route 2 delivery/bootstrap evidence,
  classifying the movement as rebalance/funding capital, or returning/reallocating
  surplus through the reserve path.
- **Under-backed liability (B > EVM_total)** is critical when confirmed by a
  complete Base sample: it means unbacked bridge liability exists.
- **Detection**: Reconciler runs periodically, comparing B against EVM_total (see Known Gaps below).
- **Recovery**: Asset burn plus governance-gated `reconcile_bridge_minted_down(new_total)` when the bridge-net counter must be moved down to match externally verified backing. There is no upward `totalMinted` repair path.

## Tier 3: B - EVM_total Bounded & Converging (Soft Invariant)

**Rule**: The gap between B and EVM_total should be small and should close within
a bounded window. Positive gaps are under-backed. Negative gaps are surplus
backing drift. Material drift in either direction is `warn`; confirmed
under-backed drift beyond the critical threshold is `critical`.

- **Expected transient gaps**:
  - Deposit in transit (Base collateral locked, PRMX not yet credited): temporary
    surplus backing, closes when Hyperlane Warp Route delivers (typical testnet
    latency: a few minutes; on 2026-04-19 nonce 1 PRMX->Base and 91814/91815
    Base->PRMX all delivered without manual intervention)
  - Exit in transit (PRMX burned, Base collateral not yet released): gap = exit amount, closes on the next Hyperlane delivery in the reverse direction
  - Yield accrual between reporter cycles: gap = unreported yield, closes after the next Hyperlane vault report or pre-settlement freshness report
  - Settlement preparation: gap = prepared liquidity, closes when settlement executes
- **Thresholds** (configured in oracle-service `.env`):
  - `CAPITAL_RECONCILIATION_WARN_THRESHOLD`: default 10 USDC
  - `CAPITAL_RECONCILIATION_CRITICAL_THRESHOLD`: default 100 USDC, used by the reconciler halt path
- **Detection**: Reconciler + vault reporter `checkParity`.

## Yield Reporting — Hyperlane Current Path (spec 510 / SBP-453)

Vault-asset reports from the trusted reporter-pallet path apply in both directions: `new_assets` overwrites the on-chain snapshot regardless of sign. The live periodic route reaches that pallet through Hyperlane `0x0802`; rollback/operator direct batches can still enter the same reporter pallet. The prior credit-only guard (SBP-178) was removed in spec 481 to unblock real-loss strategies and the triggered-settlement yield finalize hook (SBP-401). Spec 510 moved baseline calibration to funding/admin correction, and SBP-453 restricts legacy policy-v4 direct-extrinsic reports so only the trusted yield-report transport can mint upward yield.

### Pallet behaviour

| Pallet | Extrinsic | Behaviour |
|--------|-----------|-----------|
| `pallet-prmx-policy-vault-reporter` | `report_vault_assets(batch)` | Applies credit and debit deltas, then invokes the runtime `VaultAssetsReportHook`. On current spec `510`, this calls `pallet-policy-v4::apply_vault_assets_report(..., VaultReportSource::YieldTransport)`, which credits/debits policy pool accounting and canonical `pallet-assets(1)` via `CapitalApiV4Adapter`. Emits `VaultAssetsReported { applied }` and per-policy `PolicyVaultAssetsUpdated`. |
| `pallet-prmx-policy-vault-reporter` | `acknowledge_rebalance(policy_id, credit, debit, message_id)` | Applies `assets = assets + credit - debit`. A debit larger than current assets returns `DebitExceedsAssets`. |
| `pallet-policy-v4` | `update_vault_assets(policy_id, total_assets)` | Legacy direct extrinsic. After SBP-453 it can apply no-op/downward reports only; upward deltas return `UpwardReportRequiresYieldOrigin`. |
| `pallet-prmx-user-yield-vault` | `update_vault_assets(new_assets)` | Overwrites `TotalYieldVaultAssets`. Downward updates change the ERC-4626 share price — existing shareholders absorb the delta pro-rata; `PendingWithdrawals` entries are locked at burn-time rate and unaffected. |

### Safety guards that still apply

- **Exit / settlement**: `pallet-policy-v4::finalize_settlement` distributes PRMX pool accounting, not a Base wallet payout; users exit to Base later through the normal Route 2 exit path.
- **Baseline calibration (spec 510)**: `record_vault_funded` marks `VaultCalibrated` at the guarded funding event, and `correctInitialVaultAssets` does the same for admin repair. The first yield report is no longer a calibration source.
- **Upward direct-report protection (SBP-453)**: legacy policy-v4 direct `update_vault_assets` cannot increase `latestVaultAssets`; upward yield/principal recovery observations must arrive through the trusted reporter-pallet transport.
- **Vault-backed settlement reserve return (SBP-454)**: the settlement watcher returns the fresh `latestVaultAssets` snapshot from Base `PolicyVault` to reserve for vault-backed policies, then acknowledges only outstanding local liquidity on PRMX. If `latestVaultAssets` is missing for a vault-backed settlement, the watcher fails closed instead of falling back to stale required-liquidity amounts.
- **1:1 backing invariant (Tier 2)**: B and S remain monitored against EVM_total. Real losses reduce backing and may require the DAO backstop before triggered payout; they are not ignored or reclassified as funding.
- **Oracle-service edge**: no longer filters debits — true strategy P/L reports reach the pallet. Add a noise-threshold filter (`|new - current| / current < 0.0001`) if a noisy strategy causes thrash.
- **Triggered payout backstop**: if a triggered settlement faces negative strategy P/L after the `latestVaultAssets` guard, the deficit is covered by the DAO backstop path, not by minting unbacked `pallet-assets(1)` or hiding the loss as rebalance capital.

### Capital movements vs yield (SBP-436 / SBP-437)

Rebalance acknowledgements and initial/manual funding are capital movement
evidence, not yield. They update pending credit/debit, reporter baselines, and
Route 2 backing totals without crediting policy yield.

The vault reporter quarantines zero-baseline funding reports until matching
rebalance/funding evidence exists. A live Base vault balance with a zero PRMX
snapshot must not be booked as yield merely because the first ack has not landed;
once pending rebalance credit/debit exists, the report can be applied normally.
Spec 510 additionally makes `record_vault_funded` / `correctInitialVaultAssets`
the explicit calibration points, so the first normal Hyperlane report computes a
real delta against that baseline.

The rebalancer follows the same separation after SBP-455. It classifies
candidate vaults as `uncalibrated_or_unknown`, `already_funded`, or
`real_drawdown_candidate`; only real drawdown candidates can become automatic
rebalance-in batches. This prevents the historical double-top-up pattern while
still allowing accepted drawdowns below target to be repaired.

### Testnet strategy status

SBP-393 added a loss-capable `MockLendingVault.simulateLoss(...)` path and an
opt-in `loss-event` smoke scenario. The live DO/Base Sepolia run on
2026-04-22 verified that a Base strategy debit lowers both
`PolicyVault.totalAssets()` and the PRMX `policyVaultReporter` snapshot.
SBP-420 then deployed and verified the Morpho Blue testnet strategy on Route 2;
it is no longer future-only. Borrower automation remains testnet-only and
off-book: it creates utilization and P/L test conditions, but it is not a
settlement-balance source, user lending product, or canonical capital path.

## Known Gaps

### 1. ~~System Yield Dead Code (recordDeposit / recordExit)~~ — FIXED (SBP-227)

`watcher.ts` now calls `recordDeposit()` at deposit attestation/dispatch success and `recordExit()` at exit finalization. The settlement yield formula correctly isolates capital movements from yield.

### 2. ~~Reconciler Scope~~ — FIXED (SBP-228)

The invariant checker now uses the vault-reporter / warp-invariant backing
sample (`collateral + PolicyVaults + settlement`) instead of a collateral-only
or legacy `SettlementVault`-only snapshot. Sample quality is surfaced separately
from the backing numbers so discovery gaps can be diagnosed without silently
classifying principal movement as yield.

## Invariant Checker API

Unified 3-tier health report available on-demand:

```
GET /capital/invariants?routeId=2
```

Returns structured `InvariantReport` with per-tier status, EVM breakdown, and Tier 3 convergence direction (rolling 10-sample window). Overall status is the worst of all three tiers.

**Source**: `oracle-service/src/capital/invariant-checker.ts`

| Tier | Critical When | Warn When |
|------|--------------|-----------|
| 1 (bridge seam) | route-backing delta exceeds critical threshold | route-backing delta exceeds warn threshold |
| 2 (B~=EVM_total) | under-backed delta exceeds configured critical threshold | under-backed or over-backed delta exceeds configured warn threshold |
| 3 (convergence) | unsafe gap > configured warn threshold AND diverging | unsafe gap > configured warn threshold OR positive unsafe gap diverging |

## Historical Context

- **Spec 421**: `force_credit_deposit` permanently disabled after 1.2M mUSDC unbacked minting incident
- **2026-04-13**: Burned 838,365 excess mUSDC from PRMX to restore 1:1 parity (SBP-226)
- **Route 2 (Base Sepolia)**: Hyperlane-only sole active route. Routes 0 (Sepolia) and 1 (OP Sepolia) deprecated.
- **SBP-420**: Morpho Blue testnet strategy deployed/verified; borrower automation is testnet-only/off-book.
- **SBP-436 / SBP-437**: rebalance and funding capital are quarantined from yield until matching evidence exists.
- **2026-05-01 0x6858 repair**: historical Base Sepolia testnet evidence from the pre-spec-510 calibration/overfunding bug. It should not be described as normal product behavior.
