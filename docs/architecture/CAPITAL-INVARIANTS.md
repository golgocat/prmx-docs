# Capital Invariants: PRMX mUSDC / Base EVM USDC Parity

> **What this is**: the hard rules that govern the PRMX↔Base capital seam, plus the soft drift bounds the system tolerates while cross-chain deliveries are in flight.

## TL;DR

Perfect real-time equality is structurally impossible in a cross-chain system. The correct approach is **invariant tiering**:

| Tier | Invariant | Action on breach |
|---|---|---|
| **1. Bridge seam** | Every change to `bridgeMintedTotal` matches a canonical Hyperlane route effect | Investigate; recover only via governance-gated `reconcile_bridge_minted_down` |
| **2. Backing** | `B ≈ EVM_total` (1:1 backing) | Halt mint paths; reconcile down to actual backing |
| **3. Convergence** | `B − EVM_total` bounded and shrinking | `warn` at 10 USDC drift, `critical` at 100 USDC |

## Canonical capital path

```
Inbound:  Base HypERC20Collateral.lock → Hyperlane → PRMX EVM recipient
                                       → 0x0800 precompile → pallet-assets(1).mint
Outbound: pallet exit extrinsic → PRMX EVM dispatcher (validated via 0x0801)
                                → Hyperlane → Base HypERC20Collateral.release
```

`pallet-assets(1)` is the settlement-balance SSoT. Any oracle-attested mint path, direct PRMX EVM synthetic exit path, or non-Hyperlane deposit/exit path is outside the design.

## Symbols

| Symbol | Source | Meaning |
|---|---|---|
| **S** | `api.query.assets.asset(1).supply` | Total mUSDC supply on PRMX (the SSoT) |
| **B** | `warpAccount.bridgeMintedTotal` | Strict bridge-net counter: canonical mints minus `finalize_exit` burns |
| **C** | `IERC20(USDC).balanceOf(HypERC20Collateral)` | Idle USDC in the Hyperlane collateral router |
| **PV** | Σ `PolicyVault.totalAssets()` | USDC across all PolicyVaults |
| **ST** | Settlement / exit transient custody | USDC held for in-flight settlement or exit |
| **EVM_total** | `C + PV + ST` | Total Base-side backing |

> `B` is the narrow seam counter — it tracks only bridge-seam mints/burns, making it the correct operand for "Base collateral backs the bridge."

## Tier 1: Bridge Seam Audit

**Rule**: `warpAccount.bridgeMintedTotal` on PRMX is the bridge-seam counter and must be audited against the canonical Hyperlane route, not against ad-hoc side paths or raw idle-collateral snapshots taken in isolation.

- **Enforcement target**: every canonical code path that changes B changes the canonical route-backing amount by the same amount (and vice-versa).
  - Canonical inbound target: Base `HypERC20Collateral` lock → Hyperlane delivery → PRMX EVM recipient/facade/precompile → `pallet-assets(1)` mint and `bridgeMintedTotal` increment.
  - Canonical outbound target: pallet exit extrinsic records `PendingExits`, PRMX EVM dispatcher validates the payload through `0x0801`, Hyperlane delivers, Base collateral router releases USDC, then `finalize_exit` burns escrowed `pallet-assets(1)` and decrements `bridgeMintedTotal`.
- **Why a raw `B == C` check is insufficient**: once reserve is moved from the active collateral into PolicyVaults, `C` no longer contains the full bridge-backed principal.
- **Detection**: `oracle-service/src/capital/warp-invariant-monitor.ts` reads the canonical bridge-net counter through `settlement-ledger.ts` and compares it to Base backing total (`collateral + PolicyVaults + settlement`) from the vault-reporter cache. Collateral-only drift is still exposed as a breakdown field, but it is not a critical invariant once reserve has been deployed into PolicyVaults.
- **Recovery**: Governance-gated `reconcile_bridge_minted_down(new_total)` (CouncilMajority origin) to realign B down to the actual backed amount. Cannot silently increase B — a mismatch in the other direction indicates unbacked mint, which must be investigated, not papered over.

> **Live transport summary**: yield reports run through Hyperlane (`VAULT_REPORTER_TRANSPORT=hyperlane`) via Base `YieldReporter` → PRMX `YieldReportRecipient` → `0x0802`. Base reserve, rebalance, and strategy operations are executed through Hyperlane ICA; direct Base writes to operational targets are not permitted. The dispatcher transaction may be submitted by oracle-service as a keeper, but authority comes from pallet `PendingExits` via `0x0801`. See [m76 — Yield Report Hyperlane Transport](/docs/hyperlane-migration/m76-yield-report-hyperlane-transport).

## Tier 2: B ~= EVM_total (1:1 Backing Invariant)

**Rule**: PRMX bridge-backed liability should match total EVM-locked USDC within
the configured tolerance. Every bridged mUSDC must be backed, and material
surplus backing is also drift that needs reconciliation.

- **Enforcement target**: a single authorized minting path — Hyperlane Warp Route ending in `pallet-assets(1)` via the PRMX EVM recipient/facade/precompile path. See [m72 — pallet-assets canonical path decision](/docs/hyperlane-migration/m72-pallet-assets-hyperlane-canonical-path-decision).
- **Runtime guard**: `frame_system::Config::BaseCallFilter` blocks native
  `pallet-assets` create/mint/burn/team/status/destroy calls for
  `SETTLEMENT_ASSET_ID = 1`. External users, operators, and governance
  motions cannot issue or retire Assets(1) through the ordinary assets pallet
  extrinsics; the permitted supply-changing paths are the Hyperlane-backed
  runtime/precompile paths.
- **`force_credit_deposit` is permanently disabled** (returns `ForceDepositDisabled`).
- **S remains the `pallet-assets(1)` settlement-balance SSoT**, and fresh chain generations must bootstrap user / LP / DAO balances through Hyperlane rather than genesis. It is exposed for observability; B is the bridge-backed liability used for Base-route safety checks.
- **Transient surplus backing (B `<` EVM_total)** can happen while collateral is
  locked before the matching PRMX bridge liability is minted, or when operational
  backing is added ahead of a Hyperlane bootstrap allocation. It is not as
  dangerous as under-backed liability, but material surplus must be reported as
  unhealthy drift rather than `OK`.
- **Over-backed surplus repair** never increases B to make the numbers match.
  Repair by waiting for the matching Hyperlane delivery/bootstrap evidence,
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
    latency: a few minutes)
  - Exit in transit (PRMX burned, Base collateral not yet released): gap = exit amount, closes on the next Hyperlane delivery in the reverse direction
  - Yield accrual between reporter cycles: gap = unreported yield, closes after the next Hyperlane vault report or pre-settlement freshness report
  - Settlement preparation: gap = prepared liquidity, closes when settlement executes
- **Thresholds** (configured in oracle-service `.env`):
  - `CAPITAL_RECONCILIATION_WARN_THRESHOLD`: default 10 USDC
  - `CAPITAL_RECONCILIATION_CRITICAL_THRESHOLD`: default 100 USDC, used by the reconciler halt path
- **Detection**: Reconciler + vault reporter `checkParity`.

## Yield Reporting — Hyperlane Path

Vault-asset reports from the trusted reporter-pallet path apply in both directions: `new_assets` overwrites the on-chain snapshot regardless of sign. The live periodic route reaches that pallet through Hyperlane `0x0802`. Direct `pallet-policy-v4::update_vault_assets` is restricted so only the trusted yield-report transport can mint upward yield; baseline calibration is established by the funding/admin correction path.

### Pallet behaviour

| Pallet | Extrinsic | Behaviour |
|--------|-----------|-----------|
| `pallet-prmx-policy-vault-reporter` | `report_vault_assets(batch)` | Applies credit and debit deltas, then invokes the runtime `VaultAssetsReportHook`, which calls `pallet-policy-v4::apply_vault_assets_report(..., VaultReportSource::YieldTransport)` to credit/debit policy pool accounting and canonical `pallet-assets(1)` via `CapitalApiV4Adapter`. Emits `VaultAssetsReported { applied }` and per-policy `PolicyVaultAssetsUpdated`. |
| `pallet-prmx-policy-vault-reporter` | `acknowledge_rebalance(policy_id, credit, debit, message_id)` | Applies `assets = assets + credit - debit`. A debit larger than current assets returns `DebitExceedsAssets`. |
| `pallet-policy-v4` | `update_vault_assets(policy_id, total_assets)` | Direct extrinsic restricted to no-op/downward reports; upward deltas return `UpwardReportRequiresYieldOrigin`. |
| `pallet-prmx-user-yield-vault` | `update_vault_assets(new_assets)` | Overwrites `TotalYieldVaultAssets`. Downward updates change the ERC-4626 share price — existing shareholders absorb the delta pro-rata; `PendingWithdrawals` entries are locked at burn-time rate and unaffected. |

### Safety guards that still apply

- **Exit / settlement**: `pallet-policy-v4::finalize_settlement` distributes PRMX pool accounting, not a Base wallet payout; users exit to Base later through the normal exit path.
- **Baseline calibration**: `record_vault_funded` marks `VaultCalibrated` at the guarded funding event, and `correctInitialVaultAssets` does the same for admin repair. The first yield report computes a real delta against that baseline, not an implicit calibration cycle.
- **Upward direct-report protection**: direct `pallet-policy-v4::update_vault_assets` cannot increase `latestVaultAssets`; upward yield/principal recovery observations must arrive through the trusted reporter-pallet transport.
- **Vault-backed settlement reserve return**: the settlement watcher returns the fresh `latestVaultAssets` snapshot from Base `PolicyVault` to reserve for vault-backed policies, then acknowledges only outstanding local liquidity on PRMX. If `latestVaultAssets` is missing for a vault-backed settlement, the watcher fails closed instead of falling back to stale required-liquidity amounts.
- **1:1 backing invariant (Tier 2)**: B and S remain monitored against EVM_total. Real losses reduce backing and may require the DAO backstop before triggered payout; they are not ignored or reclassified as funding.
- **Oracle-service edge**: does not filter debits — true strategy P/L reports reach the pallet. Add a noise-threshold filter (`|new - current| / current < 0.0001`) if a noisy strategy causes thrash.
- **Triggered payout backstop**: if a triggered settlement faces negative strategy P/L after the `latestVaultAssets` guard, the deficit is covered by the DAO backstop path, not by minting unbacked `pallet-assets(1)` or hiding the loss as rebalance capital.

### Capital movements vs yield

Rebalance acknowledgements and initial/manual funding are capital movement
evidence, not yield. They update pending credit/debit, reporter baselines, and
Base backing totals without crediting policy yield.

The vault reporter quarantines zero-baseline funding reports until matching
rebalance/funding evidence exists. A live Base vault balance with a zero PRMX
snapshot must not be booked as yield merely because the first ack has not landed;
once pending rebalance credit/debit exists, the report can be applied normally.
`record_vault_funded` / `correctInitialVaultAssets` are the explicit calibration
points, so the first normal Hyperlane report computes a real delta against that
baseline.

The rebalancer applies the same separation. It classifies candidate vaults as
`uncalibrated_or_unknown`, `already_funded`, or `real_drawdown_candidate`; only
real drawdown candidates can become automatic rebalance-in batches. This
prevents double-top-up while still allowing accepted drawdowns below target to
be repaired.

### Testnet strategy

The Base Sepolia testnet strategy is `MorphoBlueStrategy`, with a loss-capable
`MockLendingVault.simulateLoss(...)` path available for exercising debit flows.
A Base strategy debit lowers both `PolicyVault.totalAssets()` and the PRMX
`policyVaultReporter` snapshot. Borrower automation is testnet-only and
off-book: it creates utilization and P/L test conditions, but it is not a
settlement-balance source, user lending product, or canonical capital path.

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

