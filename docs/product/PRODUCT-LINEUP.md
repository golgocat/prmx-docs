# PRMX Product Lineup

> **Status**: Current (updated 2026-03-31)
> **Purpose**: Canonical product-line taxonomy for PRMX V4.

## Line Architecture

| Line | Mission | Current Scope |
|---|---|---|
| **PRMX Rain Guard** | Core rainfall parametric protection with clear rain-trigger semantics | 3 products (Storm Protection, Daily Total, Hourly Intensity) |
| **PRMX Weather Gate** | General weather-trigger protection beyond rainfall | 4 products (Heat, Frost, Wind, Precipitation Type) |
| **PRMX Climate Parametrics** | Expanded climate and specialty parametrics from catalog-active perils | 7 products |

## Portfolio Matrix (14 Active Catalog Products)

| Product | Peril ID | Event Type | Product Line | Activation |
|---|---|---|---|---|
| Storm Protection (7-Day) | `rainfall_7d_12h_accum_binary` | `Precip12hMaxGte` | **PRMX Rain Guard** | App + chain + catalog |
| Daily Total (7-Day) | `rainfall_7d_24h_total_binary` | `PrecipSumGte` | **PRMX Rain Guard** | App + chain + catalog |
| Hourly Intensity (7-Day) | `rainfall_7d_1h_max_binary` | `Precip1hGte` | **PRMX Rain Guard** | App + chain + catalog |
| Heat Protection (7-Day) | `temp_max_7d_binary` | `TempMaxGte` | **PRMX Weather Gate** | App + chain + catalog |
| Frost Protection (7-Day) | `temp_min_7d_binary` | `TempMinLte` | **PRMX Weather Gate** | App + chain + catalog |
| Wind Damage Protection (7-Day) | `wind_gust_max_7d_binary` | `WindGustMaxGte` | **PRMX Weather Gate** | App + chain + catalog |
| Precipitation Type (7-Day) | `precip_type_7d_binary` | `PrecipTypeOccurred` | **PRMX Weather Gate** | App + chain + catalog |
| Heat Index Protection (7-Day) | `heat_index_max_7d_binary` | `HeatIndexMaxGte` | **PRMX Climate Parametrics** | App + chain + catalog |
| Snow Accumulation Protection (7-Day) | `snow_depth_max_7d_binary` | `SnowDepthMaxGte` | **PRMX Climate Parametrics** | App + chain + catalog |
| Severe Weather Protection (7-Day) | `pressure_drop_max_7d_binary` | `PressureDropMaxGte` | **PRMX Climate Parametrics** | App + chain + catalog |
| Low Sunshine Protection (7-Day) | `sunshine_duration_7d_binary` | `SunshineDurationSumLte` | **PRMX Climate Parametrics** | App + chain + catalog |
| Flood Protection (7-Day) | `river_discharge_max_7d_binary` | `RiverDischargeMaxGte` | **PRMX Climate Parametrics** | App + chain + catalog |
| Marine Storm Protection (7-Day) | `wave_height_max_7d_binary` | `WaveHeightMaxGte` | **PRMX Climate Parametrics** | App + chain + catalog |
| Air Quality Protection (7-Day) | `pm25_max_7d_binary` | `Pm25MaxGte` | **PRMX Climate Parametrics** | App + chain + catalog |

## UX and Routing Strategy

- **Primary purchase route**: `/climate-parametrics` ("Protection Terminal" in sidebar). This unified page exposes all 14 products grouped by product line.
- **Legacy product routes**: `/rainguard` and `/weather-gate` still exist as standalone purchase pages with their own product-line scoped tabs.
- **Landing pages**: `/products` hub links to `/products/rain-guard`, `/products/weather-gate`, `/products/climate-parametrics` for marketing/detail pages.
- Present products grouped by line: Rain Guard first, Weather Gate second, Climate Parametrics third.
- All 14 active catalog products are purchasable from the app workspace.

## Activation Gates (Future Product Additions)

1. Runtime + OCW support exists for the peril's trigger logic.
2. Oracle snapshot computation supports the peril state shape.
3. Frontend mapping (`peril_id`, thresholds, units) is added.
4. Pricing proxy validation allowlist is updated.
5. End-to-end settlement tests pass.

## Related Docs

- `docs/product/RAINGUARD-PRODUCT-SPEC.md`
- `docs/product/V4-APP-DESIGN.md`
- `docs/product/V4-PRODUCT-CATALOG.md`
- `docs/product/OPEN-METEO-EVENT-EXPANSION.md`
