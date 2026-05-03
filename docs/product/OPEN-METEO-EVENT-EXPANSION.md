# Open-Meteo Event Expansion Roadmap (V4)

> **Purpose**: Track end-to-end implementation depth by event type.

## Coverage at a glance

| Layer | Coverage |
|---|---|
| On-chain `EventTypeV4` | 14 variants |
| Frontend purchasable products | 14 — unified in `/climate-parametrics` (Protection Terminal) |
| OCW fetch | Core hourly + specialty endpoints (air-quality / marine / flood) |
| Ingest observation fields | All event families |
| Timeline `current_value` | All `AggStateV4` variants parsed |
| Weather chart | All 14 event types rendered |
| Cross-validation API | All 14 trigger types accepted |

## E2E status by event type

| Status | Meaning |
|---|---|
| **Full** | Purchase / monitor / settle + observability path implemented |
| **CV-Partial** | Cross-validation endpoint accepts it, but no compatible secondary source yet |

| Event Type | Peril ID | Status | Secondary sources |
|---|---|---|---|
| `PrecipSumGte` | `rainfall_7d_24h_total_binary` | Full | ERA5, NASA |
| `Precip1hGte` | `rainfall_7d_1h_max_binary` | Full | ERA5, NASA |
| `Precip12hMaxGte` | `rainfall_7d_12h_accum_binary` | Full | ERA5, NASA |
| `TempMaxGte` | `temp_max_7d_binary` | Full | ERA5 |
| `TempMinLte` | `temp_min_7d_binary` | Full | ERA5 |
| `WindGustMaxGte` | `wind_gust_max_7d_binary` | Full | ERA5 |
| `PrecipTypeOccurred` | `precip_type_7d_binary` | Full | CV-Partial |
| `HeatIndexMaxGte` | `heat_index_max_7d_binary` | Full | CV-Partial |
| `SnowDepthMaxGte` | `snow_depth_max_7d_binary` | Full | CV-Partial |
| `PressureDropMaxGte` | `pressure_drop_max_7d_binary` | Full | CV-Partial |
| `SunshineDurationSumLte` | `sunshine_duration_7d_binary` | Full | CV-Partial |
| `RiverDischargeMaxGte` | `river_discharge_max_7d_binary` | Full | CV-Partial |
| `WaveHeightMaxGte` | `wave_height_max_7d_binary` | Full | CV-Partial |
| `Pm25MaxGte` | `pm25_max_7d_binary` | Full | CV-Partial |

## Cross-validation API

`POST /api/policies/:policyId/cross-validate`

| Field | Notes |
|---|---|
| `latitude`, `longitude` | Required |
| `coverage_start`, `coverage_end` | Coverage window |
| `trigger_type` | Any of the 14 active `EventTypeV4` variants |
| `threshold_value` | Standard threshold field |
| `threshold_mm` | Backward-compatible alias |

Response always computes primary trigger from stored observations. Secondary sources (`ERA5`, `NASA POWER`) are used where compatible; otherwise `confidence=insufficient_data` is returned.

## Code references

- `primitives/src/lib.rs`
- `pallets/pallet-oracle-v4/src/http_client.rs`
- `oracle-service/src/api/routes.ts`
- `oracle-service/src/chain/listener.ts`
- `oracle-service/src/weather/cross-validator.ts`
- `frontend/src/lib/product-specs.ts` (SSoT for product config)
- `frontend/src/components/features/WeatherHistoryChart.tsx`
- `frontend/src/lib/product-lineup.ts`
