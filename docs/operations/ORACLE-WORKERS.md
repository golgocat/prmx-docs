# Oracle Service Worker / Automation Surface

## Overview

The oracle service can run in two process modes:

- **`MODE=monolith`** (default): one API process owns the capital watchers,
  reporters, monitors, and operator endpoints.
- **`MODE=workers`**: selected capital domains run as independent processes with
  Supabase-backed leader election.

Current Route 2 operations are Hyperlane-native. `VAULT_REPORTER_TRANSPORT=hyperlane`
is the live posture; `both` is a rollback / next-generation soak mode after a
runtime or `YieldReportRecipient` redeploy, not the steady-state target.
Codex automations supplement the service with read-only monitors; they should
report deltas and first suspected blockers, not mutate live capital state.

For the role-level architecture and cross-chain responsibility boundaries, see
`docs/architecture/V4-ORACLE-ARCHITECTURE.md`.

## Worker Processes

| Worker | `WORKER_NAME` | Domain | Health Port | Metrics Port |
|--------|---------------|--------|-------------|--------------|
| Deposit Worker | `deposit` | Reconciles pending wallet links and recent Base -> PRMX Hyperlane deposits into `pallet-assets(1)` | 8081 | 9091 |
| Exit Worker | `exit` | Releases canonical `warpAccount.request_exit` exits on Base and finalizes them on PRMX | 8082 | 9092 |
| Settlement Worker | `settlement` | Finalizes PRMX settlements after Base reserve-return evidence is available | 8083 | 9093 |
| Yield Reporter | `yield-reporter` | Runs vault asset reporting; live Route 2 reports go through Base `YieldReporter` -> Hyperlane -> PRMX `0x0802` | 8085 | 9095 |

## Runtime Automation

These surfaces are usually owned by the monolith API process, even when the
older worker split is available.

| Surface | Purpose | Primary env | Operator / readback |
|---------|---------|-------------|---------------------|
| Vault reporter | Periodic and on-demand PolicyVault `totalAssets()` reporting | `VAULT_REPORTER_ENABLED`, `VAULT_REPORTER_TRANSPORT=hyperlane`, `VAULT_REPORTER_ROUTE_ID=2`, `VAULT_REPORTER_YIELD_REPORTER_ADDRESS`, `VAULT_REPORTER_POLICY_VAULT_MANAGER_ADDRESS` | `GET /capital/vault/status`, `POST /capital/operator/run-vault-report` |
| Yield accrual driver | ICA yield command driver for testnet strategy accrual | `YIELD_ACCRUAL_DRIVER_ENABLED`, `YIELD_ACCRUAL_PROTOCOL`, `YIELD_ACCRUAL_INTERVAL_MS` | `GET /capital/yield-accrual/status`, `POST /capital/operator/run-yield-accrual` |
| Morpho borrower driver | Keeps the testnet Morpho market utilized so vaults can observe borrow interest | `MORPHO_BORROWER_DRIVER_ENABLED`, `MORPHO_BLUE_ADDRESS`, `MORPHO_MARKET_ID`, `MORPHO_BORROWER_*` | `GET /capital/morpho-borrower/status`, `POST /capital/operator/run-morpho-borrower` |
| Morpho live NAV probe | Display-only Base RPC probe for policy `PolicyVault.totalAssets()` between canonical Hyperlane reports | `MORPHO_LIVE_NAV_PROBE_ENABLED`, `MORPHO_LIVE_NAV_PROBE_INTERVAL_MS`, `MORPHO_LIVE_NAV_PROBE_ROUTE_ID` | `GET /capital/morpho-live-nav-probe/status`, `GET /capital/policies/:policyId/live-nav` |
| Morpho vault migration | Moves eligible vaults to the Morpho strategy surface during an explicit migration window | `MORPHO_VAULT_MIGRATION_ENABLED`, `MORPHO_BLUE_STRATEGY`, `MORPHO_VAULT_MIGRATION_*` | `GET /capital/morpho-vault-migration/status`, `POST /capital/operator/run-morpho-vault-migration` |
| Rebalancer monitor / decision / executor | Classifies vaults, builds rebalance decisions, and submits only eligible drift through post-SBP-404 ICA dispatch | `REBALANCER_MONITOR_ENABLED`, `REBALANCER_DECISION_ENABLED`, `REBALANCER_EXECUTOR_ENABLED`, `REBALANCER_MONITOR_FUNDING_BASELINE_TOLERANCE_ATOMIC`, `ICA_DISPATCH_ENABLED`, `PRMX_ICA_*`, `BASE_ICA_*` | Hardened smoke `api-lifecycle-hyperlane`; executor logs must show ICA dispatch on production runs |
| Warp invariant monitor | Observes Route 2 bridge-net supply vs Base collateral and persists history for alerts | `WARP_INVARIANT_MONITOR_ENABLED`, `WARP_INVARIANT_EVM_RPC_URL`, `WARP_INVARIANT_COLLATERAL_ADDRESS`, `WARP_INVARIANT_USDC_ADDRESS` | `GET /capital/warp-invariant`, `warp_invariant_samples`, `oracle_warp_invariant_*` metrics, Grafana alert |

`POST /capital/operator/*` endpoints require `Authorization: Bearer
$CAPITAL_OPERATOR_TOKEN`. Keep the token only in the host secret store; never
paste it into docs, manifests, logs, or screenshots.

## Running Workers

### npm scripts (local dev)

```bash
npm run worker:deposit
npm run worker:exit
npm run worker:settlement
npm run worker:yield-reporter
```

### Docker Compose

```bash
cd oracle-service && npm run build
docker build -f Dockerfile.worker -t prmx-oracle-worker .
docker compose -f docker-compose.workers.yml up -d
```

## Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `MODE` | `monolith` | `monolith` or `workers` |
| `WORKER_NAME` | - | Required in worker mode: `deposit`, `exit`, `settlement`, `yield-reporter` |
| `WORKER_INSTANCE_ID` | hostname-random | Unique instance identifier for leader election |
| `LEADER_HEARTBEAT_MS` | 30000 | Leadership heartbeat interval |
| `LEADER_STALE_MS` | 90000 | Time before a stale leader is replaced |
| `HEALTH_PORT` | 8081 | Port for `/healthz` in worker mode |
| `METRICS_PORT` | 9090 | Port for Prometheus `/metrics` |
| `CAPITAL_OPERATOR_TOKEN` | unset | Required for operator-triggered capital endpoints |
| `VAULT_REPORTER_TRANSPORT` | `direct` | Use `hyperlane` in the current live Route 2 generation |
| `VAULT_REPORTER_REBALANCE_ACK_INTERVAL_MS` | 30000 | Scan cadence for Base rebalance ack evidence used by the funding/report guard |
| `VAULT_REPORTER_REBALANCE_ACK_LOOKBACK_BLOCKS` | 5000 | Bounded Base log lookback for rebalance ack evidence |
| `MORPHO_LIVE_NAV_PROBE_ENABLED` | false | Enables display-only Base RPC NAV probes into `policy_live_nav`; never a settlement or pricing input |
| `MORPHO_LIVE_NAV_PROBE_INTERVAL_MS` | 30000 | Minimum 5s; live display cadence for the Morpho NAV probe |
| `REBALANCER_MONITOR_FUNDING_BASELINE_TOLERANCE_ATOMIC` | 1000000 | 1 USDC tolerance for classifying recorded funding vs real drawdown |
| `CAPITAL_WATCHER_SETTLEMENT_LOOKBACK_BLOCKS` | 5000 | Bounded Base reserve-return log lookback for settlement finalization recovery |
| `SETTLEMENT_DISPATCH_EVENT_LOOKBACK_BLOCKS` | 5000 | Override for settlement dispatch event recovery |
| `WARP_INVARIANT_MONITOR_ENABLED` | false | Enables `/capital/warp-invariant`, `warp_invariant_samples`, and `oracle_warp_invariant_*` metrics |

See `oracle-service/PRODUCTION.md` for the grouped production env surface.

## Leader Election

In worker mode, each worker type has at most one active leader in Supabase
`worker_leadership`. Standby instances poll and take over after
`LEADER_STALE_MS`.

## Health Endpoint

```text
GET /healthz -> 200 healthy, 503 unhealthy
```

Returns 503 if tick lag exceeds 3x poll interval or leader changes exceed
5/hour.

## Prometheus Metrics

```text
GET /metrics -> text/plain
```

Core metrics:

- `oracle_worker_tick_duration_seconds{worker}`
- `oracle_worker_errors_total{worker}`
- `oracle_kind_dispatched_total{kind, result}`
- `oracle_relay_latency_seconds{direction}`
- `oracle_leader_changes_total{worker}`
- `oracle_pending_items{worker, type}`
- `oracle_warp_invariant_*`

## Operational Notes

- The vault reporter has a zero-baseline funding/report guard: when a vault is
  newly funded on Base but no rebalance ack has landed yet, the reporter should
  suppress false yield/funding reports instead of minting synthetic accounting.
- For vault-backed settlements, the settlement worker uses the fresh
  `latestVaultAssets` snapshot as the Base reserve-return amount and fails
  closed if that snapshot is missing. PRMX acknowledgement is still clamped to
  outstanding local liquidity, and Base `PolicyVault -> Reserve` is not a user
  payout.
- Rebalancer monitor output uses three classes:
  `uncalibrated_or_unknown`, `already_funded`, and
  `real_drawdown_candidate`. Only `real_drawdown_candidate` may produce an
  automatic rebalance-in batch.
- `scripts/prmx-monitor/new-policy-monitor.mjs` is the Codex automation surface
  for new or recently changed policies. It keeps cursor/snapshot state under
  `~/.codex/automations/prmx4-new-policy-monitor/memory.md`, checks Base
  `bridgedIn - returnedOut` overfunding, PRMX pool balance vs
  `maxPayout + reportedYield`, and vault-report freshness. If periodic report
  dispatch is intentionally disabled, missing/stale report evidence is a note,
  not a delivery anomaly.
- Rebalancer production proof should come from the hardened
  `api-lifecycle-hyperlane` gate or targeted executor logs showing ICA dispatch.
  Direct Base signer fallback is local/dev only after SBP-404 cutover.
- SBP-437 Route 2 surplus repair is a manual repair script, not a recurring
  worker. Dry-run it first, execute only during an approved repair window, and
  follow with `/capital/invariants?routeId=2` plus the hardened lifecycle gate.
- In worker mode, multiple workers can still share a signer. Existing retry
  logic handles nonce contention; avoid enabling overlapping ad-hoc operator
  runs during a mutating smoke unless the gate explicitly expects it.
