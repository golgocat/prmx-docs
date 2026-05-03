# PRMX Parametric Insurance Rulebook
Design, implementation, and verification rules for building robust parametric insurance.

> Settlement asset migration note: this rulebook historically used `USDT` as the canonical settlement asset. The current implementation target is `USDC`, with PRMX-side internal settlement credits backed by canonical USDC on the EVM side. Remaining `USDT` references in this document should be read as settlement-asset legacy terminology until the text is fully migrated.

## 0. Purpose
Maintain the following properties across all insurance products offered by PRMX:

- No implementation without unambiguous definitions
- No divergence between specification and implementation
- Resilience against unexpected input, missing data, delays, duplicates, and tampering
- Pricing, underwriting, event monitoring, decision, and payout bound by consistent invariants
- Reproducibility, auditability, and explainability at all times

This document is a **rulebook** — no exceptions. If an exception is needed, extend the specification, add tests and audit logs, and define compatibility and migration procedures at the same time.

### 0.1 Implementation Status Legend
Each item is annotated with one of the following:
- ✅ Implemented — working in the current codebase
- 🔧 Partial — directionally correct but incomplete
- 📋 Planned — to be implemented
- 🔮 Future — not needed for current products but required when extending

## 1. Terminology and Common Definitions
Do not implement anything while undefined terms remain in this section.

### 1.1 Entities
Mapping between rulebook terms and the current V4 Rainguard implementation:

| Rulebook Term | Implementation Name | Description | Status |
|---|---|---|---|
| Policy | PolicyV4 (on-chain) | Insurance contract. Contains coverage period, covered peril, threshold, payout terms, premium, and max payout | ✅ |
| Exposure | LocationId + EventSpec | Unit of risk. Defined as 40 cities x 14 event types | ✅ |
| Event | EventSpec (EventTypeV4 + threshold) | Observable natural phenomenon. 14 active event types (see docs/product/V4-PRODUCT-CATALOG.md) | ✅ |
| Observation | WeatherObservation | Hourly weather data point. epoch_time, precip_1h_mm_x1000, temp_c_x1000, wind_gust_mps_x1000 | ✅ |
| Trigger | PolicyTriggered (on-chain event) | Final settlement outcome after threshold is exceeded and liquidity is ready. Oracle detection may precede final settlement. | 🔧 |
| Claim | (not applicable) | Parametric insurance uses automatic determination. No manual claim filing | ✅ (by design) |
| Payout | Settlement (on-chain) | Fixed payout of shares x 100 USDC. On-chain transfer | ✅ |

### 1.2 Lifecycle
Current implementation lifecycle target:
```
Request (created) → Accepted (underwritten) → Active (coverage in force) → Triggered/Matured (oracle outcome) → Pending Settlement (liquidity requested) → Settled (paid out)
```

### 1.3 Time Definitions 🔧
- All timestamps are UTC. Local conversion is allowed only for display ✅
  - Frontend: unified via `formatDateTimeUTC()`
  - On-chain: `timezone=UTC` parameter
  - Oracle Service: `timezone=auto` — **to fix: should be unified to UTC** 📋
- Policy period:
  - Current implementation uses `coverage_start <= epoch_time <= coverage_end` (closed interval) ✅ (kept as-is)
  - Note: the difference between closed and half-open intervals is only 1 second at end_at. Not material for current products (day-granularity periods) but needs attention for future short-term products
- Timestamps:
  - On-chain: Unix seconds (epoch_time) ✅
  - Frontend: milliseconds (`Date.now()` basis) ✅
  - Auto-conversion: if `timestamp >= 10^12`, treated as milliseconds and divided by 1000 ✅
- Separation of `source_timestamp` and `ingested_at`:
  - `epoch_time` (= source_timestamp) is implemented ✅
  - `ingested_at` is not implemented 📋 — Supabase `observations` table has `created_at` but no explicit `ingested_at`

### 1.4 Units and Scale ✅
All values are normalized to integers. Current implementation:

| Value | Scale | Example | Implementation |
|---|---|---|---|
| Precipitation | mm x 1000 | `75000` = 75 mm | `http_client.rs`, `fetcher.rs` |
| Temperature | C x 1000 | `25500` = 25.5 C | `http_client.rs` |
| Wind speed | m/s x 1000 | `20000` = 20 m/s | `http_client.rs` (km/h to m/s conversion) |
| Coordinates | degrees x 1,000,000 | `14599512` = 14.599512 deg | `cities.ts`, `http_client.rs` |
| USDC | 10^6 (6 decimals) | `100000000` = 100 USDC | on-chain |
| PRMX | 10^12 (12 decimals) | — | on-chain |

- Scale is embedded in field names by convention (`precip_1h_mm_x1000`)
- Rounding occurs only at input normalization ✅
- All subsequent operations use integer arithmetic ✅

### 1.5 Invariants
Must hold across all modules. Changes that violate these cannot be merged.

| Invariant | Status | Notes |
|---|---|---|
| **Reproducibility**: identical inputs always produce identical decisions | ✅ | Deterministic OCW evaluation |
| **Determinism**: decisions do not depend on randomness, external state, or non-deterministic ordering | ✅ | `evaluate_threshold()` is a pure function |
| **Monotonicity**: adding observations never reverses a finalized decision | ✅ | `triggered` is irreversible |
| **Idempotency**: processing the same event multiple times converges to one result | 🔧 | Implicitly guaranteed by state machine. No explicit idempotency key |
| **Completeness**: when data is insufficient, defer rather than mis-judge | 📋 | Currently missing data = skip (no decision). No explicit pending state |
| **Auditability**: every decision is linked to input hashes, rule version, and source signature verification | 🔧 | Blake2-256 commitment chain exists. Rule version is not recorded |

## 2. Specification Versioning 📋

### 2.1 Rule Versions
Currently only implicit versioning (pallet name V3/V4).

Future implementation plan:
- ProductVersion: product specification version (e.g., `rainguard-storm-v1.0`)
- EngineVersion: decision engine version (pallet build hash)
- DataSchemaVersion: data schema version (Supabase migration number)

Policies must reference a fixed (ProductVersion, EngineVersion, DataSchemaVersion) tuple.
Past policies must be reproducible under their original versions.

### 2.2 Compatibility Rules
- Changes that alter the semantics of existing policies are forbidden
- If a change is necessary, create a new ProductVersion
- If migration is required, submit a migration plan and backfill plan together
- Do not break the audit log format

## 3. Data Sources and Trust Model

### 3.1 Open-Meteo: Primary Data Provider ✅

**API specification:**
```
Endpoint: https://archive-api.open-meteo.com/v1/archive
Parameters:
  latitude, longitude       — position (degrees, 6 decimal places)
  hourly=precipitation,...  — hourly data
  models=ecmwf_ifs          — ECMWF IFS historical model
  past_days=1               — past 24 hours of observed data
  forecast_days=1           — same-day archive rows; future rows are filtered out
  timezone=UTC              — UTC basis
```

**Data characteristics:**

| Property | Value | Rulebook Impact |
|---|---|---|
| Time resolution | 1 hour | Sufficient for 12h max and 24h sum ✅ |
| Data update cadence | Every 6 hours | Settlement normally waits for the next historical update ✅ |
| Preliminary/final distinction | Historical archive rows only | Forecast API is not a settlement source ✅ |
| Coverage | Global ~9 km resolution | All 40 cities covered ✅ |
| Rate limits (free tier) | 600/min, 10,000/day | 40 cities x hourly = 48 calls/day, well within limits ✅ |
| Commercial use | Paid plan required | Must migrate to paid plan before production |
| WMO compliance | Yes | Used for weather code classification ✅ |

**Oracle source rule:**
1. Settlement decisions use Open-Meteo Historical Weather API / ECMWF IFS.
2. Forecast API data may be displayed as reference, but must not submit final reports.
3. Future archive rows are filtered by `now_epoch` before aggregation.

### 3.2 Cross-Validation Data Sources ✅
Multi-source strategy for improved decision accuracy:

| Layer | Source | Endpoint | Latency | Independence | Status |
|---|---|---|---|---|---|
| Settlement source | Open-Meteo Historical API (ECMWF IFS) | `archive-api.open-meteo.com/v1/archive` | 6h update cadence | Open-Meteo historical model (~9 km) | ✅ `http_client.rs` |
| Legacy reference only | Open-Meteo Forecast API | `api.open-meteo.com/v1/forecast` | Near real-time | NWP model output | 🔒 not settlement |
| Confirmation / audit | Open-Meteo Historical API (ECMWF IFS) | `archive-api.open-meteo.com/v1/archive` | 6h update cadence | ECMWF IFS (~9 km) | ✅ `era5.ts` |
| Independent verification | NASA POWER API (MERRA-2) | `power.larc.nasa.gov/api/temporal/hourly/point` | Near real-time | NASA reanalysis (~50 km) | ✅ `nasa-power.ts` |
| Ground truth (US cities) | NOAA NCEI (ISD) | `ncei.noaa.gov/access/services/data/v1` | Hours to days | Station observations | 🔮 |

Open-Meteo sources are free for non-commercial use and require no API keys. NASA POWER remains an optional independent audit source.

**Cross-validation module** (`cross-validator.ts`):
- Aligns hourly precipitation series by hour-rounded epoch time
- Computes per-source summaries (total, 12h max, 24h max)
- Calculates Pearson correlation between source pairs
- Determines trigger agreement per secondary source
- Produces confidence: `high` (all agree) / `medium` (partial) / `low` (all disagree) / `insufficient_data`
- API: `POST /api/policies/:policyId/cross-validate`
- Results logged as `CROSS_VALIDATION_COMPLETED` timeline event

### 3.3 Data Acquisition Principles 🔧
- Outages, delays, and duplicates are assumed to always occur
- OCW fetch failure: recovered automatically on next fetch via `past_days=1` (covers past 24h) ✅
- Outage > 24 hours: data is permanently lost under current design (weakness)
- Future mitigation: extend to `past_days=2`, or backfill from Historical API 📋

### 3.4 Observation Validation 🔧
Validations performed at ingest time, with results logged:

| Validation | Status | Notes |
|---|---|---|
| Schema validation: required fields, types | ✅ | TypeScript type checking + Open-Meteo response parsing |
| Timestamp validation: reject future timestamps, check delay tolerance | 📋 | Not implemented. Any epoch_time is accepted |
| Unit validation: unit and scale match specification | ✅ | mm x 1000 normalization in http_client.rs |
| Deduplication | ✅ | By (source_id, source_timestamp, metric_id) |
| Range validation: reject negative precipitation, outlier detection | 📋 | Not implemented |
| Quality flags | ✅ | Settlement uses the historical archive source; forecast rows are excluded from final reports |

### 3.5 Finalization and Revision
Open-Meteo Forecast API values are not final settlement input. Finalization rules:
- ECMWF IFS historical archive rows are treated as the settlement input ✅
- OCW re-fetches historical rows and updates observations as the 6-hour archive cadence advances
- Already-finalized on-chain decisions are not automatically changed

## 4. Underwriting

### 4.1 Required Inputs for Underwriting ✅
Inputs required at Policy (Request) creation:

| Field | Type | Implementation |
|---|---|---|
| location_id | u32 | Index into 40-city registry (0-39) ✅ |
| coverage_start, coverage_end | Unix seconds | Coverage period ✅ |
| event_spec | V3EventSpec | metric_id + threshold + operator ✅ |
| shares | u64 | Coverage units (1 share = 100 USDC payout) ✅ |
| premium | u128 (USDC 6 decimals) | Premium amount ✅ |
| max_payout | implicit | shares x 100 USDC (auto-calculated) ✅ |

### 4.2 DAO Auto-Underwrite ✅
DAO auto-underwrite acceptance criteria:

| Criterion | Value | Status |
|---|---|---|
| locationWhitelist | All 40 cities (id=0-39) | ✅ |
| eventTypeWhitelist | All 14 active event types (Precip12hMaxGte, PrecipSumGte, Precip1hGte, TempMaxGte, TempMinLte, WindGustMaxGte, PrecipTypeOccurred, HeatIndexMaxGte, SnowDepthMaxGte, PressureDropMaxGte, SunshineDurationSumLte, RiverDischargeMaxGte, WaveHeightMaxGte, Pm25MaxGte) | ✅ |
| maxSharesPerRequest | 5000 | ✅ |
| daoSharePercent | 50% | ✅ |
| premiumTolerance | 20% | ✅ (validated against Pricing API) |

### 4.3 Acceptance Conditions 🔧
- Basic on-chain validation exists ✅
- Systematic acceptance checks (threshold `<` 0, zero-length period, etc.) are insufficient 📋
- Coverage gap control is handled implicitly via location_id whitelist ✅

### 4.4 Contract Immutability ✅
- After policy issuance, trigger definition, period, location, and payout function are immutable (enforced on-chain)
- Changes require cancellation and reissuance

## 5. Pricing 🔧

### 5.1 Current Pricing Models
- **Open-Meteo Observed Catalog (current)**: ERA5 reanalysis frequency counting, 14 products, 40 cities ✅
- **NeuralGCM Ensemble (deprecated)**: AI climate forecasting model, replaced by Open-Meteo catalog
- **R Markov Chain (offline)**: Legacy statistical model, Luzon-only

### 5.2 Pricing Responsibilities
| Output | Status | Notes |
|---|---|---|
| premium | ✅ | Calculated by DAO pricing API |
| expected_loss, variance | 📋 | Risk metrics not returned |
| Loading breakdown | 📋 | No decomposition into fees, cost of capital, operating costs |
| Model version | 📋 | No Pricing API version tracking |
| Input hash | 📋 | No reproducibility hash for full input set |

### 5.3 Pricing Invariants
- Premium must not exceed max_payout ✅ (implicit)
- Identical inputs return identical premium ✅ (deterministic API)
- Currency is fixed as USDC. No FX conversion required ✅

## 6. Event Monitoring

### 6.1 Monitoring Architecture ✅
```
Open-Meteo API (hourly data)
  |
  v
OCW HTTP Client (normalize to mm x 1000 integers, classify WMO codes)
  |
  v
Fetcher (12h sliding window with 13-hour lookback)
  |
  v
Observations array (hourly WeatherObservation structs)
  |
  v
Ingest API (send to Oracle Service)
  |
  v
Supabase (observations, snapshots, timeline_events)
```

### 6.2 State Management 🔧
Per-policy state:

| State | Status | Notes |
|---|---|---|
| last_processed_source_timestamp | ✅ | Managed on-chain as `last_seen_epoch` |
| aggregation_state | ✅ | V3AggState: 12h ring buffer, cumulative precipitation |
| data_gaps (missing interval records) | 📋 | Hours skipped by OCW are not recorded |
| finalized (decision finalization flag) | 🔧 | oracle outcome is irreversible; settlement finalization remains a separate terminal step |

### 6.3 Window Aggregation ✅
**12-hour rolling max** (`Precip12hMaxGte`):
- `compute_12h_max_from_batch()` (`fetcher.rs:182-221`)
- Sliding window of 43,200 seconds over sorted observations
- 13-hour lookback for cross-batch continuity (`fetcher.rs:276-316`)
- Sums precipitation within each window, tracks maximum

**24-hour cumulative precipitation** (`PrecipSumGte`):
- `sum_mm_x1000.saturating_add(observation.precip_1h_mm_x1000)`
- Simple accumulation ✅

### 6.4 Open-Meteo Monitoring Constraints
- **Data latency**: most recent 1-3 hours may be unavailable
- **Gap recovery**: on fetch failure, next fetch retrieves past 24h via `past_days=1`, automatically recovering from transient outages ✅
- **Outage > 24 hours**: data beyond `past_days=1` is permanently lost under current design
- **Future mitigation**: extend to `past_days=2`, or backfill via Historical API 📋

## 7. Event Decision

### 7.1 Decision Model (Current Implementation) ✅
The current V4 implementation uses an **immediate finalization model**:

```
Data fetch -> Threshold evaluation -> triggered (final) or continue monitoring
```

The three-stage model (provisional -> eligible -> final) is **not implemented**.
Reason: settlement now uses Open-Meteo Historical Weather / ECMWF IFS directly rather than a separate forecast confirmation stage.

Conditions for future introduction of the three-stage model 🔮:
- Cross-validation with multiple independent data sources is introduced
- A separate post-verification source is added
- Large contracts require a pre-finalization approval flow

### 7.2 Decision Logic ✅
Current implementation:
- Input: Policy (event_spec, threshold), Observation set, aggregation_state
- Output: boolean (triggered: true/false)
- Operation: integer comparison (`computed_value >= threshold_x1000`)

Future extension (structured decision output) 📋:
```typescript
interface TriggerDecision {
  status: 'triggered' | 'not_triggered' | 'pending';
  reason_code: string;        // machine-readable code
  reason_text: string;        // human-readable text
  evidence_hashes: string[];  // hashes of input observations
  computed_metrics: {
    value: number;            // computed value (mm x 1000)
    threshold: number;        // threshold (mm x 1000)
    operator: string;         // '>='
    window: string;           // '12h' | '24h'
  };
}
```

### 7.3 Threshold Strictness ✅
- Operator: `>=` (greater than or equal) for most products; `<=` for Frost (`TempMinLte`) and Low Sunshine (`SunshineDurationSumLte`)
- All event types: see `docs/product/V4-PRODUCT-CATALOG.md` for per-product trigger conditions and threshold buckets
- Threshold unit: varies by product (mm x 1000, °C x 1000, m/s x 1000, etc.) — all scaled to integers on-chain
- Comparison uses integer arithmetic ✅

### 7.4 Pending Rules 📋
Currently, insufficient data results in a skip (no decision). No explicit pending state is implemented.

Implementation plan:
- Condition for pending: data gap during coverage period exceeds threshold (e.g., 6 consecutive hours)
- Retry: next OCW execution recovers via `past_days=1`
- grace_period: `coverage_end + 72 hours` (accounting for Open-Meteo latency)
- Final disposition after grace_period: decide using available data. If undecidable, process as `not_triggered` and log the reason

## 8. Payout

### 8.1 Payout Prerequisites ✅
Payout execution requires:
- Oracle outcome has reached a terminal state (`Triggered` or `Matured`)
- Local liquidity has been confirmed for settlement finalization
- Decision is irreversible (no rollback from triggered)
- Settlement managed by `triggered/matured -> pending settlement -> settled` state machine

### 8.2 Payout Function ✅
Current implementation uses a **fixed-amount step function**:
```
payout = shares x 100 USDC
```
- Below threshold: payout = 0
- At or above threshold: flat `shares x 100 USDC`

Future extensions 🔮:
- Linear: proportional from threshold to cap
- Piecewise: piecewise linear
- These require adding a payout_curve parameter

### 8.3 Double-Payment Prevention 🔧
- Implicitly prevented by state machine (`triggered/matured -> pending settlement -> settled`) 🔧
- Explicit `payout_id` idempotency key is not implemented 📋
- On-chain `PolicyTriggered` event records `tx_hash` ✅

### 8.4 Failure Recovery 🔧
- Error events are recorded in timeline ✅
- Systematic `retry_policy` is not defined 📋
- Failure classification (transient / permanent / validation) is not implemented 📋

## 9. Audit Logs and Explainability

### 9.1 Current Logging System 🔧
`policy_timeline_events` table:

| Field | Status | Notes |
|---|---|---|
| event_type | ✅ | observation, snapshot, evaluation, trigger, settlement |
| payload | ✅ | Detailed data in JSON format |
| payload_hash | ✅ | Hash of payload |
| tx_hash | ✅ | On-chain transaction hash |
| block_number | ✅ | Block number |
| rule_version | 📋 | Not recorded |
| evidence_hashes | 📋 | Individual observation hashes not recorded |

**Warning**: Supabase timeline events have a **90-day TTL** and are auto-deleted.
An archival strategy is needed for long-term retention 📋

### 9.2 Blake2-256 Commitment Chain ✅
- Sample hashes of each observation are chained on-chain
- Implemented in `commitment.rs`
- However, replay from individual observations is not supported 📋

### 9.3 Replay 📋
Not implemented. Future plan:
- CLI tool to reproduce decisions for any policy
- Required data is snapshotted for network-independent execution
- Combined with golden tests for regression verification

## 10. Testing Standards

### 10.1 Current Test Framework 🔧

| Test Layer | Status | Notes |
|---|---|---|
| Unit | ✅ | Aggregation functions, comparisons, payout calculation |
| Property | 📋 | Property tests for idempotency, monotonicity, boundary conditions not implemented |
| Integration | ✅ | Ingest-to-decision pipeline |
| E2E | 🔧 | 75+ scenario test plan defined. Distribution: 40% happy path, 27% boundary, 33% adversarial |
| Regression | 📋 | Past incidents and bug reproduction cases need to be systematized |
| Golden | 📋 | Golden input/output fixtures need to be established |

### 10.2 Required Test Cases
The following must be covered by tests:

- [x] Rolling aggregation across batch boundaries (13-hour lookback) ✅
- [ ] Late-arriving data 📋
- [x] Duplicate observations (deduplication implemented, tests need verification)
- [ ] Missing data intervals 📋
- [x] Period boundaries: start_at, end_at handling ✅
- [x] Threshold equality boundary: exactly at threshold ✅ (`>=` comparison)
- [x] Max payout clipping ✅ (fixed at shares x 100 USDC)
- [ ] Payout idempotency: re-execution with same payout_id 📋

## 11. Monitoring and Alerts 📋

### 11.1 Current Monitoring
- Timeline events capture foundational data ✅
- DAO underwrite records track portfolio state ✅
- Frontend admin dashboard for service health ✅

### 11.2 Unimplemented Alerts
The following need to be implemented:
- Ingest delay rate, missing data rate 📋
- Pending decision count and aging 📋
- Payout failure rate and retry count 📋
- Daily total payout anomaly detection 📋
- **Payout kill switch** 📋 (emergency halt of all payouts)

## 12. Security and Permissions 🔧

| Requirement | Status | Notes |
|---|---|---|
| Oracle member permission management | ✅ | Managed on-chain |
| Separation of policy issuance / rule update / payout execution permissions | 📋 | Fine-grained permission separation not implemented |
| No signing keys in code | ✅ | Managed via environment variables |
| No sensitive information in logs | ✅ | — |

## 13. Change Procedures

### 13.1 Change Classification
- Bug fix: must not change the semantics of existing versions
- Specification addition: new ProductVersion
- Data schema change: new DataSchemaVersion with migration

### 13.2 Required Deliverables for Changes
- Change rationale and impact scope
- Invariant impact assessment
- Test additions
- Audit log diff verification

## 14. Pre-Merge Checklist
Must be satisfied before merging.

- [ ] Terms and units are defined
- [ ] Idempotency is guaranteed
- [ ] Cross-batch-boundary aggregation is correct
- [ ] Missing data and delays do not cause incorrect decisions
- [ ] Results do not change after triggered
- [ ] No double payments
- [ ] Decision rationale can be explained from timeline_events

## 15. Supplementary Documents (To Be Created)

| Document | Purpose | Status |
|---|---|---|
| `docs/product_specs/` | Per-product-type specification | 📋 `docs/product/RAINGUARD-PRODUCT-SPEC.md` partially covers this |
| `docs/data_sources/` | Per-source characteristics, latency, finalization rules | 📋 |
| `docs/payout_curves/` | Payout function catalog and test vectors | 📋 |
| `docs/incidents/` | Past bugs and prevention measures | 📋 |
| `tools/replay/` | Replay CLI and procedures | 📋 |
