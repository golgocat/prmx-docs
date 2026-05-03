# PRMX V4 Product Catalog (14 Products)

> **Scope**: All products currently active in the V4 pricing catalog.

| Field | Value |
|---|---|
| Active catalog | `obs-openmeteo-14prod-2026-03-12` |
| Products | 14 |
| Cities | 40 |
| Catalog entries | 638,352 |
| Reanalysis source | 31-year ERA5 |
| Catalog endpoint | `GET /v4/catalog/info/live` |
| API base | `https://neuralgcm-pricing-api-pm6goneopq-as.a.run.app` |

---

## Shared specs

- **Geography** — 40 supported regions/cities.
- **Observation window** — 168 h (7 days) for most products; 24 h for `PrecipSumGte` (Daily Total).
- **Payout shape** — binary.
- **Thresholds** — discrete bucket values per product.
- **Quote API field** — `threshold_mm` (named for backward compatibility; unit varies by product).

---

## Product lines

| Product line | Scope |
|---|---|
| **PRMX Rain Guard** | Core rainfall products (3) |
| **PRMX Weather Gate** | General weather-trigger products (4) |
| **PRMX Climate Parametrics** | Catalog-first expansion products (7) |

Canonical taxonomy: [Product Lineup](/docs/product/PRODUCT-LINEUP).

---

## Activation matrix

| Layer | Scope |
|------|----------------|
| Pricing catalog (`/v4/catalog/info/live`) | 14 products |
| Frontend `/climate-parametrics` (Protection Terminal) | 14 products (unified purchase flow) |
| On-chain `EventTypeV4` enum | 14 variants |
| DAO default whitelist | 14 event types |

A product is **purchasable in app** when it has all four: pricing entry, frontend mapping, on-chain event type, and DAO whitelist coverage.

---

## Tier definitions

| Tier | Meaning |
|------|---------|
| `tier_1` | Core Rain Guard / Weather Gate products with established UX and operator playbooks |
| `tier_2` | Climate Parametrics expansion products with moderate operational complexity |
| `tier_3` | Climate Parametrics specialty products with higher data/model complexity |

---

## Product specifications

| # | Display name | Product line | Peril ID | Tier | Threshold buckets | Trigger condition |
|---|---|---|---|---|---|---|
| 1 | Storm Protection (7-Day) | Rain Guard | `rainfall_7d_12h_accum_binary` | `tier_1` | 20, 30, 40, 50, 75, 100 mm | 12h rolling precipitation max within 7d `≥ threshold` |
| 2 | Daily Total | Rain Guard | `rainfall_7d_24h_total_binary` | `tier_1` | 20, 30, 40, 50, 75, 100 mm | Daily precipitation total within 24h `≥ threshold` |
| 3 | Hourly Intensity (7-Day) | Rain Guard | `rainfall_7d_1h_max_binary` | `tier_1` | 5, 10, 15, 20, 30 mm/h | 1-hour max rainfall intensity within 7d `≥ threshold` |
| 4 | Heat Protection (7-Day) | Weather Gate | `temp_max_7d_binary` | `tier_1` | 35, 38, 40, 42, 45 °C | Max temperature within 7d `≥ threshold` |
| 5 | Frost Protection (7-Day) | Weather Gate | `temp_min_7d_binary` | `tier_1` | 0, -3, -5, -10 °C | Min temperature within 7d `≤ threshold` |
| 6 | Wind Damage Protection (7-Day) | Weather Gate | `wind_gust_max_7d_binary` | `tier_1` | 15, 20, 25, 30 m/s | Max wind gust within 7d `≥ threshold` |
| 7 | Precipitation Type (7-Day) | Weather Gate | `precip_type_7d_binary` | `tier_1` | 2, 4, 6 mask | Precipitation-type bitmask match occurs |
| 8 | Heat Index Protection (7-Day) | Climate Parametrics | `heat_index_max_7d_binary` | `tier_2` | 35, 38, 40, 42, 45 °C | Heat-index max within 7d `≥ threshold`* |
| 9 | Snow Accumulation Protection (7-Day) | Climate Parametrics | `snow_depth_max_7d_binary` | `tier_2` | 50, 100, 200, 500 mm | Snow depth max within 7d `≥ threshold`* |
| 10 | Severe Weather Protection (7-Day) | Climate Parametrics | `pressure_drop_max_7d_binary` | `tier_2` | 5, 10, 15, 20 hPa | Pressure-drop max within 7d `≥ threshold`* |
| 11 | Low Sunshine Protection (7-Day) | Climate Parametrics | `sunshine_duration_7d_binary` | `tier_2` | 7,200, 14,400, 21,600, 28,800 s/day | Sunshine-duration sum (low-sun trigger) `≤ threshold`* |
| 12 | Flood Protection (7-Day) | Climate Parametrics | `river_discharge_max_7d_binary` | `tier_3` | 500, 1,000, 2,000, 5,000 m³/s | River discharge max within 7d `≥ threshold`* |
| 13 | Marine Storm Protection (7-Day) | Climate Parametrics | `wave_height_max_7d_binary` | `tier_3` | 2, 3, 4, 6 m | Wave height max within 7d `≥ threshold`* |
| 14 | Air Quality Protection (7-Day) | Climate Parametrics | `pm25_max_7d_binary` | `tier_3` | 35, 50, 100, 150 μg/m³ | PM2.5 max within 7d `≥ threshold`* |

All 14 are **App + chain + catalog** active.

`*` For tier 2/3 products, comparator wording is inferred from `peril_id` naming and display labels — `catalog/info/live` does not expose a formal comparator field.

---

## Event type mapping (runtime)

Each catalog product maps directly to one `EventTypeV4` variant:

| Event type | Product |
|---|---|
| `Precip12hMaxGte` | Storm Protection |
| `PrecipSumGte` | Daily Total |
| `Precip1hGte` | Hourly Intensity |
| `TempMaxGte` | Heat Protection |
| `TempMinLte` | Frost Protection |
| `WindGustMaxGte` | Wind Damage Protection |
| `PrecipTypeOccurred` | Precipitation Type |
| `HeatIndexMaxGte` | Heat Index Protection |
| `SnowDepthMaxGte` | Snow Accumulation Protection |
| `PressureDropMaxGte` | Severe Weather Protection |
| `SunshineDurationSumLte` | Low Sunshine Protection |
| `RiverDischargeMaxGte` | Flood Protection |
| `WaveHeightMaxGte` | Marine Storm Protection |
| `Pm25MaxGte` | Air Quality Protection |

SCALE codec indices `7`, `9`, and `11` are reserved (skipped) to preserve compatibility for the 14 active variants.

---

## Adding a new product

1. Add runtime `EventTypeV4` + `AggStateV4` coverage for the peril.
2. Implement OCW fetch + aggregation for the required weather fields.
3. Add a peril-to-event-type mapping in `frontend/src/lib/product-specs.ts` (SSoT).
4. Extend the pricing proxy allowlist in `frontend/src/app/api/pricing/route.ts`.
5. Add the product to the DAO event-type whitelist and validation path.
6. Add an end-to-end lifecycle test (request → monitoring → settlement).

---

## Related

- Rainfall-only semantics — [Rainguard Product Spec](/docs/product/RAINGUARD-PRODUCT-SPEC).
- Product-line taxonomy — [Product Lineup](/docs/product/PRODUCT-LINEUP).
