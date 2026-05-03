# PRMX Rain Guard Product Spec

> **Scope**: Core rainfall line in the PRMX portfolio. Sister lines are **Weather Gate** (non-rain weather) and **Climate Parametrics** (catalog-first specialty). Canonical taxonomy lives in [Product Lineup](/docs/product/PRODUCT-LINEUP).

## Product matrix

| Product | Event Type | Peril ID | Coverage | Thresholds |
|---|---|---|---|---|
| Storm Protection | `Precip12hMaxGte` | `rainfall_7d_12h_accum_binary` | 7 days | 20, 30, 40, 50, 75, 100 mm |
| Daily Total | `PrecipSumGte` | `rainfall_7d_24h_total_binary` | 24 h | 20, 30, 40, 50, 75, 100 mm |
| Hourly Intensity | `Precip1hGte` | `rainfall_7d_1h_max_binary` | 7 days | 5, 10, 15, 20, 30 mm/h |

Purchase routes: `/rainguard` (standalone) and `/climate-parametrics` (unified Protection Terminal).

## Product semantics

| Product | Trigger condition | Aggregation state |
|---|---|---|
| Storm Protection | Any rolling 12-hour precipitation sum meets/exceeds threshold | `AggStateV4::Precip12hMax { max_12h_mm_x1000 }` |
| Daily Total | Cumulative 24-hour precipitation meets/exceeds threshold | `AggStateV4::PrecipSum { sum_mm_x1000 }` |
| Hourly Intensity | 1-hour max rainfall intensity meets/exceeds threshold | `AggStateV4::Precip1hMax { max_1h_mm_x1000 }` |

**Early trigger** — all three products may settle before coverage end once threshold conditions are reached.

## Units and encoding

- Rainfall thresholds use `MmX1000` (`50 mm → 50000`).
- Frontend quote path: `/api/pricing` with `peril_id`, `threshold`, `coverage`, `duration_in_hours`.
- Proxy validation currently allows durations `24` and `168` hours.

## Request Examples

### Storm Protection

```javascript
{
  eventSpec: {
    eventType: 'Precip12hMaxGte',
    threshold: { value: 75000, unit: 'MmX1000' },
    earlyTrigger: true
  },
  coverageStart: timestamp,
  coverageEnd: timestamp + 7 * 86400
}
```

### Daily Total

```javascript
{
  eventSpec: {
    eventType: 'PrecipSumGte',
    threshold: { value: 50000, unit: 'MmX1000' },
    earlyTrigger: true
  },
  coverageStart: timestamp,
  coverageEnd: timestamp + 86400
}
```

## Related

- [Product Lineup](/docs/product/PRODUCT-LINEUP)
- [V4 App Design](/docs/product/V4-APP-DESIGN)
- [V4 Product Catalog](/docs/product/V4-PRODUCT-CATALOG)
