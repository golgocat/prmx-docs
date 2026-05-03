# PRMX Rain Guard Product Spec

> **Status**: Current (updated 2026-03-31)
> **Scope**: Core rainfall line in the PRMX portfolio.

## Position in Portfolio

PRMX V4 now uses a 3-line structure:

- **PRMX Rain Guard**: rainfall products (this document)
- **PRMX Weather Gate**: non-rain weather products
- **PRMX Climate Parametrics**: catalog-first specialty products

Canonical taxonomy: `docs/product/PRODUCT-LINEUP.md`.

## Rain Guard Product Matrix

| Product | Event Type | Peril ID | Coverage | Thresholds |
|---|---|---|---|---|
| Storm Protection | `Precip12hMaxGte` | `rainfall_7d_12h_accum_binary` | Fixed 7 days | 20, 30, 40, 50, 75, 100 mm |
| Daily Total | `PrecipSumGte` | `rainfall_7d_24h_total_binary` | Fixed 24h | 20, 30, 40, 50, 75, 100 mm |
| Hourly Intensity | `Precip1hGte` | `rainfall_7d_1h_max_binary` | Fixed 7 days | 5, 10, 15, 20, 30 mm/h |

Rain Guard products are purchasable from both `/rainguard` (legacy standalone page) and `/climate-parametrics` (unified Protection Terminal).

## Product Semantics

### Storm Protection (`Precip12hMaxGte`)

- Trigger condition: any rolling 12-hour precipitation sum meets/exceeds threshold.
- Coverage window: fixed 7 days.
- Aggregation state: `AggStateV4::Precip12hMax { max_12h_mm_x1000 }`.

### Daily Total (`PrecipSumGte`)

- Trigger condition: cumulative precipitation over policy period meets/exceeds threshold.
- Coverage window: fixed 24 hours.
- Aggregation state: `AggStateV4::PrecipSum { sum_mm_x1000 }`.

### Hourly Intensity (`Precip1hGte`)

- Trigger condition: 1-hour max rainfall intensity meets/exceeds threshold.
- Coverage window: fixed 7 days.
- Aggregation state: `AggStateV4::Precip1hMax { max_1h_mm_x1000 }`.

### Early Trigger

All three Rain Guard products support early trigger behavior and may settle before coverage end once threshold conditions are reached.

## Units and Encoding

- Rainfall thresholds use `MmX1000` (`50 mm` -> `50000`).
- Frontend quote path uses `/api/pricing` with `peril_id`, `threshold`, `coverage`, `duration_in_hours`.
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

## References

- `docs/product/PRODUCT-LINEUP.md`
- `docs/product/V4-APP-DESIGN.md`
- `docs/product/V4-PRODUCT-CATALOG.md`
