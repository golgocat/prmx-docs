# PRMX Test Design Rulebook

> **V4 Rainguard — agent reference.** In PRMX, test quality defines product quality. This rulebook **overrides** implementation convenience, development speed, UI/UX priorities, and short-term business decisions.

## 1. Absolute principles

### Principle 1: Testing is an act of destruction

Tests do not confirm expected behavior; they exist to **break the system**.

- At least **50%** of test cases must be abnormal, boundary, or adversarial.
- Test suites composed only of happy paths are **invalid**.

### Principle 2: Money and finality are P0

Always treat as P0: insurance payouts, Early Trigger, maturity settlement, idempotency (no double execution), policy state transitions.

P0 logic must be tested at **all three layers**: unit, integration, end-to-end.

### Principle 3: Oracles are hostile by default

Oracles are unreliable and adversarial external systems. Mandatory oracle-failure tests:

| Failure | Required test |
|---|---|
| Delayed data | Trigger-window late delivery |
| Missing data | Intermediate / final observation absent |
| Duplicated submissions | Idempotency on retry |
| Out-of-order delivery | Snapshot reordering |
| Invalid values | NaN, negative, extreme magnitudes |
| Partial success | Off-chain success but on-chain failure |

## 2. Mandatory test classification

Every test must carry **one or more** of these labels (unclassified tests are invalid):

| Code | Classification |
|---|---|
| **A** | Economic Integrity |
| **B** | State Machine Safety |
| **C** | Temporal Consistency |
| **D** | Off-chain Interaction |
| **E** | Adversarial User Behavior |

## 3. V4 product model (14 active products)

| Line | Product | Event Type | Time model | Notes |
|---|---|---|---|---|
| Rain Guard | Storm Protection | `Precip12hMaxGte` | 12 h rolling max | Sensitive to window boundaries |
| Rain Guard | Daily Total | `PrecipSumGte` | 24 h cumulative | Vulnerable to ordering and missing data |
| Rain Guard | Hourly Intensity | `Precip1hGte` | 1 h max | Highest temporal resolution |
| Weather Gate | Heat | `TempMaxGte` | 7 d max | `≥ threshold` |
| Weather Gate | Frost | `TempMinLte` | 7 d min | `≤ threshold` (inverted) |
| Weather Gate | Wind Damage | `WindGustMaxGte` | 7 d max | |
| Weather Gate | Precipitation Type | `PrecipTypeOccurred` | Bitmask | `snow=2, ice=4, snow-or-ice=6` |
| Climate Parametrics | Heat Index | `HeatIndexMaxGte` | 7 d max | |
| Climate Parametrics | Snow Accumulation | `SnowDepthMaxGte` | 7 d max | |
| Climate Parametrics | Pressure Drop | `PressureDropMaxGte` | 7 d max | |
| Climate Parametrics | Low Sunshine | `SunshineDurationSumLte` | 7 d sum | `≤ threshold` (inverted) |
| Climate Parametrics | Flood | `RiverDischargeMaxGte` | 7 d max | |
| Climate Parametrics | Marine Storm | `WaveHeightMaxGte` | 7 d max | |
| Climate Parametrics | Air Quality | `Pm25MaxGte` | 7 d max | |

All 14 products share oracle infrastructure (Open-Meteo, Supabase) and Pricing API validation. Two operators are inverted (`TempMinLte`, `SunshineDurationSumLte`). **Highest risk: incorrect product routing or evaluation.**

## 4. Mandatory declarations per test

Every test case must declare:

- **Target product(s)** — one or more of the 14, or "all".
- **Time model** — rolling-window / cumulative / min-tracking / bitmask / other.
- Whether **product routing** is involved.
- Whether **migration or upgrade** is involved.

> Undeclared tests are forbidden.

## 5. Storm Protection rules (`Precip12hMaxGte`)

> **Storm Protection is most fragile at window boundaries.**

| Failure mode | Mandatory test |
|---|---|
| 12 h window boundary inclusion/exclusion | Exact-boundary tests |
| Delayed data causing missed trigger | Late oracle delivery where trigger should have fired |
| Duplicated oracle reports | Idempotency on retry |
| Window persistence across batches | 13-hour history span |

## 6. Daily Total rules (`PrecipSumGte`)

> **Daily Total is most fragile with ordering and missing data.**

Cumulative values must be defined by a **set of observations**, not arrival order. If arrival order matters, invalid ordering must be **explicitly rejected**.

Mandatory P0 tests: missing intermediate / final observations, reversed snapshot order, duplicated timestamps, Early Trigger followed by maturity settlement (no double payout), duration boundaries (min and max), abnormal values, cumulative error amplification.

## 7. Cross-product & infrastructure

Every infrastructure test must carry at least one tag:

| Tag | Scope |
|---|---|
| **A** | Product coexistence (multiple products evaluated from same oracle data) |
| **B** | Oracle infrastructure (Open-Meteo, Supabase, OCW) |
| **C** | DAO auto-underwrite (Pricing API validation) |
| **D** | Cross-validation (ERA5 + NASA POWER audit) |

### 7.1 Product routing (P0)

- Each of the 14 event types routes to its correct evaluator (rolling-window, cumulative, min-tracking, bitmask).
- Inverted operators (`≤`) for `TempMinLte` and `SunshineDurationSumLte` are correctly applied.
- `PrecipTypeOccurred` bitmask matching works for all mask values (2, 4, 6).
- Unknown event types are rejected.
- DAO whitelist filtering works per product type (14 event types × 40 cities).

### 7.2 Migration & upgrade safety (P0)

- Migration is idempotent (0, 1, N executions converge).
- Upgrade interruption and recovery.
- Active policies complete after upgrade.
- Total assets, reserves, and policy counts are preserved.

> Migration correctness is part of the **specification**, not an implementation detail.

### 7.3 Off-chain consistency (P0)

- Recovery from partial success.
- At-least-once delivery convergence.
- Replay safety.
- Oracle/node restarts do not create duplicate monitoring.
- Supabase write idempotency (deterministic composite keys).

### 7.4 Operational & governance controls (P0 if present)

> **Operational features must be attacked first, not last.**

- Behavior during pause.
- Settlement behavior for triggers occurring before pause.
- Authority boundaries.
- Behavior when risk limits are reached.

## 8. Mandatory test design template

| Field | Description |
|---|---|
| Test name | Unique identifier |
| Target product | One of 14, or "all" |
| Classification | A–E (Section 2) |
| Expected failure mode | What the test attempts to break |
| Attacker perspective | How a malicious actor would exploit this |
| Expected defense | How the system should prevent exploitation |
| Success criteria | Clear pass/fail conditions |
| Impact if broken | funds / trust / legal |

## 9. Coverage definition

> **Numeric (percentage) code coverage is not an accepted metric.**

Accepted coverage metrics:

- State-transition coverage
- Economic-scenario coverage
- Oracle-failure scenario count
- Migration-completion scenarios
- Cross-product arbitrage scenarios

## 10. AI agent self-audit

Before finalizing a test suite, the agent must verify:

- [ ] Can a malicious human profit if this fails?
- [ ] Will this survive future specification changes?
- [ ] Can the test's existence be clearly justified?

Tests that fail this self-audit must be **removed**.

## 11. Severity

| Severity | Description | Action |
|---|---|---|
| **P0** | Fund loss, double payout, product-routing failure | Immediate halt |
| **P1** | Unfairness, broken insurance experience | Fix before redeploy |
| **P2** | UI, logs, non-critical observability | Backlog |
