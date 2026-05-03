# PRMX V4 — Engineer Reading Guide

> **Audience**: engineers learning the PRMX V4 architecture and design.
> **Status**: curated reading set (updated 2026-05-03).
> **Generation**: post-Hyperlane-cutover (2026-04-19), runtime spec ≥ 510, Hyperbridge fully retired.

PRMX V4 is a Substrate-based parametric weather insurance system. It mints a 1:1-backed settlement asset (`mUSDC`) on PRMX from USDC locked on Base Sepolia via Hyperlane, prices weather products from a 31-year ERA5 catalog, and settles policies on-chain when measured weather events trigger their thresholds.

This guide orders the documentation so that a new engineer can build a full mental model of the system without wading through historical migration notes or operator runbooks. Pre-cutover designs and superseded plans live in the source repo's `docs/archive/` directory and are only referenced for audit-trail purposes.

---

## 1 — Start here (system overview)

Read in this order to get end-to-end coverage:

| # | Document | What you will learn |
|---|----------|--------------------|
| 1 | [V4 Architecture](/docs/architecture/V4-ARCHITECTURE) | The whole system: Substrate runtime pallets, Frontier EVM precompiles, off-chain oracle service, Open-Meteo OCW, Vercel frontend, MCP server, and where each piece lives. |
| 2 | [V4 Oracle Architecture](/docs/architecture/V4-ORACLE-ARCHITECTURE) | Oracle responsibilities: weather observation ingest, snapshot finalization, and the Route 2 capital / vault / rebalancer / yield-reporter workers run by the oracle service. |
| 3 | [Capital Invariants](/docs/architecture/CAPITAL-INVARIANTS) | The two non-negotiable rules: 1:1 USDC↔mUSDC backing, and `pallet-assets(1)` as the single source of truth for all PRMX balances. Reading this before touching capital code prevents the most damaging class of bug. |
| 4 | [Parametric Insurance Rulebook](/docs/architecture/PARAMETRIC-INSURANCE-RULEBOOK) | How parametric insurance is modeled in V4: peril taxonomy, threshold semantics, settlement decision rules, dispute windows. |
| 5 | [PRMX V4 / V5 Product Split](/docs/architecture/PRMX-V4-V5-PRODUCT-SPLIT) | The product-line split: V4 is the DeFi version (this repo); V5 is a separate regulated Bermuda SAC product line. Read so you do not retrofit V5 concerns into V4 code. |

After section 1 you should be able to answer: *what does PRMX do, who keeps balances honest, and how does it decide whether a policy paid out?*

---

## 2 — Cross-chain transport (post-cutover designs)

Hyperlane Warp Route + Interchain Accounts (ICA) is the **only** transport between PRMX and Base. The Hyperbridge ISMP path was retired on 2026-04-19. These five design docs describe the live path:

| # | Document | What you will learn |
|---|----------|--------------------|
| 1 | [m72 — pallet-assets canonical path decision](/docs/hyperlane-migration/m72-pallet-assets-hyperlane-canonical-path-decision) | Why `pallet-assets(1)` (not an EVM-side ledger) is the SSoT, and how the canonical inbound / outbound paths flow through the Mailbox, ISM, and PRMX `0x0800` precompile. |
| 2 | [m73 — Exit Dispatcher design](/docs/hyperlane-migration/m73-exit-dispatcher-design) | Outbound exit semantics: `request_exit` extrinsic → pallet escrow → `EvmDispatchHook` → `0x0801` pending-exit precompile validation → Hyperlane release on Base. |
| 3 | [m75 — ICA Yield Command Bus](/docs/hyperlane-migration/m75-ica-yield-command-bus) | The ICA command bus: PRMX `InterchainAccountRouter` → Base ICA proxy → `IcaCommandRouter` → allowlisted Base targets (`PolicyVaultManager`, `VaultFactory`, etc.). This is how PRMX drives Base operations without any operator EOA holding direct Base authority. |
| 4 | [m76 — Yield Report Hyperlane Transport](/docs/hyperlane-migration/m76-yield-report-hyperlane-transport) | How vault yield is reported back: Base `YieldReporter` → Hyperlane → PRMX `YieldReportRecipient` → `0x0802` precompile → `pallet-prmx-policy-vault-reporter` → `policy-v4` → `pallet-assets(1)`. |
| 5 | [m78 — PRMX EVM User Actions design](/docs/hyperlane-migration/m78-prmx-evm-user-actions-design) | The PRMX EVM write surface (`0x0808` UserActions) that lets a linked EVM wallet submit user-owned PRMX actions through the same precompile pattern. |

After section 2 you should be able to answer: *how does USDC become mUSDC and back again, and how does PRMX command Base contracts without holding their keys?*

---

## 3 — Product layer

| Document | What you will learn |
|----------|--------------------|
| [V4 App Design](/docs/product/V4-APP-DESIGN) | Application-level UX and route design: how a user discovers a product, gets a quote, submits a policy, and tracks settlement. |
| [V4 Product Catalog](/docs/product/V4-PRODUCT-CATALOG) | The 14 active products, their thresholds, units, and which event types they map to on-chain. |
| [Product Lineup](/docs/product/PRODUCT-LINEUP) | Product taxonomy: Rain Guard / Weather Gate / Climate Parametrics. |
| [Rainguard Product Spec](/docs/product/RAINGUARD-PRODUCT-SPEC) | Detailed Rain Guard product spec (the lead product). |
| [Open-Meteo Event Expansion](/docs/product/OPEN-METEO-EVENT-EXPANSION) | Roadmap for adding new event types from Open-Meteo data. |

---

## 4 — Reference (deeper or specialized)

Read these when working on the relevant area, not as part of onboarding:

| Document | When to read |
|----------|-------------|
| [DeFi Operational Patterns](/docs/architecture/DEFI-OPERATIONAL-PATTERNS) | When designing or modifying Hyperlane Warp Route operations (rebalancer, MovableCollateralRouter patterns, optional XERC20 considerations). |
| [Test Principles](/docs/guidelines/TEST-PRINCIPLE) | Before adding tests; the principles for what to assert and what to mock. |
| [UI Design Principles](/docs/guidelines/UI-DESIGN-PRINCIPLES) | When building or editing frontend components (terminal-dark visual system). |
| [Restart Guide](/docs/operations/RESTART-GUIDE) | When you need to restart the local or DO-hosted environment. |
| [Oracle Workers](/docs/operations/ORACLE-WORKERS) | When debugging an oracle-service worker (capital, rebalancer, vault-reporter, Morpho). |

---

## 5 — Out of scope for new readers

The following directories exist in the source repository for traceability but are **not** part of the engineer reading set published here:

- `docs/archive/` — pre-cutover Hyperbridge plans, V1/V2/V3 oracle history, superseded per-entity vault designs, and the original Hyperlane migration plan as audit trail. Use only when researching *why* the current design exists.
- `docs/hyperlane-migration/m41`–`m77` operator runbooks — cutover, queue cleanup, validator ops, kill-switch drill, zero-start. Useful for SREs, not for understanding the architecture.
- `docs/plans/` — frozen or draft future plans (Basket Token, Parametric Insurance Platform, AI Agent Platform). Not load-bearing for V4.
- `mcp-server/TOOLS.md` — MCP tool reference for LLM agents integrating against PRMX. Lives outside the docs tree.
- `tasks/` — Codex handoffs, lessons, and ephemeral session scratch.

---

## Quick reference — what is the source of truth for…?

| Question | Answer |
|----------|--------|
| User mUSDC balance | `pallet-assets(1)` on PRMX |
| Bridge-backed liability counter | `warpAccount.bridgeMintedTotal` on PRMX |
| Active spec_version | `state_getRuntimeVersion` on the live chain (testnet currently ≥ 510) |
| Active pricing catalog | `obs-openmeteo-14prod-2026-03-12` (Cloud Run API) |
| Settlement asset | `mUSDC` (MockUSDC ↔ Base via Hyperlane Warp Route) |
| Policy state machine | `pallet-prmx-policy-v4` |
| Cross-chain transport | Hyperlane Mailbox + ICA (sole transport) |
| Governance | Council (3-of-5 multisig); sudo permanently removed in spec 441 |

For invariants and lifecycle rules, see [Capital Invariants](/docs/architecture/CAPITAL-INVARIANTS) and the project root `CLAUDE.md`.
