# Open-Meteo Event Expansion Roadmap (V4)

> **Status**: Implemented snapshot (updated 2026-03-31)
> **Purpose**: Track E2E implementation depth by event type.

## Verified Current State (2026-03-31)

| Layer | Current State |
|---|---|
| On-chain event types (`EventTypeV4`) | 14 variants (3 removed from enum: UvIndexMaxGte, SoilMoistureMinLte, VisibilityMinLte) |
| Frontend buyable products | 14 (active catalog products) — unified in `/climate-parametrics` (Protection Terminal) |
| OCW fetch coverage | Core hourly + specialty endpoints (air-quality/marine/flood) |
| Ingest observation fields | Core + extended fields for all event families |
| Timeline evaluation `current_value` | All `AggStateV4` variants parsed |
| Weather chart support | All 14 active event types rendered |
| Cross-validation API accepted trigger types | All 14 active event types |

## E2E Event Matrix

Legend:
- `Full`: Purchase/monitor/settle + observability path implemented.
- `Internal`: Runtime/oracle supports it but not exposed as a sold product.
- `CV-Partial`: Cross-validation endpoint accepts it, but secondary source coverage is limited.

| Event Type | Product / Peril | E2E Status | Cross-Validation Secondary Sources |
|---|---|---|---|
| `PrecipSumGte` | `rainfall_7d_24h_total_binary` | Full | ERA5 + NASA |
| `Precip1hGte` | `rainfall_7d_1h_max_binary` | Full | ERA5 + NASA |
| `Precip12hMaxGte` | `rainfall_7d_12h_accum_binary` | Full | ERA5 + NASA |
| `TempMaxGte` | `temp_max_7d_binary` | Full | ERA5 |
| `TempMinLte` | `temp_min_7d_binary` | Full | ERA5 |
| `WindGustMaxGte` | `wind_gust_max_7d_binary` | Full | ERA5 |
| `PrecipTypeOccurred` | `precip_type_7d_binary` | Full | CV-Partial (no compatible secondary source yet) |
| ~~`UvIndexMaxGte`~~ | ~~`uv_index_max_7d_binary`~~ | **Removed** (from enum and catalog — ERA5 null) | N/A |
| `HeatIndexMaxGte` | `heat_index_max_7d_binary` | Full | CV-Partial (no compatible secondary source yet) |
| ~~`SoilMoistureMinLte`~~ | ~~`soil_moisture_min_7d_binary`~~ | **Removed** (from enum and catalog — ERA5 null) | N/A |
| `SnowDepthMaxGte` | `snow_depth_max_7d_binary` | Full | CV-Partial (no compatible secondary source yet) |
| ~~`VisibilityMinLte`~~ | ~~`visibility_min_7d_binary`~~ | **Removed** (from enum and catalog — ERA5 null) | N/A |
| `PressureDropMaxGte` | `pressure_drop_max_7d_binary` | Full | CV-Partial (no compatible secondary source yet) |
| `SunshineDurationSumLte` | `sunshine_duration_7d_binary` | Full | CV-Partial (no compatible secondary source yet) |
| `RiverDischargeMaxGte` | `river_discharge_max_7d_binary` | Full | CV-Partial (no compatible secondary source yet) |
| `WaveHeightMaxGte` | `wave_height_max_7d_binary` | Full | CV-Partial (no compatible secondary source yet) |
| `Pm25MaxGte` | `pm25_max_7d_binary` | Full | CV-Partial (no compatible secondary source yet) |

## Cross-Validation API (Current Contract)

`POST /api/policies/:policyId/cross-validate`

Request fields:
- `latitude`
- `longitude`
- `coverage_start`
- `coverage_end`
- `trigger_type` (all 14 active types supported)
- `threshold_value` (new standard)
- `threshold_mm` (backward-compatible alias)

Response behavior:
- Always computes primary metric/trigger decision from stored observations.
- Uses secondary sources where compatible (`ERA5`, `NASA POWER`).
- Returns `confidence=insufficient_data` when no compatible secondary source exists.

## Runtime Activation Note

This repository now includes extended observation payload fields and observability parsing.
If the running chain runtime is older, deploy/runtime-upgrade is still required for the new OCW payload shape to be emitted on-chain.

## References

- `primitives/src/lib.rs`
- `pallets/pallet-oracle-v4/src/http_client.rs`
- `oracle-service/src/api/routes.ts`
- `oracle-service/src/chain/listener.ts`
- `oracle-service/src/weather/cross-validator.ts`
- `frontend/src/lib/product-specs.ts` (single source of truth for product config)
- `frontend/src/components/features/WeatherHistoryChart.tsx`
- `frontend/src/lib/product-lineup.ts`
