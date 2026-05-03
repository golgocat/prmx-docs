# PRMX V4 Product Catalog (14 Products)

> **Scope**: Canonical product reference for all products currently active in the V4 pricing catalog.

This document tracks products returned by:

- `GET /v4/catalog/info/live`
- API base: `https://neuralgcm-pricing-api-pm6goneopq-as.a.run.app`

Active catalog: **`obs-openmeteo-14prod-2026-03-12`** — 14 products, 40 cities, 638,352 entries, 31yr ERA5 reanalysis.

---

## Shared specs

- Product count: **14 active products** (3 removed: UV Index, Drought, Low Visibility — ERA5 returns null data)
- Geography: **40 supported regions/cities**
- Observation window: **168 hours (7 days)** for most products; **24 hours** for PrecipSumGte (Daily Total)
- Payout shape: **binary**
- Thresholds: discrete bucket values per product
- Quote API field is named `threshold_mm` for compatibility, but unit varies by product

---

## Product lines

PRMX product naming is organized into 3 lines:

| Product line | Scope |
|---|---|
| **PRMX Rain Guard** | Core rainfall products (3) |
| **PRMX Weather Gate** | General weather-trigger products (4) |
| **PRMX Climate Parametrics** | Catalog-first expansion products (7) |

Canonical taxonomy: `docs/product/PRODUCT-LINEUP.md`.

---

## Activation matrix

| Layer | Current scope |
|------|----------------|
| Pricing catalog (`/v4/catalog/info/live`) | 14 products (3 removed due to null ERA5 data) |
| Frontend `/climate-parametrics` (Protection Terminal) | 14 products (unified purchase flow) |
| On-chain `EventTypeV4` coverage | 14 enum variants (3 removed from enum: indices 7, 9, 11; remaining indices codec-pinned) |
| DAO default whitelist | 14 event types (all active products) |

Interpretation:

- **Catalog-active** means the pricing API can return quotes for the product.
- **Purchasable in app** additionally requires event-type wiring in frontend + on-chain + DAO whitelist/config.

---

## Tier definitions

| Tier | Meaning |
|------|---------|
| `tier_1` | Core Rain Guard / Weather Gate products with established UX and operator playbooks |
| `tier_2` | Climate Parametrics expansion products with moderate operational complexity |
| `tier_3` | Climate Parametrics specialty products with higher data/model complexity |

---

## Product specifications

| # | Display name | Product line | Peril ID | Tier | Threshold buckets | Trigger condition | Current activation |
|---|--------------|--------------|----------|------|-------------------|------------------|-------------------|
| 1 | Storm Protection (7-Day) | **PRMX Rain Guard** | `rainfall_7d_12h_accum_binary` | `tier_1` | 20, 30, 40, 50, 75, 100 mm | Any 12h rolling precipitation max within 7d `>= threshold` | App + chain + catalog |
| 2 | Daily Total | **PRMX Rain Guard** | `rainfall_7d_24h_total_binary` | `tier_1` | 20, 30, 40, 50, 75, 100 mm | Daily precipitation total within 24h `>= threshold` | App + chain + catalog |
| 3 | Hourly Intensity (7-Day) | **PRMX Rain Guard** | `rainfall_7d_1h_max_binary` | `tier_1` | 5, 10, 15, 20, 30 mm/h | 1-hour max rainfall intensity within 7d `>= threshold` | App + chain + catalog |
| 4 | Heat Protection (7-Day) | **PRMX Weather Gate** | `temp_max_7d_binary` | `tier_1` | 35, 38, 40, 42, 45 °C | Maximum temperature within 7d `>= threshold` | App + chain + catalog |
| 5 | Frost Protection (7-Day) | **PRMX Weather Gate** | `temp_min_7d_binary` | `tier_1` | 0, -3, -5, -10 °C | Minimum temperature within 7d `<= threshold` | App + chain + catalog |
| 6 | Wind Damage Protection (7-Day) | **PRMX Weather Gate** | `wind_gust_max_7d_binary` | `tier_1` | 15, 20, 25, 30 m/s | Maximum wind gust within 7d `>= threshold` | App + chain + catalog |
| 7 | Precipitation Type (7-Day) | **PRMX Weather Gate** | `precip_type_7d_binary` | `tier_1` | 2, 4, 6 mask | Precipitation type bitmask match occurs | App + chain + catalog |
| 8 | Heat Index Protection (7-Day) | **PRMX Climate Parametrics** | `heat_index_max_7d_binary` | `tier_2` | 35, 38, 40, 42, 45 °C | Heat index max within 7d `>= threshold`* | App + chain + catalog |
| ~~8~~ | ~~UV Index Protection (7-Day)~~ | ~~**PRMX Weather Gate**~~ | ~~`uv_index_max_7d_binary`~~ | | | | **REMOVED** — ERA5 returns null for `uv_index` |
| ~~9~~ | ~~Drought Protection (7-Day)~~ | ~~**PRMX Climate Parametrics**~~ | ~~`soil_moisture_min_7d_binary`~~ | | | | **REMOVED** — ERA5 returns null for `soil_moisture_0_to_10cm` |
| 9 | Snow Accumulation Protection (7-Day) | **PRMX Climate Parametrics** | `snow_depth_max_7d_binary` | `tier_2` | 50, 100, 200, 500 mm | Snow depth max within 7d `>= threshold`* | App + chain + catalog |
| ~~11~~ | ~~Low Visibility Protection (7-Day)~~ | ~~**PRMX Climate Parametrics**~~ | ~~`visibility_min_7d_binary`~~ | | | | **REMOVED** — ERA5 returns null for `visibility` |
| 10 | Severe Weather Protection (7-Day) | **PRMX Climate Parametrics** | `pressure_drop_max_7d_binary` | `tier_2` | 5, 10, 15, 20 hPa | Pressure-drop max within 7d `>= threshold`* | App + chain + catalog |
| 11 | Low Sunshine Protection (7-Day) | **PRMX Climate Parametrics** | `sunshine_duration_7d_binary` | `tier_2` | 7200, 14400, 21600, 28800 s/day | Sunshine-duration sum condition (low-sun trigger) `<= threshold`* | App + chain + catalog |
| 12 | Flood Protection (7-Day) | **PRMX Climate Parametrics** | `river_discharge_max_7d_binary` | `tier_3` | 500, 1000, 2000, 5000 m³/s | River discharge max within 7d `>= threshold`* | App + chain + catalog |
| 13 | Marine Storm Protection (7-Day) | **PRMX Climate Parametrics** | `wave_height_max_7d_binary` | `tier_3` | 2, 3, 4, 6 m | Wave height max within 7d `>= threshold`* | App + chain + catalog |
| 14 | Air Quality Protection (7-Day) | **PRMX Climate Parametrics** | `pm25_max_7d_binary` | `tier_3` | 35, 50, 100, 150 μg/m³ | PM2.5 max within 7d `>= threshold`* | App + chain + catalog |

`*` For tier 2/3 products, comparator wording is inferred from `peril_id` naming and display labels because `catalog/info/live` does not expose a formal comparator field.

---

## Event type mapping (runtime)

Current `EventTypeV4` enum directly maps to these catalog products:

| Event type | Product(s) |
|------------|------------|
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

`UvIndexMaxGte` (index 7), `SoilMoistureMinLte` (index 9), and `VisibilityMinLte` (index 11) have been **removed from the `EventTypeV4` enum**. Their codec indices are permanently skipped to preserve SCALE compatibility for the remaining 14 variants.

---

## Practical activation checklist for future products

To move a future catalog product into purchasable UX:

1. Add/confirm runtime `EventTypeV4` + `AggStateV4` coverage for that peril.
2. Implement OCW fetch + aggregation logic for required weather field(s).
3. Map peril-to-event type in frontend purchase flow (`frontend/src/lib/product-specs.ts` — single source of truth).
4. Extend pricing proxy whitelist (`frontend/src/app/api/pricing/route.ts`).
5. Add product to DAO event-type whitelist and validation path.
6. Add end-to-end lifecycle tests (request -> monitoring -> settlement).

---

## Notes

- This catalog doc supersedes older “2-product only” references for pricing-catalog coverage.
- Rainfall-only semantics are defined in `docs/product/RAINGUARD-PRODUCT-SPEC.md`.
- Product-line taxonomy is defined in `docs/product/PRODUCT-LINEUP.md`.
