# PRMX V4 / V5 Product Split

> Status: Decision recorded 2026-04-26
> Scope: Product/repository boundary and V5 regulated SAC architecture baseline.

## Decision

PRMX will split into two product lines and repositories:

| Product line | Repository | Positioning | Architecture stance |
|---|---|---|---|
| PRMX V4 | `PRMX-v4-defi` | Fully Decentralized DeFi Version | Continue the current working V4 architecture: user-held balances, policy-level LP mechanics, orderbook-oriented DeFi UX, Hyperlane-backed settlement. |
| PRMX V5 | `PRMX-v5-regulated` | Regulated / Bermuda SAC Version | New regulated architecture: SAC is the risk-taking entity, funds are held in SAC-scoped accounts, users hold positions/claims, and exits are same-SAC-only. |

Workspace convention:

| Product line | Repository |
|---|---|
| PRMX V4 | `PRMX4` (this codebase) |
| PRMX V5 | `PRMX5` (separate repository) |

V5 is not a small feature branch on V4. It changes the ownership model for capital, the transfer model for user funds, the reserve model on Base, and the audit story. It should be developed from a V4 completion baseline in a separate GitHub repository.

## V4 Scope

V4 remains the DeFi version:

- Current Rainguard / Weather Gate / Climate Parametrics product catalog.
- User mUSDC balances remain normal user balances.
- Policy-level LP tokens and LP orderbook behavior remain part of the DeFi direction.
- Hyperlane Route 2 / Base Sepolia remains the active capital route.
- V4 should be completed and stabilized as `PRMX-v4-defi`, not partially converted into a regulated SAC product.

## V5 Regulated SAC Baseline

V5 uses the following final SAC design:

`Product-Year SAC Cell + per-SAC Hyperlane Reserve + per-SAC PRMX Asset(1) accounts + user position/claim ledger + policy-level LP token owned by SAC + same-SAC-only Exit`

### SAC Cell Unit

The conservative default cell key is:

`product = location x event_type`, plus `year_cohort`

Example:

`Rainguard 24H / Tokyo / Rain / 2026`

Bermuda law does not appear to require a new SAC every year by itself; the requirement is that assets, liabilities, rights, and obligations are clearly linked to the relevant segregated account. V5 still keeps year cohort as a first-class dimension so the operating model can choose conservative yearly isolation or a lighter product-level model after counsel review.

### Base-Side Reserve

Each SAC Cell gets its own Base-side Hyperlane reserve surface:

- SAC-specific Hyperlane collateral/reserve.
- SAC-specific deposit route.
- SAC-specific exit route.

A user who enters SAC A must exit through SAC A's reserve. SAC A exits must not release funds from SAC B.

### PRMX-Side Asset(1) Accounts

`pallet-assets(1)` remains the single settlement asset and bookkeeping source of truth for mUSDC. V5 does not create a separate settlement token per SAC.

Instead, each SAC has deterministic PRMX accounts under Asset(1), such as:

- LP capital account.
- Premium account.
- Loss reserve account.
- Claim payout account.
- Yield account.
- Pending protocol fee account.
- LP return account.

SAC-bound funds are not credited as transferable user mUSDC.

### User Positions / Claims

When User X deposits into SAC A, PRMX records:

- Asset(1) mUSDC in SAC A's dedicated account.
- A position/claim record for User X in SAC A.

The user does not receive freely transferable mUSDC for SAC-bound capital. A position record should include at least:

- `sac_id`
- `user`
- `role` (LP, policyholder, beneficiary, etc.)
- `source_deposit_id`
- `reserve_id`
- `contribution_amount`
- `position_units`
- `locked_amount`
- `withdrawable_amount`
- `status`

User actions are signed by the user's normal PRMX account, but the SAC pallet moves funds only after validating the user's position and SAC rules.

### Transfer Rules

SAC-bound funds are not freely transferable.

Allowed:

- SAC-internal premium handling.
- SAC-internal claim payout.
- SAC-internal LP return.
- Exit from SAC A through SAC A's reserve.
- Free transfer only after funds have exited the SAC and become ordinary user funds.

Denied by default:

- SAC A funds sent directly to a SAC B user.
- SAC A funds used for SAC B claims.
- SAC A yield or reserves moved to SAC B.
- SAC-to-SAC transfers through ordinary user transfer flows.

Exceptional SAC-to-SAC transfers, if ever needed for correction, rollover, or restructuring, must be a separate governance/compliance-controlled action and default off.

### LP Token Model

LP tokens remain policy-level, preserving the current PRMX V4 mental model.

The V5 change is the initial owner:

- A policy is underwritten by a SAC.
- When the policy is formed, 100% of that policy's LP tokens are issued to the SAC.
- The LP token is non-transferable by default.
- External LPs do not receive LP tokens directly at policy formation.

Future regulated distribution is optional:

- A SAC or policy-level setting can later enable external sale.
- The first allowed flow should be primary sale from SAC to KYC/eligible external LPs.
- External LP-to-LP secondary transfer remains a separate later decision.

### Revenue Waterfall

SAC-linked revenue must settle inside the SAC before any protocol-level distribution:

`SAC accounting close -> protocol fee calculation -> transfer to general revenue -> staker distribution`

Gas fees can remain global. Premiums, yield, underwriting PnL, payout residuals, and LP returns must first be attributed to the relevant SAC.

### Governance

Roles:

- Global Council: create/close SACs, emergency controls, global settings.
- SAC Governor / Cell Operator: day-to-day SAC operations, underwriting enablement, SAC-level fund movement approval, reporting checks.
- Underwriter: risk decisioning within the SAC mandate.
- Compliance/Admin: KYC, sanctions, eligible-investor and account-owner records.

Underwriter and SAC Governor should be distinct roles.

### Pilot

The first V5 pilot should be only:

`Rainguard 24H / Tokyo / Rain / 2026`

The pilot must prove:

- SAC creation.
- Dedicated Base reserve.
- Dedicated PRMX Asset(1) accounts.
- LP deposit into SAC.
- Policy purchase via user-signed instruction.
- 24h oracle settlement.
- Claim payout or maturity close.
- Same-SAC-only exit.
- SAC-level audit report.

## Repository Actions

1. Complete and label the current repository state as `PRMX-v4-defi`.
2. Clone from the V4 completion baseline into the GitHub repository `PRMX-v5-regulated`.
3. Keep V4 work in the `PRMX4` repository and V5 work in the `PRMX5` repository as separate workspaces.
4. Move Bermuda SAC implementation work to V5 only.
5. Keep V4 roadmap and V5 roadmap separate in Linear.

## Linear Mapping

- SBP-430 should track V5 SAC architecture, including per-SAC reserve, Asset(1) SAC accounts, position/claim records, and same-SAC-only exit.
- SBP-431 should track V5 policy-LP transferability controls, default off, with future SAC-to-external-LP primary sale gated by compliance settings.
- A separate repository-split issue should track GitHub repository creation and migration hygiene.
- Linear project split should be explicit:
  - `PRMX-v4-defi`
  - `PRMX-v5-regulated`
