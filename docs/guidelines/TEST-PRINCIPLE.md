# PRMX Test Design Rulebook

**V4 Rainguard**
**Agent Reference (Cursor-ready)**

---

## Purpose

This rulebook defines the immutable rules for test design in PRMX.

> **In PRMX, test quality defines product quality.**

All humans and AI agents must follow this document when designing tests.

This rulebook **overrides**:
- implementation convenience
- development speed
- UI / UX priorities
- short-term business decisions

---

## 1. Absolute Principles

### Principle 1: Testing is an act of destruction

Tests are **not** for confirming expected behavior.
Tests exist to **break the system**.

**Mandatory:**
- At least **50%** of test cases must be abnormal, boundary, or adversarial
- Test suites composed only of happy paths are **invalid**

---

### Principle 2: Money and finality are P0

Anything related to money or finality is top priority.

**Always treat the following as P0:**
- Insurance payouts
- Early Trigger
- Maturity settlement
- Idempotency (double execution prevention)
- Policy state transitions

P0 logic must be tested at **all three layers**:
- Unit
- Integration
- End-to-End

---

### Principle 3: Oracles are hostile by default

Oracles must be treated as **unreliable and adversarial** external systems.

**Mandatory oracle failure tests:**
- delayed data
- missing data
- duplicated submissions
- out-of-order delivery
- invalid values (NaN, negative, extreme)
- partial success (off-chain success, on-chain failure)

---

## 2. Mandatory Test Classification

Every test must be classified into exactly one or more of the following:

| Code | Classification |
|------|----------------|
| **A** | Economic Integrity |
| **B** | State Machine Safety |
| **C** | Temporal Consistency |
| **D** | Off-chain Interaction |
| **E** | Adversarial User Behavior |

> Unclassified tests are invalid.

---

## 3. V4 Product Model (14 Active Products)

### Rain Guard (3 products)

**Storm Protection** (`Precip12hMaxGte`): 12-hour rolling window maximum. Sensitive to window boundaries. Short-term spike driven.

**Daily Total** (`PrecipSumGte`): 24-hour cumulative rainfall. Multiple snapshots aggregated. Vulnerable to ordering and missing data. Cumulative (integration) risk.

**Hourly Intensity** (`Precip1hGte`): 1-hour max rainfall intensity. Highest temporal resolution, most sensitive to single-hour data gaps.

### Weather Gate (4 products)

**Heat Protection** (`TempMaxGte`): 7-day max temperature `>=` threshold. **Frost Protection** (`TempMinLte`): 7-day min temperature `<=` threshold (inverted operator). **Wind Damage** (`WindGustMaxGte`): 7-day max wind gust. **Precipitation Type** (`PrecipTypeOccurred`): Bitmask match (snow=2, ice=4, snow-or-ice=6).

### Climate Parametrics (7 products)

**Heat Index** (`HeatIndexMaxGte`), **Snow Accumulation** (`SnowDepthMaxGte`), **Pressure Drop** (`PressureDropMaxGte`), **Low Sunshine** (`SunshineDurationSumLte` — inverted operator), **Flood** (`RiverDischargeMaxGte`), **Marine Storm** (`WaveHeightMaxGte`), **Air Quality** (`Pm25MaxGte`).

### Cross-product considerations
- All 14 products share oracle infrastructure (Open-Meteo, Supabase)
- All use Pricing API for pricing validation
- Different aggregation logic per product (rolling max, cumulative sum, min tracking, bitmask match)
- Two inverted operators (`<=`): TempMinLte, SunshineDurationSumLte
- **Highest risk: incorrect product routing or evaluation**

---

## 4. Mandatory Test Declarations

Every test case must explicitly declare:
- **target product(s):** one or more of the 14 active products (e.g., Storm Protection, Daily Total, Frost Protection, etc.) or "all"
- **time model:** rolling-window / cumulative / min-tracking / bitmask / other
- whether **product routing** is involved
- whether **migration or upgrade** is involved

> Undeclared tests are forbidden.

---

## 5. Storm Protection Rules (`Precip12hMaxGte`)

### Primary failure modes:
- 12h window boundary inclusion/exclusion
- delayed data causing missed trigger within window
- noise-driven false positives in high-frequency data

### Mandatory tests:
- exact 12h window boundary tests
- delayed oracle delivery where trigger should have fired
- duplicated oracle reports (idempotency)
- cross-batch window persistence (13-hour history)

> **Storm Protection is most fragile at window boundaries.**

---

## 6. Daily Total Rules (`PrecipSumGte`)

### Primary failure modes:
- missing snapshots within 24h period
- out-of-order snapshots
- duplicate aggregation
- cumulative error amplification

### Mandatory rules:
- cumulative values must be defined by a **set of observations**, not arrival order
- if arrival order matters, invalid ordering must be **explicitly rejected**

### Mandatory P0 tests:
- missing intermediate observation
- missing final observation
- reversed snapshot order
- duplicated timestamps
- Early Trigger followed by maturity settlement (no double payout)
- duration boundaries (minimum and maximum)
- abnormal values and cumulative amplification

> **Daily Total is most fragile with ordering and missing data.**

---

## 7. Cross-Product and Infrastructure Rules

Every infrastructure test must declare at least one tag:

| Tag | Description |
|-----|-------------|
| **A** | product coexistence (both evaluated from same oracle data) |
| **B** | oracle infrastructure (Open-Meteo, Supabase, OCW) |
| **C** | DAO auto-underwrite (Pricing API pricing validation) |
| **D** | cross-validation (ERA5 + NASA POWER audit) |

> Untagged infrastructure tests are forbidden.

---

### 7.1 Product Routing is P0

**Mandatory tests:**
- Each of the 14 event types routes to its correct evaluator (e.g., `Precip12hMaxGte` → rolling-window, `PrecipSumGte` → cumulative, `TempMinLte` → min-tracking)
- Inverted operators (`<=`) for `TempMinLte` and `SunshineDurationSumLte` are correctly applied
- `PrecipTypeOccurred` bitmask matching works for all mask values (2, 4, 6)
- Unknown event types are rejected
- DAO whitelist filtering works per product type (all 14 event types, all 40 cities)

---

### 7.2 Migration and Upgrade Safety (P0)

**Mandatory tests:**
- migration is idempotent (0, 1, N executions converge)
- upgrade interruption and recovery
- active policies complete after upgrade
- total assets, reserves, and policy counts are preserved

> Migration correctness is part of the **specification**, not an implementation detail.

---

### 7.3 Off-chain Consistency (P0)

**Mandatory tests:**
- recovery from partial success
- at-least-once delivery convergence
- replay safety
- oracle or node restarts do not create duplicate monitoring
- Supabase write idempotency (deterministic composite keys)

---

### 7.4 Operational and Governance Controls (If Introduced)

If pause, emergency stop, or risk limits exist, they are **P0**.

**Mandatory tests:**
- behavior during pause
- settlement behavior for triggers occurring before pause
- authority boundaries
- behavior when risk limits are reached

> **Operational features must be attacked first, not last.**

---

## 8. Mandatory Test Design Template

Every test case must include:

| Field | Description |
|-------|-------------|
| **Test name** | Unique identifier for the test |
| **Target product** | One or more of 14 active products, or "all" |
| **Classification** | A-E (from Section 2) |
| **Expected failure mode** | What the test attempts to break |
| **Attacker perspective** | How a malicious actor would exploit this |
| **Expected defense** | How the system should prevent exploitation |
| **Success criteria** | Clear pass/fail conditions |
| **Impact if broken** | funds / trust / legal |

---

## 9. Coverage Definition (Numeric Coverage Forbidden)

> Do not rely on percentage code coverage.

**Accepted coverage metrics:**
- state transition coverage
- economic scenario coverage
- oracle failure scenario count
- migration completion scenarios
- cross-product arbitrage scenarios

---

## 10. AI Agent Self-Audit Checklist

Before finalizing a test suite, the agent must verify:

- [ ] Can a malicious human profit if this fails?
- [ ] Will this survive future specification changes?
- [ ] Can I clearly explain why this test exists?

> Tests that fail this self-audit must be **removed**.

---

## 11. Failure Severity Classification

| Severity | Description | Action |
|----------|-------------|--------|
| **P0** | fund loss, double payout, product routing failure | immediate halt |
| **P1** | unfairness, broken insurance experience | fix before redeploy |
| **P2** | UI, logs, non-critical observability | backlog |

---

**End of Rulebook**
