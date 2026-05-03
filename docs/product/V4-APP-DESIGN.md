# PRMX V4 App Design (Rainguard UI)

> **Scope**: UX/UI for V4 policy purchase, request lifecycle, and policy monitoring.

## Goals

| Goal | What it means |
|---|---|
| **Fast purchase** | Wallet → quote → submit in ~60 seconds |
| **Clarity** | User understands payout, trigger, threshold, coverage window, and settlement-finalizes-after-liquidity |
| **Verifiability** | Every lifecycle transition is backed by on-chain state + timeline events |
| **Operator-friendly** | Chain / oracle / DAO health visible without direct DB access |

## Related docs

- [Product Lineup](/docs/product/PRODUCT-LINEUP) — taxonomy
- [Product Catalog](/docs/product/V4-PRODUCT-CATALOG) — 14 active pricing products
- [V4 Architecture](/docs/architecture/V4-ARCHITECTURE) — system architecture
- [Parametric Insurance Rulebook](/docs/architecture/PARAMETRIC-INSURANCE-RULEBOOK) — insurance rules
- [UI Design Principles](/docs/guidelines/UI-DESIGN-PRINCIPLES) — visual system

---

## Product scope (V4)

### Product lines (portfolio)

| Product line | Purpose | Current scope |
|---|---|---|
| **PRMX Rain Guard** | Core rainfall protection | 3 products |
| **PRMX Weather Gate** | General weather-trigger protection | 4 products |
| **PRMX Climate Parametrics** | Specialty and expansion parametrics | 7 products |

### Purchasable products on `/climate-parametrics` (Protection Terminal, 14 products)

| UI label | Product line | Event type | Peril ID | Duration | Threshold buckets |
|----------|--------------|------------|----------|----------|-------------------|
| Storm Protection | **PRMX Rain Guard** | `Precip12hMaxGte` | `rainfall_7d_12h_accum_binary` | Fixed 7 days | 20, 30, 40, 50, 75, 100 mm |
| Daily Total | **PRMX Rain Guard** | `PrecipSumGte` | `rainfall_7d_24h_total_binary` | Fixed 24h | 20, 30, 40, 50, 75, 100 mm |
| Hourly Intensity | **PRMX Rain Guard** | `Precip1hGte` | `rainfall_7d_1h_max_binary` | Fixed 7 days | 5, 10, 15, 20, 30 mm/h |
| Heat Protection | **PRMX Weather Gate** | `TempMaxGte` | `temp_max_7d_binary` | Fixed 7 days | 35, 38, 40, 42, 45 °C |
| Frost Protection | **PRMX Weather Gate** | `TempMinLte` | `temp_min_7d_binary` | Fixed 7 days | 0, -3, -5, -10 °C |
| Wind Damage | **PRMX Weather Gate** | `WindGustMaxGte` | `wind_gust_max_7d_binary` | Fixed 7 days | 15, 20, 25, 30 m/s |
| Snow & Ice | **PRMX Weather Gate** | `PrecipTypeOccurred` | `precip_type_7d_binary` | Fixed 7 days | 2, 4, 6 (mask) |
| Heat Index | **PRMX Climate Parametrics** | `HeatIndexMaxGte` | `heat_index_max_7d_binary` | Fixed 7 days | 35, 38, 40, 42, 45 °C |
| Snow Accumulation | **PRMX Climate Parametrics** | `SnowDepthMaxGte` | `snow_depth_max_7d_binary` | Fixed 7 days | 50, 100, 200, 500 mm |
| Pressure Drop | **PRMX Climate Parametrics** | `PressureDropMaxGte` | `pressure_drop_max_7d_binary` | Fixed 7 days | 5, 10, 15, 20 hPa |
| Low Sunshine | **PRMX Climate Parametrics** | `SunshineDurationSumLte` | `sunshine_duration_7d_binary` | Fixed 7 days | 7200, 14400, 21600, 28800 s |
| River Discharge | **PRMX Climate Parametrics** | `RiverDischargeMaxGte` | `river_discharge_max_7d_binary` | Fixed 7 days | 500, 1000, 2000, 5000 m3/s |
| Wave Height | **PRMX Climate Parametrics** | `WaveHeightMaxGte` | `wave_height_max_7d_binary` | Fixed 7 days | 2, 3, 4, 6 m |
| Air Quality (PM2.5) | **PRMX Climate Parametrics** | `Pm25MaxGte` | `pm25_max_7d_binary` | Fixed 7 days | 35, 50, 100, 150 ug/m3 |

`PrecipTypeOccurred` thresholds are bitmasks:
- `2` = snow
- `4` = ice/freezing
- `6` = snow or ice

### Catalog-level products (pricing API)

- The V4 pricing catalog currently exposes **14 active products** (`/v4/catalog/info/live`).
- Removed from active catalog due null ERA5 data: UV Index, Drought (Soil Moisture), Low Visibility.
- Full taxonomy and product ownership is defined in `docs/product/PRODUCT-LINEUP.md`.

### Monetary and unit conventions

- Monetary values use the settlement asset with **6 decimals**. The current migration target is **USDC**.
- On-chain numeric thresholds use scaled units (`x1000`) for numeric event types.
- Coverage is represented as `shares x payoutPerShare`.

### Geographic scope

- **40 cities** grouped into **9 regions**.

---

## Primary personas

- **Buyer** (customer/user role): needs quick coverage setup and clear trigger semantics.
- **LP / underwriter** (lp role): evaluates premium, exposure, settlement trust, and pending liquidity risk.
- **Operator** (dao role): monitors health, DAO behavior, ingestion/timeline integrity, vault status, and AI agent activity.

---

## UX principles

- **Compact and information-dense**: prioritize concise rows and clear labels.
- **Minimal noise**: avoid decorative clutter in purchase/monitoring surfaces.
- **Clear state**: each screen should answer “what is happening now?”

See `docs/guidelines/UI-DESIGN-PRINCIPLES.md` for component-level patterns.

---

## Information architecture (routes)

The app uses Next.js App Router.

### Landing pages

| Route | Purpose |
|-------|---------|
| `/` | Landing page and entry points |
| `/products` | Product lineup hub (3 product lines) |
| `/products/[line]` | Per-product-line detail (rain-guard, weather-gate, climate-parametrics) |
| `/provide-liquidity` | LP onboarding landing page |
| `/manila` | Manila landing hub |
| `/start` | Onboarding flow entry |
| `/start/[step]` | Onboarding step pages |
| `/climate-peril-map` | Interactive climate hazard map (standalone, outside workspace) |

### Purchase and product workspace

| Route | Purpose |
|-------|---------|
| `/climate-parametrics` | **Protection Terminal** — unified purchase flow for all 14 products |
| `/rainguard` | Rain Guard standalone purchase page (product-line scoped tabs) |
| `/weather-gate` | Weather Gate standalone purchase page (product-line scoped tabs) |

### Policy and request lifecycle

| Route | Purpose |
|-------|---------|
| `/policies` | Policy list (tabbed by status, searchable) |
| `/policies/[id]` | Policy detail with timeline + settlement context |
| `/rainguard/policy/[id]` | Rainguard-branded policy detail page |
| `/weather-gate/policy/[id]` | Weather Gate-branded policy detail page |
| `/climate-parametrics/policy/[id]` | Climate Parametrics-branded policy detail page |
| `/my-policies` | User's own policies (filtered by connected wallet) |
| `/requests/new` | Create request entry (redirects to purchase flow) |
| `/requests/[id]` | Request detail (fill status, cancellation, accept shares) |
| `/markets` | City/region market browser (40 cities, 9 regions) |
| `/markets/[id]` | Market/location detail |

### Dashboard and operations

| Route | Purpose |
|-------|---------|
| `/dashboard` | User capital path overview: wallet link, deposit, active policies, exit |
| `/lp` | LP portfolio and trading flows |
| `/equity` | Equity token (buy, vest/unlock, claim dividends) — operator/DAO gated |
| `/vault` | Vault dashboard (vault reporter, Hyperlane yield transport, discovered vaults) — DAO only |
| `/dao` | DAO operations and request processing view — DAO only |
| `/oracle` | Oracle status and diagnostics — DAO only |
| `/oracle-service` | Oracle service status |
| `/agents` | AI Agent dashboard (portfolio, activity, per-agent detail) — DAO only |
| `/admin` | Control Plane (system health, capital, break-glass) — DAO only |
| `/deposit` | User bridge-in flow and Hyperlane delivery status |
| `/exit` | User canonical PRMX -> Base exit request and release/finalization status |
| `/history` | Capital and policy lifecycle history with Hyperlane evidence links |
| `/settings` | Wallet/environment controls |
| `/help` | Docs and FAQ |

### Shared integration capital UX

Current shared integration UX is split between user workflows and operator workflows:

- `/dashboard`, `/deposit`, `/exit`, `/history`: normal user path for wallet link, Base-side allowance + bridge-in initiation, exit submission, cross-chain status tracking, and explorer evidence
- `/settings`: wallet/environment controls only
- `/admin`: DAO/operator control plane for watcher health, Hyperlane relayer / yield transport health, capital invariants, reserve-return and exit-release status, and break-glass actions

Normal-user navigation must not expose Oracle Auth, Control Plane, relayer
health, or invariant panels. Direct visits to `/oracle`, `/vault`, `/agents`,
`/dao`, `/admin`, and `/equity` should show role/connection gating before any
operator-only polling occurs.

The UI should treat the following as distinct values:

- `policy reserve`
- `user voluntary yield`
- `prepared settlement liquidity`
- `available policy reserve`

User-visible PRMX settlement balances should be derived from the canonical
`pallet-assets(1)` state and its Hyperlane lifecycle. The PRMX EVM synthetic
token must not be presented as an independent balance ledger.

The source of truth for shared integration addresses is the generated deployment manifest and env snippets under `evm/abi/deployments/`, not hand-edited frontend docs.

---

## Core user journey: buy cover

### Step 0 - Wallet state

- Users can browse read-only without connecting.
- Submission requires wallet connection.
- Wallet panel shows account, chain connection, and balance.

### Step 1 - Product selection

On `/climate-parametrics` (Protection Terminal), users select a product from dropdown groups: Rain Guard, Weather Gate, and Climate Parametrics.

Design requirement:
- Every tab must show a one-line trigger explanation.
- Threshold options should be restricted to supported bucket values.

### Step 2 - Location selection

- City selector lists all 40 cities grouped by region.
- Search supports city/country keywords.
- Selection displays city, country, and coordinates context.

### Step 3 - Configure coverage

Inputs:
- Coverage start (validated)
- Duration (product-constrained)
- Threshold (bucket-constrained)
- Shares/coverage amount

Date constraints:
- Minimum coverage start lead: **24 hours**.
- Maximum future start: **365 days**.

### Step 4 - Quote

Quote panel shows:
- Premium per share
- Total premium
- Implied probability
- Max payout

Pricing path:
- Frontend calls `/api/pricing` proxy.
- Proxy enforces supported durations (`24` or `168` hours).
- Proxy enforces supported threshold buckets per mapped peril.

### Step 5 - Submit request

On submit:
- Show compact confirmation summary (location, product, threshold, window, premium, payout).
- Submit `createUnderwriteRequest`.

After submit:
- Show request/policy id immediately.
- Show request fill lifecycle and transition into policy monitoring.

---

## Lifecycle UX

### Request states

- `Pending`
- `PartiallyFilled`
- `FullyFilled`
- `Cancelled`
- `Expired`

### Policy/monitoring states

- `Waiting`
- `Active`
- `Triggered`
- `Matured`

### Financial/settlement states

- `Escrowed`: policy capital remains locked.
- `Pending Settlement`: oracle outcome is known, but PRMX is still preparing local liquidity.
- `Settled`: payout or LP distribution has finalized.

### Policy capital UX

Policy list and detail surfaces should expose capital readiness without turning
operator diagnostics into user alarms.

- Policy list rows show settlement capacity as
  `latestVaultAssets / maxPayout`, with buckets: surplus `>= 102%`, covered
  `>= 99.95%`, at-risk `>= 95%`, deficit below that.
- The annualized APY badge appears between capacity and report freshness when
  `initialVaultAssets`, `latestVaultAssets`, and at least one hour of elapsed
  time are available. It is hidden for settled policies.
- Report freshness prefers `latestVaultReportBlock` and falls back to
  `vaultReportDirectBlock`. Vault-backed active policies show `awaiting report`
  until the first report arrives and are stale after 25 hours, matching the
  24-hour reporter cycle plus a small grace window.
- Detail pages show `Payout Capacity` at the top of `Capital & Funds` using
  the same `latestVaultAssets / maxPayout` ratio and the same display buckets.
- The `Cross-Chain Vault Report` section is shown for vault-backed policies
  even when no report has arrived yet. States are awaiting first report, fresh,
  and stale.
- `Live NAV (off-chain monitor)` may appear on detail pages for Morpho-backed
  policies. It comes from the display-only Base RPC probe and is compared
  against canonical `latestVaultAssets`; it must not be presented as settlement
  authority.
- Yield health distinguishes `NO REPORT YET`, `AWAITING YIELD`, `YIELD
  STALLED`, `EARNING`, and `LOSS` so a zero-yield report is not confused with a
  missing report.

### Policy detail timeline

Timeline data is sourced from oracle service records (Supabase):
- `SNAPSHOT_STORED`
- `SNAPSHOT_ANCHORED`
- `EVALUATION_RESULT`
- `CROSS_VALIDATION_COMPLETED`
- `PAYOUT_CONFIRMED`

Design requirement:
- Timeline and status badges must distinguish `oracle outcome known` from `funds delivered`.
- If a policy is waiting on capital readiness, the primary state should read `Pending Settlement` rather than `Triggered` or `Matured` alone.

Each timeline row should show event type, timestamp, status, and optional tx hash.

### Cross-validation badge

- `High`: all sources agree
- `Medium`: partial agreement
- `Low`: disagreement
- `Insufficient data`: reanalysis lag/no data

Cross-validation is informational only; on-chain settlement is authoritative.

---

## Operator experience

Admin/operations surfaces should clearly display:
- Chain online status and latest block
- Oracle service health
- Supabase status
- DAO status and recent accept/reject outcomes
- Warp invariant severity and recent samples
- Hyperlane yield-report transport mode
- Morpho live NAV probe status when enabled
- Rebalancer monitor / decision / executor health
- Reserve-return and canonical exit release evidence

Recommended quick actions:
- Copy WS/API endpoints
- Link to restart docs (`docs/operations/LOCAL-RESTART-PROMPT.md`, `docs/operations/CLOUD-RESTART-PROMPT.md`)

---

## Error and edge-case UX

- **Chain disconnected**: persistent banner + retry/settings links.
- **Oracle service unavailable**: disable timeline-dependent widgets, keep chain reads.
- **DAO rejection**: show explicit rejection reason from DAO records.
- **Observation lag**: show pending observation status instead of hard failure.
- **Pricing unavailable**: explicit pricing error state (no silent fallback).

---

## Acceptance criteria

A purchase flow is complete when:

1. Request is created on-chain.
2. Request is accepted (or rejected with reason).
3. Oracle reaches a terminal outcome and PRMX requests settlement liquidity.
4. Policy may remain in `Pending Settlement` until capital readiness is confirmed.
5. Timeline exposes the finalized settlement/audit event, including `PAYOUT_CONFIRMED`.

---

## Appendix: terminology

- **Peril ID**: pricing-catalog product identifier (for quote lookup).
- **Event type**: on-chain trigger type used in `eventSpec`.
- **Threshold bucket**: supported discrete threshold values per product.
- **Shares**: payout multiplier unit.
- **Early trigger**: policy resolves immediately when trigger condition is met.
